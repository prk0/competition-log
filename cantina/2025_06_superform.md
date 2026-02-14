# Superform Core

## Findings Summary 

| ID | Title | Duplicates |
| - | - | - |
| [M-01](#spectraexchangehookuseprevhookamountposition-should-be-read-from-index-24) | SpectraExchangeHook - USE_PREV_HOOK_AMOUNT_POSITION should be read from index 24 | 8 |

## SpectraExchangeHook - USE_PREV_HOOK_AMOUNT_POSITION should be read from index 24

## Summary

Wrong byte index is read from which results in the previous hook amount overwriting to be skipped in `SpectraExchangeHook`.

## Vulnerability Detail

```solidity
/// @title SpectraExchangeHook
/// @author Superform Labs
/// @dev data has the following structure
/// @notice         bytes4 placeholder = bytes4(BytesLib.slice(data, 0, 4), 0);
/// @notice         address yieldSource = BytesLib.toAddress(data, 4);
/// @notice         bool usePrevHookAmount = _decodeBool(data, 24);
/// @notice         uint256 value = BytesLib.toUint256(data, 57);  // @audit-info assume 25 - 57, 32 bytes
/// @notice         bytes txData_ = BytesLib.slice(data, 57, data.length - 57);
contract SpectraExchangeHook is BaseHook, ISuperHookContextAware, ISuperHookInspector {
    using HookDataDecoder for bytes;

    uint256 private constant USE_PREV_HOOK_AMOUNT_POSITION = 0;    // @audit this should be 24
    uint256 private constant AMOUNT_POSITION = 57;

    // SNIP
```

`USE_PREV_HOOK_AMOUNT_POSITION` is set to `0` in `SpectraExchangeHook`.

This value indicates the byte index, which will be read from for `usePrevHookAmount` in the encoded calldata string.

`placeholder` is usually in the first four bytes, which will be `0x0`, since `placeholder` is not used in this contract.

`0x0` interpreted as a bool is false - therefore, `usePrevHookAmount` will always be false. 

```solidity
function build(address prevHook, address account, bytes calldata data)
    external
    view
    override
    returns (Execution[] memory executions)
{
    address pt = data.extractYieldSource();
>   bool usePrevHookAmount = _decodeBool(data, USE_PREV_HOOK_AMOUNT_POSITION);   
    uint256 value = abi.decode(data[25:AMOUNT_POSITION], (uint256));  
    bytes memory txData_ = data[AMOUNT_POSITION:];  

    bytes memory updatedTxData = _validateTxData(data[AMOUNT_POSITION:], account, usePrevHookAmount, prevHook, pt);

    executions = new Execution[](1);
    executions[0] =
        Execution({target: address(router), value: value, callData: usePrevHookAmount ? updatedTxData : txData_});
}
```

This becomes an issue in `build()` when `usePrevHookAmount` is required for some flows.

Since `usePrevHookAmount` is always be `false`, the previous hook amount will never be set when using the Spectra Exchange hook, which can break some flows.

## PoC

Copy and paste the test case below into `SpectraExchangeSwapHook.t.sol`

Run with the following command: `forge test --match-contract SpectraExchangeHookTest --match-test test_usePrevHookAmountWrongSlice -vv`

```solidity
/**
  bytes4 placeholder = bytes4(BytesLib.slice(data, 0, 4), 0);
  address yieldSource = BytesLib.toAddress(data, 4);
  bool usePrevHookAmount = _decodeBool(data, 24);
  uint256 value = BytesLib.toUint256(data, 57); 
  bytes txData_ = BytesLib.slice(data, 57, data.length - 57);
*/
function test_usePrevHookAmountWrongSlice() public {
    bytes4 placeholder = 0x00;
    address yieldSource = address(984530);
    bool usePrevHookAmount = true;   // @audit-info usePrevHookAmount passed in as true
    uint256 value = 90283452;
    bytes memory txData = "0x123423423324234";

    bytes memory payload = abi.encodePacked(
        placeholder,
        yieldSource,
        usePrevHookAmount,
        value,
        txData
    );

    bool result = hook.decodeUsePrevHookAmount(payload);
    assert(result != usePrevHookAmount);    // @audit-info not reading from correct slice
    assert(result == false);
} 
```

```solidity
Ran 1 test for test/integration/spectra/SpectraExchangeSwapHook.t.sol:SpectraExchangeHookTest
[PASS] test_usePrevHookAmountWrongSlice() (gas: 7012)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.59s (629.00µs CPU time)

Ran 1 test suite in 5.59s (5.59s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact

Unintended DoS when `usePrevHookAmount` is used

## Recommendation

```solidity
contract SpectraExchangeHook is BaseHook, ISuperHookContextAware, ISuperHookInspector {
    using HookDataDecoder for bytes;

-   uint256 private constant USE_PREV_HOOK_AMOUNT_POSITION = 0;
+   uint256 private constant USE_PREV_HOOK_AMOUNT_POSITION = 24;
    uint256 private constant AMOUNT_POSITION = 57;

    // SNIP
```
