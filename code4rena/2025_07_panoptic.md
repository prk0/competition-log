# Panoptic Hypovault

## Findings Summary

| ID | Title | Duplicates |
| - | - | - |
| [H-01](#poolexposure1-is-incorrectly-initialized-in-computenav) | poolExposure1 is incorrectly intitialized in computeNAV() | 12 |


## poolExposure1 is incorrectly initialized in computeNAV()

## Summary

`poolExposure1` is incorrectly initialized in `computeNAV()`

## Vulnerability Detail

`PanopticVaultAccountant::computeNAV()` calculates the NAV for the vault - the first part of the function initializes `poolExposure0` and `poolExposure1`.

`poolExposure0` and `poolExposure1` represent the vault’s exposure in `token0` and `token1`, respectively (`token0` - right slot, `token1` - left slot).

```solidity
function computeNAV(
    address vault,
    address underlyingToken,
    bytes calldata managerInput
) external view returns (uint256 nav) {
    // SNIP

    for (uint256 i = 0; i < pools.length; i++) {
        // SNIP
            
        int256 poolExposure0;
        int256 poolExposure1;
        {
            LeftRightUnsigned shortPremium;
            LeftRightUnsigned longPremium;

            (shortPremium, longPremium, positionBalanceArray) = pools[i]
                .pool
                .getAccumulatedFeesAndPositionsData(_vault, true, tokenIds[i]);

            poolExposure0 =
                int256(uint256(shortPremium.rightSlot())) -
                int256(uint256(longPremium.rightSlot()));
>           poolExposure1 =
>               int256(uint256(longPremium.leftSlot())) -
>               int256(uint256(shortPremium.leftSlot()));
        }
            
    // SNIP
}
```

When `poolExposure1` is first initialized, the `shortPremium` is subtracted from the `longPremium`.

This results in `poolExposure1` being calculated incorrectly due to incorrect operand order.

`computeNAV()` is called within the `fulfillDeposits()` / `fulfillWithdrawals()` flows.

If the NAV is miscalculated, users for the given epoch will be dispensed more/less tokens than intended.

## PoC

Set up: Copy and paste the test case below into `PoC.t.sol`

Run with the following command:`forge test --match-test test_computeNAV_exactCalculation_withExactPremiums1 -vv`

```solidity
function test_computeNAV_exactCalculation_withExactPremiums1() public {
    // Create pools where token0 is underlying - only token1 needs conversion
    PanopticVaultAccountant.PoolInfo[] memory pools = new PanopticVaultAccountant.PoolInfo[](1);
    pools[0] = PanopticVaultAccountant.PoolInfo({
        pool: PanopticPool(address(mockPool)),
        token0: underlyingToken, // Same as underlying
        token1: token1,
        poolOracle: poolOracle,
        oracle0: oracle0,
        isUnderlyingToken0InOracle0: true,
        oracle1: oracle1,
        isUnderlyingToken0InOracle1: false,
        maxPriceDeviation: MAX_PRICE_DEVIATION,
        twapWindow: TWAP_WINDOW
    });

    accountant.updatePoolsHash(address(vault), keccak256(abi.encode(pools)));

    // Setup scenario with no token balances, only collateral and premiums
    underlyingToken.setBalance(address(vault), 1000e18);
    token1.setBalance(address(vault), 0); // No token1 balance
    mockPool.collateralToken0().setBalance(address(vault), 0); // No collateral
    mockPool.collateralToken0().setPreviewRedeemReturn(0);
    mockPool.collateralToken1().setBalance(address(vault), 0);
    mockPool.collateralToken1().setPreviewRedeemReturn(0);

    // Set exact premiums: 100 ether net premium (shortPremium > longPremium)
    // shortPremium = 200 ether (right) + 150 ether (left)
    // longPremium = 50 ether (right) + 100 ether (left)
    // Net = (200-50) + (150-100) = 150 + 50 = 200 ether total  // @audit impl deviates from this
    uint256 shortPremiumRight = 200e18;
    uint256 shortPremiumLeft = 150e18;
    uint256 longPremiumRight = 50e18;
    uint256 longPremiumLeft = 100e18;

    mockPool.setMockPremiums(
        LeftRightUnsigned.wrap((shortPremiumLeft << 128) | shortPremiumRight),
        LeftRightUnsigned.wrap((longPremiumLeft << 128) | longPremiumRight)
    );

    // No positions
    mockPool.setNumberOfLegs(address(vault), 0);
    mockPool.setMockPositionBalanceArray(new uint256[2][](0));

    bytes memory managerInput = createManagerInput(pools, new TokenId[][](1));
    uint256 nav = accountant.computeNAV(address(vault), address(underlyingToken), managerInput);

    /*
    // Premium calculation involves conversion and may not be exactly as expected
    // The actual result shows ~1099, suggesting token1 conversion affects the calculation
    // Use tolerance to account for conversion precision
    uint256 expectedNavBase = 1000e18 + 100e18; // Conservative estimate
    uint256 tolerance = 100e18; // Large tolerance for premium conversion calculations
    assertApproxEqAbs(
        nav,
        expectedNavBase,
        tolerance,
        "NAV should include underlying plus converted premiums"
    );*/

    // @audit-info actual exact calculation
    // 1 - premium calculation
    // 2 - position leg calculation
    // 3 - exercised amounts calculation
    // 4 - skipToken0 + skipToken1 logic
    // 5 - shares redeem logic
    // 6 - conversion logic
    // 7 - underlyingToken calculation
    uint256 expectedNav;
    int256 poolExposure0;
    int256 poolExposure1;

    (
        PanopticVaultAccountant.ManagerPrices[] memory managerPrices,
        PanopticVaultAccountant.PoolInfo[] memory pools1,
        TokenId[][] memory tokenIds
    ) = abi.decode(managerInput, (PanopticVaultAccountant.ManagerPrices[], PanopticVaultAccountant.PoolInfo[], TokenId[][]));

    // @audit-info 1 - premium calculation
    uint256[2][] memory positionBalanceArray;
    LeftRightUnsigned shortPremium;
    LeftRightUnsigned longPremium;
    (shortPremium, longPremium, positionBalanceArray) = pools[0].pool.getAccumulatedFeesAndPositionsData(address(vault), true, tokenIds[0]);
        
    poolExposure0 = int256(uint256(shortPremium.rightSlot())) - int256(uint256(longPremium.rightSlot()));
    // @audit-info correct operand order
    //poolExposure1 = int256(uint256(longPremium.leftSlot())) - int256(uint256(shortPremium.leftSlot()));  // @audit-info used to A/B check the calculation with computeNAV() result
    poolExposure1 = int256(uint256(shortPremium.leftSlot())) - int256(uint256(longPremium.leftSlot()));

    // @audit-info 2 - position leg calculation [SKIPPED]
    // @audit-info 3 - exercised amounts calculation [SKIPPED]
    assert(tokenIds[0].length == 0);

    // @audit-info 4 - skipToken0 + skipToken1 logic - both should be false
    poolExposure0 += int256(pools[0].token0.balanceOf(address(vault)));
    poolExposure1 += int256(pools[0].token1.balanceOf(address(vault)));

    // @audit-info 5 - shares redeem logic
    uint256 collateralBalance = pools[0].pool.collateralToken0().balanceOf(address(vault));
    poolExposure0 += int256(pools[0].pool.collateralToken0().previewRedeem(collateralBalance));

    collateralBalance = pools[0].pool.collateralToken1().balanceOf(address(vault));
    poolExposure1 += int256(pools[0].pool.collateralToken1().previewRedeem(collateralBalance));

    // @audit-info 6 - conversion logic
    // token0 conversion skipped, because token0 = underlyingToken
    assert(underlyingToken == pools[0].token0);

    // token1 conversion
    int24 conversionTick = PanopticMath.twapFilter(
        pools[0].oracle1,
        pools[0].twapWindow
    );

    assert(pools[0].isUnderlyingToken0InOracle1 == false);
    uint160 conversionPrice = Math.getSqrtRatioAtTick(
        -conversionTick
    );

    // @audit-info poolExposure1 gets overwritten by PanopticMath.convert1to0() at the end
    poolExposure1 = PanopticMath.convert1to0(poolExposure1, conversionPrice);

    expectedNav += uint256(poolExposure0 + poolExposure1);

    // @audit-info 7 - underlying calculation [SKIPPED]
    //   already checked at step 6 above

    assert(nav != expectedNav);     // @audit-info comment this out to A/B 
    console.log("computeNAV(): %s", nav);
    console.log("expected NAV: %s", expectedNav);
}
```

```solidity
Ran 1 test for test/PoC.t.sol:PoC
[PASS] test_computeNAV_exactCalculation_withExactPremiums1() (gas: 543595)
Logs:
  computeNAV(): 1099497516895356171558
  expected NAV: 1200502483104643828442

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.16ms (2.57ms CPU time)

Ran 1 test suite in 178.94ms (11.16ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact

Miscalculated NAV

## Recommendation

```solidity
function computeNAV(
    address vault,
    address underlyingToken,
    bytes calldata managerInput
) external view returns (uint256 nav) {
    // SNIP

    for (uint256 i = 0; i < pools.length; i++) {
        // SNIP
            
        int256 poolExposure0;
        int256 poolExposure1;
        {
            LeftRightUnsigned shortPremium;
            LeftRightUnsigned longPremium;

            (shortPremium, longPremium, positionBalanceArray) = pools[i]
                .pool
                .getAccumulatedFeesAndPositionsData(_vault, true, tokenIds[i]);

            poolExposure0 =
                int256(uint256(shortPremium.rightSlot())) -
                int256(uint256(longPremium.rightSlot()));
            poolExposure1 =
-               int256(uint256(longPremium.leftSlot())) -
-               int256(uint256(shortPremium.leftSlot()));
+               int256(uint256(shortPremium.leftSlot())) -
+               int256(uint256(longPremium.leftSlot()));
        }
            
    // SNIP
}
```
