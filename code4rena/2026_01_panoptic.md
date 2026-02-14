# Panoptic: Next Core

## Findings Summary

| Id | Title | Duplicates |
| - | - | - |
| [H-01](#commission-fees-can-always-be-bypassed) | Commission fees can always be bypassed | 4 |
| [M-01](#riskengine-computedelayedswap-does-not-return-the-correct-value) | RiskEngine::_computeDelayedSwap() does not return the correct value | 6 |
| [QA](#qa-report) | QA Report | N/A |

## Commission fees can always be bypassed

## Vulnerability Detail

```solidity
function settleBurn(
    address optionOwner,
    int128 longAmount,
    int128 shortAmount,
    int128 ammDeltaAmount,
    int128 realizedPremium,
    RiskParameters riskParameters
) external onlyPanopticPool returns (int128) {
    (, int128 tokenPaid, uint256 _totalAssets, uint256 _totalSupply) = _updateBalancesAndSettle(
        optionOwner,
        false, // isCreation = false
        longAmount,
        shortAmount,
        ammDeltaAmount,
        realizedPremium
    );

    if (realizedPremium != 0) {
        uint128 commissionFee;
        // compute the minimum of the notionalFee and the premiumFee
        {
            uint128 commissionP;
            unchecked {
                commissionP = realizedPremium > 0
                    ? uint128(realizedPremium)
                    : uint128(-realizedPremium);
            }
            uint128 commissionFeeP = Math
                .mulDivRoundingUp(commissionP, riskParameters.premiumFee(), DECIMALS)
                .toUint128();
            uint128 commissionN = uint256(int256(shortAmount) + int256(longAmount)).toUint128();
            uint128 commissionFeeN;
            unchecked {
                commissionFeeN = Math 
                    .mulDivRoundingUp(commissionN, 10 * riskParameters.notionalFee(), DECIMALS)
                    .toUint128(); 
            }
>           commissionFee = Math.min(commissionFeeP, commissionFeeN).toUint128();
        }

        // SNIP - distribution
    }

    return tokenPaid;
}
```

`CollateralTracker::settleBurn()` computes the `commissionFee` as the *minimum* between premium fee and notional fee.

`CollateralTracker::settleBurn()` is called in the following flows:

- Flow 1 - Force Exercise:
    
    `PanopticPool::dispatchFrom()`
    
    → `PanopticPool::_forceExercise()`
    
    → `PanopticPool::_burnOptions()`
    
    → `CollateralTracker::settleBurn()`
    
- Flow 2 - Burn:
    
    `PanopticPool::dispatch()`
    
    → `PanopticPool::_burnOptions()`
    
    → `CollateralTracker::settleBurn()`
    
- Flow 3 - Liquidate:
    
    `PanopticPool::dispatchFrom()`
    
    → `PanopticPool::_liquidate()`
    
    → `PanopticPool::_burnAllOptionsFrom()`
    
    → `PanopticPool::_burnOptions()` 
    
    → `CollateralTracker::settleBurn()`
    
- Flow 4 - Settle Premium:
    
    `PanopticPool::dispatchFrom()`
    
    → `PanopticPool::_settlePremium()`
    
    → `PanopticPool::_settleOptions()`
    
    → `CollateralTracker::settleBurn()`
    
- Flow 5 - Settle Premium:
    
    `PanopticPool::dispatch()`
    
    → `PanopticPool::_settleOptions()`
    
    → `CollateralTracker::settleBurn()`
    

```solidity
function _settleOptions(
    address owner,
    TokenId tokenId,
    uint128 positionSize,
    RiskParameters riskParameters,
    int24 currentTick
) internal {
    // call _updateSettlementPostBurn to settle the long premia or the short premia (only for self calling)
    LeftRightUnsigned[4] memory emptyCollectedByLegs;
    LeftRightSigned realizedPremia;
    unchecked {
        // cannot be miscast because currentTick is a int24
        (realizedPremia, ) = _updateSettlementPostBurn(
            owner, // owner
            tokenId, // tokenId
            emptyCollectedByLegs, // collectedByLeg
            positionSize, // positionSize
            riskParameters, // riskParameters
            LeftRightSigned.wrap(1).addToLeftSlot(1 + (int128(currentTick) << 2)) 
        );
    }
    // deduct the paid premium tokens from the owner's balance
>   collateralToken0().settleBurn(owner, 0, 0, 0, realizedPremia.rightSlot(), riskParameters); // @audit
>   collateralToken1().settleBurn(owner, 0, 0, 0, realizedPremia.leftSlot(), riskParameters); // @audit
}
```

In Flow 4 and Flow 5, `settleBurn()` is called from `_settleOptions()` - The supplied `longAmount`, `shortAmount`, and `ammDeltaAmount` parameters are all `0`.

Therefore, we can expect the computed notional fee to also be `0`.

As mentioned above, the commission fee is taken from the *minimum* between notional fee and premium fee.

As a result, the commission fee will always be computed to `0`, when `settleBurn()` is called from `_settleOptions()` (Flow 4 and Flow 5).

```solidity
function settleBurn(
    address optionOwner,
    int128 longAmount,
    int128 shortAmount,
    int128 ammDeltaAmount,
    int128 realizedPremium,
    RiskParameters riskParameters
) external onlyPanopticPool returns (int128) {
    (, int128 tokenPaid, uint256 _totalAssets, uint256 _totalSupply) = _updateBalancesAndSettle(
        optionOwner,
        false, // isCreation = false
        longAmount,
        shortAmount,
        ammDeltaAmount,
        realizedPremium
    );

>   if (realizedPremium != 0) {
        // SNIP
    }

    return tokenPaid;
}
```

`CollateralTracker::settleBurn()` also skips the commission fee computation altogether when `realizedPremium = 0`.

Therefore, a user can always avoid commission fees by settling premium first, then burning.

## Impact

Commission fees are always computed to 0 in flows that involve `_settleOptions()`.

In addition, commission fees are skipped if `realizedPremium` is `0`.

This results in commission fees being skipped in many flows.

Users can avoid commission fees completely by settling premium, then burning, on exit. 

## Recommendation

Consider  removing the `realizedPremium != 0`  check and consider deriving commission fee directly from premium fee, when `shortAmount` and `longAmount` are both `= 0`.

```solidity
function settleBurn(
    address optionOwner,
    int128 longAmount,
    int128 shortAmount,
    int128 ammDeltaAmount,
    int128 realizedPremium,
    RiskParameters riskParameters
) external onlyPanopticPool returns (int128) {
    (, int128 tokenPaid, uint256 _totalAssets, uint256 _totalSupply) = _updateBalancesAndSettle(
        optionOwner,
        false, // isCreation = false
        longAmount,
        shortAmount,
        ammDeltaAmount,
        realizedPremium
    );

-   if (realizedPremium != 0) {
        uint128 commissionFee;
        // compute the minimum of the notionalFee and the premiumFee
        { 
            uint128 commissionP;
            unchecked {
                commissionP = realizedPremium > 0
                    ? uint128(realizedPremium)
                    : uint128(-realizedPremium);
            }
            uint128 commissionFeeP = Math 
                .mulDivRoundingUp(commissionP, riskParameters.premiumFee(), DECIMALS)
                .toUint128();
            uint128 commissionN = uint256(int256(shortAmount) + int256(longAmount)).toUint128();
            uint128 commissionFeeN;
            unchecked {
                commissionFeeN = Math 
                    .mulDivRoundingUp(commissionN, 10 * riskParameters.notionalFee(), DECIMALS)
                    .toUint128(); 
            }
            
+           if (shortAmount == 0 && longAmount == 0) {
+               commissionFee = commissionFeeP;
+           } else {
                commissionFee = Math.min(commissionFeeP, commissionFeeN).toUint128();
+           }
        }

        // SNIP
-   }

    return tokenPaid;
}
```

## PoC

```solidity
// test/foundry/coreV3/Misc.t.sol
// ❯ forge test --match-test test_success_settleShortPremium_self_PoC -vvvv

// @audit-info adapted test_success_settleShortPremium_self()
function test_success_settleShortPremium_self_PoC() public {
    swapperc = new SwapperC();
    vm.startPrank(Swapper);
    token0.mint(Swapper, type(uint128).max);
    token1.mint(Swapper, type(uint128).max);
    token0.approve(address(swapperc), type(uint128).max);
    token1.approve(address(swapperc), type(uint128).max);

    {
        poolId = uint40(uint160(address(uniPool)) >> 112) + uint64(uint256(vegoid) << 40);
        poolId += uint64(uint24(uniPool.tickSpacing())) << 48;
    }
    swapperc.mint(uniPool, -887200, 887200, 10 ** 18);

    // sell primary chunk
    $posIdLists[0].push(TokenId.wrap(0).addPoolId(poolId).addLeg(0, 1, 1, 0, 0, 0, 15, 1));

    // mint some amount of liquidity with Alice owning 1/2 and Bob and Charlie owning 1/4 respectively
    // then, remove 9.737% of that liquidity at the same ratio
    // Once this state is in place, accumulate some amount of fees on the existing liquidity in the pool
    // The fees should be immediately available for withdrawal because they have been paid to liquidity already in the pool
    // 8.896% * 1.022x vegoid = +~10% of the fee amount accumulated will be owed by sellers
    vm.startPrank(Alice);

    mintOptions(
        pp,
        $posIdLists[0],
        1_000,
        0,
        Constants.MAX_POOL_TICK,
        Constants.MIN_POOL_TICK,
        true
    );

    vm.startPrank(Bob);

    mintOptions(
        pp,
        $posIdLists[0],
        499_999_500,
        0,
        Constants.MAX_POOL_TICK,
        Constants.MIN_POOL_TICK,
        true
    );

    vm.startPrank(Charlie);

    mintOptions(
        pp,
        $posIdLists[0],
        499_999_500,
        0,
        Constants.MAX_POOL_TICK,
        Constants.MIN_POOL_TICK,
        true
    );

    {
        poolId = uint40(uint160(address(uniPool)) >> 112) + uint64(uint256(vegoid) << 40);
        poolId += uint64(uint24(uniPool.tickSpacing())) << 48;
    }

    // position type A: 1-leg long primary
    $posIdLists[2].push(TokenId.wrap(0).addPoolId(poolId).addLeg(0, 1, 1, 1, 0, 0, 15, 1));

    // Buyer 1 buys the chunk
    vm.startPrank(Buyers[0]);
    mintOptions(
        pp,
        $posIdLists[2],
        9_884_444,
        type(uint24).max,
        Constants.MAX_POOL_TICK,
        Constants.MIN_POOL_TICK,
        true
    );

    vm.startPrank(Swapper);

    swapperc.swapTo(uniPool, Math.getSqrtRatioAtTick(100) + 1);

    // There are some precision issues with this (1B is not exactly 1B) but close enough to see the effects
    accruePoolFeesInRange(address(uniPool), uniPool.liquidity() - 1, 1_000_000, 1_000_000_000);

    // accumulate lower order of fees on dummy chunk
    swapperc.swapTo(uniPool, Math.getSqrtRatioAtTick(-10));

    accruePoolFeesInRange(address(uniPool), uniPool.liquidity() - 1, 10_000, 100_000);

    swapperc.swapTo(uniPool, 2 ** 96);

    vm.startPrank(Alice);
    {
        (LeftRightUnsigned shortPremium, , ) = pp.getAccumulatedFeesAndPositionsData(
            Alice,
            true,
            $posIdLists[0]
        );

        assertGe(shortPremium.rightSlot(), 0);
        assertGe(shortPremium.leftSlot(), 0);

        (uint256 aliceBalanceBefore0, uint256 aliceBalanceBefore1) = (
            ct0.balanceOf(Alice),
            ct1.balanceOf(Alice)
        );

        // Alice settles her own position, received nothing because the chunks haven't been poked.
        settlePremiumSelf(pp, $posIdLists[0], 1_000, true);
        (uint256 aliceBalanceAfter0, uint256 aliceBalanceAfter1) = (
            ct0.balanceOf(Alice),
            ct1.balanceOf(Alice)
        );

        (shortPremium, , ) = pp.getAccumulatedFeesAndPositionsData(Alice, true, $posIdLists[0]);

        // has 0 owed premium because it was settled at 0 in settlePremium
        assertEq(shortPremium.rightSlot(), 0);
        assertEq(shortPremium.leftSlot(), 0);

        // burn options and forfeit her premium, which was settled as 0 in settlePremium
        burnOptions(
            pp,
            $posIdLists[0],
            new TokenId[](0),
            Constants.MAX_POOL_TICK,
            Constants.MIN_POOL_TICK,
            false
        );

        (uint256 aliceBalancePost0, uint256 aliceBalancePost1) = (
            ct0.balanceOf(Alice),
            ct1.balanceOf(Alice)
        );

        // Alice re-mints an option, pokes the chunk and make the protocol collect some premium
        mintOptions(
            pp,
            $posIdLists[0],
            1,
            0,
            Constants.MAX_POOL_TICK,
            Constants.MIN_POOL_TICK,
            true
        );
    }

    vm.startPrank(Charlie);

    uint256 charlieDeltaPremia0;
    uint256 charlieDeltaPremia1;

    {
        (LeftRightUnsigned shortPremium, , ) = pp.getAccumulatedFeesAndPositionsData(
            Charlie,
            true,
            $posIdLists[0]
        );
        uint256 owedPremia0 = shortPremium.rightSlot();
        uint256 owedPremia1 = shortPremium.leftSlot();
        console2.log("owedPremia-total0", owedPremia0);
        console2.log("owedPremia-total1", owedPremia1);

        assertGe(owedPremia0, 0);
        assertGe(owedPremia1, 0);

        (uint256 charlieBalanceBefore0, uint256 charlieBalanceBefore1) = (
            ct0.balanceOf(Charlie),
            ct1.balanceOf(Charlie)
        );

        // Charlie settles Buyers[0] premium first
        settlePremium(pp, $posIdLists[0], $posIdLists[2], Buyers[0], 0, true);

        // Charlie settles his own premium, receives only realize premia from settled longs
        settlePremiumSelf(pp, $posIdLists[0], 499_999_500, true);

        (uint256 charlieBalanceAfter0, uint256 charlieBalanceAfter1) = (
            ct0.balanceOf(Charlie),
            ct1.balanceOf(Charlie)
        );

        assertGt(charlieBalanceAfter0, charlieBalanceBefore0, "charlie received premia0");
        assertGt(charlieBalanceAfter1, charlieBalanceBefore1, "charlie received premia1");

        charlieDeltaPremia0 = ct0.convertToAssets(charlieBalanceAfter0 - charlieBalanceBefore0);
        charlieDeltaPremia1 = ct1.convertToAssets(charlieBalanceAfter1 - charlieBalanceBefore1);

        assertApproxEqAbs(
            charlieDeltaPremia0,
            owedPremia0,
            1,
            "charlie received exactly what they are owed due to settled token0"
        );
        assertApproxEqAbs(
            charlieDeltaPremia1,
            owedPremia1,
            1,
            "charlie received exactly what they are owed due to settled token0"
        );

        // @audit-info burn position
        burnOptions(
            pp,
            $posIdLists[0],
            new TokenId[](0),
            Constants.MAX_POOL_TICK,
            Constants.MIN_POOL_TICK,
            true
        );
    }

    vm.stopPrank();
}
```

```solidity
    ├─ [191106] 0x2304e805a66ee4627561F470475EAf9dE34D85F7::dispatch([1267651733651528342013150880167 [1.267e30]], [1267651733651528342013150880167 [1.267e30]], [499999500 [4.999e8]], [[-887272 [-8.872e5], 887272 [8.872e5], -1]], true, 0)
    │   ├─ [190809] PanopticPool::c25813aa(00000000000000000000000000000000000000000000000000000000000000c000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000140000000000000000000000000000000000000000000000000000000000000018000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000100000f003000a0464bc31e1a7000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000100000f003000a0464bc31e1a70000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000001dcd630c0000000000000000000000000000000000000000000000000000000000000001fffffffffffffffffffffffffffffffffffffffffffffffffffffffffff2761800000000000000000000000000000000000000000000000000000000000d89e8ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffad89b4fd8f609caa0caa087c01db16f51cc320849afab8ead9ec80cd40363c037c2dfbe12ddba14d3cd4e8a3644ddc8b16954a9f50fd0dc0185161ac0000000000000000000000000000000000000000000a0464bc31e1a70000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f740078) [delegatecall]
    │   │   ├─ [2529] SemiFungiblePositionManager::getCurrentTick(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74) [staticcall]
    │   │   │   ├─ [696] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::slot0() [staticcall]
    │   │   │   │   └─ ← [Return] 79228162514264337593543950336 [7.922e28], 0, 1, 100, 100, 0, true
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [2527] RiskEngine::getRiskParameters(0, 74738209495215746795597743391008221985693311380285640030731442434076690821042 [7.473e76], 0) [staticcall]
    │   │   │   └─ ← [Return] 88257235459544876001772053827994779808 [8.825e37]
    │   │   ├─ [2529] SemiFungiblePositionManager::getCurrentTick(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74) [staticcall]
    │   │   │   ├─ [696] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::slot0() [staticcall]
    │   │   │   │   └─ ← [Return] 79228162514264337593543950336 [7.922e28], 0, 1, 100, 100, 0, true
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [12003] SemiFungiblePositionManager::getAccountPremium(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74, 0x2304e805a66ee4627561F470475EAf9dE34D85F7, 0, 10, 20, 0, 0, 4) [staticcall]
    │   │   │   ├─ [5349] FeesCalc::calculateAMMSwapFees() [delegatecall]
    │   │   │   │   ├─ [1199] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::ticks(10) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1978843488130 [1.978e12], 1978843488130 [1.978e12], 763877574750271127826032773528851 [7.638e32], 768430205267730256608715008567929 [7.684e32], 0, 0, 0, true
    │   │   │   │   ├─ [1199] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::ticks(20) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1978843488130 [1.978e12], -1978843488130 [-1.978e12], 678832485483409194085586698696522 [6.788e32], 683257459030888515890930737465698 [6.832e32], 0, 0, 0, true
    │   │   │   │   └─ ← [Return] 168543217465408504707263296559384522207497186 [1.685e44]
    │   │   │   └─ ← [Return] 4610416639687 [4.61e12], 4617333714248 [4.617e12]
    │   │   ├─ [1453] SemiFungiblePositionManager::getAccountLiquidity(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74, 0x2304e805a66ee4627561F470475EAf9dE34D85F7, 0, 10, 20) [staticcall]
    │   │   │   └─ ← [Return] 6722296912845509826321440690242695013765329674114 [6.722e48]
    │   │   ├─ emit PremiumSettled(user: 0x0000000000000000000000000000001234567891, tokenId: 1267651733651528342013150880167 [1.267e30], legIndex: 0, settledAmounts: 85114828437934337866093890556908182731739036 [8.511e43])
>   │   │   ├─ [21704] 0xaD89B4fd8F609caa0CAa087c01dB16f51Cc32084::settleBurn(0x0000000000000000000000000000001234567891, 0, 0, 0, 249756 [2.497e5], 88257235459544876001772053827994779808 [8.825e37])
    │   │   │   ├─ [21465] CollateralTracker::1d3e716e(0000000000000000000000000000000000000000000000000000001234567891000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000003cf9c000000000000000000000000000000004265b9aa82bf202012711964000000a02304e805a66ee4627561f470475eaf9de34d85f70178bc008cd5dc9dd44bc3baad07e7b0568bcff19478bc008cd5dc9dd44bc3baad07e7b0568bcff194f674deda0f700f74e6675ad569cb089d0d0e9f613cd4e8a3644ddc8b16954a9f50fd0dc0185161ac00000000000000000000000000000000000000000001f4007c) [delegatecall]
    │   │   │   │   ├─ [4009] RiskEngine::updateInterestRate(100000000000000 [1e14], 6585427604981613837176141717962047016300123 [6.585e42]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 317218735 [3.172e8], 1268301190 [1.268e9]
    │   │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x0000000000000000000000000000001234567891, amount: 249755999999 [2.497e11])
>   │   │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000001234567891, to: 0x0000000000000000000000000000000000000000, amount: 0)
>   │   │   │   │   ├─ emit CommissionPaid(owner: 0x0000000000000000000000000000001234567891, builder: 0x0000000000000000000000000000000000000000, commissionPaidProtocol: 0, commissionPaidBuilder: 0)
    │   │   │   │   └─ ← [Return] 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffc3064
    │   │   │   └─ ← [Return] -249756 [-2.497e5]
>   │   │   ├─ [24324] 0x9aFAb8Ead9EC80cD40363C037C2DFbe12dDBa14D::settleBurn(0x0000000000000000000000000000001234567891, 0, 0, 0, 250130 [2.501e5], 88257235459544876001772053827994779808 [8.825e37])
    │   │   │   ├─ [24085] CollateralTracker::1d3e716e(0000000000000000000000000000000000000000000000000000001234567891000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000003d112000000000000000000000000000000004265b9aa82bf202012711964000000a02304e805a66ee4627561f470475eaf9de34d85f700f674deda0f700f74e6675ad569cb089d0d0e9f6178bc008cd5dc9dd44bc3baad07e7b0568bcff194f674deda0f700f74e6675ad569cb089d0d0e9f613cd4e8a3644ddc8b16954a9f50fd0dc0185161ac00000000000000000000000000000000000000000001f4007c) [delegatecall]
    │   │   │   │   ├─ [4009] RiskEngine::updateInterestRate(0, 6585427568635535827432348318248572711759451 [6.585e42]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 317076049 [3.17e8], 1268301182 [1.268e9]
    │   │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x0000000000000000000000000000001234567891, amount: 250130000000 [2.501e11])
>   │   │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000001234567891, to: 0x0000000000000000000000000000000000000000, amount: 0)
>   │   │   │   │   ├─ emit CommissionPaid(owner: 0x0000000000000000000000000000001234567891, builder: 0x0000000000000000000000000000000000000000, commissionPaidProtocol: 0, commissionPaidBuilder: 0)
    │   │   │   │   └─ ← [Return] 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffc2eee
    │   │   │   └─ ← [Return] -250130 [-2.501e5]
    │   │   ├─ [386] PanopticMath::hasNoDuplicateTokenIds([1267651733651528342013150880167 [1.267e30]]) [delegatecall]
    │   │   │   └─ ← [Return] true
    │   │   ├─ [2529] SemiFungiblePositionManager::getCurrentTick(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74) [staticcall]
    │   │   │   ├─ [696] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::slot0() [staticcall]
    │   │   │   │   └─ ← [Return] 79228162514264337593543950336 [7.922e28], 0, 1, 100, 100, 0, true
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [5715] RiskEngine::getSolvencyTicks(0, 74738209495215746795597743391008221985693311380285640030731442434076690821042 [7.473e76]) [staticcall]
    │   │   │   └─ ← [Return] [0], 0
    │   │   ├─ [12003] SemiFungiblePositionManager::getAccountPremium(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74, 0x2304e805a66ee4627561F470475EAf9dE34D85F7, 0, 10, 20, 0, 0, 4) [staticcall]
    │   │   │   ├─ [5349] FeesCalc::calculateAMMSwapFees() [delegatecall]
    │   │   │   │   ├─ [1199] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::ticks(10) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1978843488130 [1.978e12], 1978843488130 [1.978e12], 763877574750271127826032773528851 [7.638e32], 768430205267730256608715008567929 [7.684e32], 0, 0, 0, true
    │   │   │   │   ├─ [1199] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::ticks(20) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1978843488130 [1.978e12], -1978843488130 [-1.978e12], 678832485483409194085586698696522 [6.788e32], 683257459030888515890930737465698 [6.832e32], 0, 0, 0, true
    │   │   │   │   └─ ← [Return] 168543217465408504707263296559384522207497186 [1.685e44]
    │   │   │   └─ ← [Return] 4610416639687 [4.61e12], 4617333714248 [4.617e12]
    │   │   ├─ [1453] SemiFungiblePositionManager::getAccountLiquidity(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74, 0x2304e805a66ee4627561F470475EAf9dE34D85F7, 0, 10, 20) [staticcall]
    │   │   │   └─ ← [Return] 6722296912845509826321440690242695013765329674114 [6.722e48]
    │   │   ├─ [36778] RiskEngine::isAccountSolvent([340282366920938463463374607432268210956 [3.402e38]], [1267651733651528342013150880167 [1.267e30]], 0, 0x0000000000000000000000000000001234567891, 0, 0, 0xaD89B4fd8F609caa0CAa087c01dB16f51Cc32084, 0x9aFAb8Ead9EC80cD40363C037C2DFbe12dDBa14D, 13333333 [1.333e7]) [staticcall]
    │   │   │   ├─ [8065] 0xaD89B4fd8F609caa0CAa087c01dB16f51Cc32084::assetsAndInterest(0x0000000000000000000000000000001234567891) [staticcall]
    │   │   │   │   ├─ [7853] CollateralTracker::849e14f7(00000000000000000000000000000000000000000000000000000012345678912304e805a66ee4627561f470475eaf9de34d85f70178bc008cd5dc9dd44bc3baad07e7b0568bcff19478bc008cd5dc9dd44bc3baad07e7b0568bcff194f674deda0f700f74e6675ad569cb089d0d0e9f613cd4e8a3644ddc8b16954a9f50fd0dc0185161ac00000000000000000000000000000000000000000001f4007c) [delegatecall]
    │   │   │   │   │   ├─ [3951] RiskEngine::interestRate(1, 6585396285046963155095886422008189160681051 [6.585e42]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 317074543 [3.17e8]
    │   │   │   │   │   └─ ← [Return] 0x00000000000000000000000000000000000001000000000000000000000152640000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   └─ ← [Return] 20282409603651670423947251372644 [2.028e31], 0
    │   │   │   ├─ [8006] 0x9aFAb8Ead9EC80cD40363C037C2DFbe12dDBa14D::assetsAndInterest(0x0000000000000000000000000000001234567891) [staticcall]
    │   │   │   │   ├─ [7794] CollateralTracker::849e14f7(00000000000000000000000000000000000000000000000000000012345678912304e805a66ee4627561f470475eaf9de34d85f700f674deda0f700f74e6675ad569cb089d0d0e9f6178bc008cd5dc9dd44bc3baad07e7b0568bcff194f674deda0f700f74e6675ad569cb089d0d0e9f613cd4e8a3644ddc8b16954a9f50fd0dc0185161ac00000000000000000000000000000000000000000001f4007c) [delegatecall]
    │   │   │   │   │   ├─ [3951] RiskEngine::interestRate(0, 6585396243508588286817265393764218526920283 [6.585e42]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 317074541 [3.17e8]
    │   │   │   │   │   └─ ← [Return] 0x000000000000000000000000000000000000010000000000000000000003d1110000000000000000000000000000000000000000000000000000000000000000
    │   │   │   │   └─ ← [Return] 20282409603651670423947251536145 [2.028e31], 0
    │   │   │   └─ ← [Return] true
    │   │   └─ ← [Stop]
    │   └─ ← [Return]

// SNIP

    ├─ [161751] 0x2304e805a66ee4627561F470475EAf9dE34D85F7::dispatch([1267651733651528342013150880167 [1.267e30]], [], [0], [[887272 [8.872e5], -887272 [-8.872e5], -1]], true, 0)
    │   ├─ [161460] PanopticPool::c25813aa(00000000000000000000000000000000000000000000000000000000000000c000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000120000000000000000000000000000000000000000000000000000000000000016000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000100000f003000a0464bc31e1a7000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000d89e8fffffffffffffffffffffffffffffffffffffffffffffffffffffffffff27618ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffad89b4fd8f609caa0caa087c01db16f51cc320849afab8ead9ec80cd40363c037c2dfbe12ddba14d3cd4e8a3644ddc8b16954a9f50fd0dc0185161ac0000000000000000000000000000000000000000000a0464bc31e1a70000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f740078) [delegatecall]
    │   │   ├─ [2529] SemiFungiblePositionManager::getCurrentTick(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74) [staticcall]
    │   │   │   ├─ [696] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::slot0() [staticcall]
    │   │   │   │   └─ ← [Return] 79228162514264337593543950336 [7.922e28], 0, 1, 100, 100, 0, true
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [2527] RiskEngine::getRiskParameters(0, 74738209495215746795597743391008221985693311380285640030731442434076690821042 [7.473e76], 0) [staticcall]
    │   │   │   └─ ← [Return] 88257235459544876001772053827994779808 [8.825e37]
    │   │   ├─ [76020] SemiFungiblePositionManager::burnTokenizedPosition(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74, 1267651733651528342013150880167 [1.267e30], 499999500 [4.999e8], 887272 [8.872e5], -887272 [-8.872e5])
    │   │   │   ├─ emit TransferSingle(operator: 0x2304e805a66ee4627561F470475EAf9dE34D85F7, from: 0x2304e805a66ee4627561F470475EAf9dE34D85F7, to: 0x0000000000000000000000000000000000000000, id: 1267651733651528342013150880167 [1.267e30], amount: 499999500 [4.999e8])
    │   │   │   ├─ emit TokenizedPositionBurnt(recipient: 0x2304e805a66ee4627561F470475EAf9dE34D85F7, tokenId: 1267651733651528342013150880167 [1.267e30], positionSize: 499999500 [4.999e8])
    │   │   │   ├─ [35136] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::burn(10, 20, 999299270623 [9.992e11])
    │   │   │   │   ├─ emit Burn(owner: SemiFungiblePositionManager: [0xFa0F05A76eBcbf400856cCcCF29FF66d8acC843f], tickLower: 10, tickUpper: 20, amount: 999299270623 [9.992e11], amount0: 499250100 [4.992e8], amount1: 0)
    │   │   │   │   └─ ← [Return] 499250100 [4.992e8], 0
    │   │   │   ├─ [1013] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::positions(0xd4c9437bd39b24a03763ba043adb620f1f1c5132e8b9f2d70fce53696b5151ae) [staticcall]
    │   │   │   │   └─ ← [Return] 979544217507 [9.795e11], 85045089266861933740446074832329 [8.504e31], 85172746236841740717784271102231 [8.517e31], 499250100 [4.992e8], 0
    │   │   │   ├─ [8947] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::collect(0x2304e805a66ee4627561F470475EAf9dE34D85F7, 10, 20, 499250100 [4.992e8], 0)
    │   │   │   │   ├─ [2924] ERC20S::transfer(0x2304e805a66ee4627561F470475EAf9dE34D85F7, 499250100 [4.992e8])
    │   │   │   │   │   ├─ emit Transfer(from: 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74, to: 0x2304e805a66ee4627561F470475EAf9dE34D85F7, amount: 499250100 [4.992e8])
    │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   ├─ emit Collect(owner: SemiFungiblePositionManager: [0xFa0F05A76eBcbf400856cCcCF29FF66d8acC843f], recipient: 0x2304e805a66ee4627561F470475EAf9dE34D85F7, tickLower: 10, tickUpper: 20, amount0: 499250100 [4.992e8], amount1: 0)
    │   │   │   │   └─ ← [Return] 499250100 [4.992e8], 0
    │   │   │   ├─ [1013] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::positions(0xd4c9437bd39b24a03763ba043adb620f1f1c5132e8b9f2d70fce53696b5151ae) [staticcall]
    │   │   │   │   └─ ← [Return] 979544217507 [9.795e11], 85045089266861933740446074832329 [8.504e31], 85172746236841740717784271102231 [8.517e31], 0, 0
    │   │   │   ├─ [696] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::slot0() [staticcall]
    │   │   │   │   └─ ← [Return] 79228162514264337593543950336 [7.922e28], 0, 1, 100, 100, 0, true
    │   │   │   └─ ← [Return] [0, 0, 0, 0], 340282366920938463463374607431268961356 [3.402e38], 0
    │   │   ├─ [1942] SemiFungiblePositionManager::getAccountPremium(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74, 0x2304e805a66ee4627561F470475EAf9dE34D85F7, 0, 10, 20, 8388607 [8.388e6], 0, 4) [staticcall]
    │   │   │   └─ ← [Return] 4610416639687 [4.61e12], 4617333714248 [4.617e12]
    │   │   ├─ [1453] SemiFungiblePositionManager::getAccountLiquidity(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74, 0x2304e805a66ee4627561F470475EAf9dE34D85F7, 0, 10, 20) [staticcall]
    │   │   │   └─ ← [Return] 6722296912845509826321440690242695012766030403491 [6.722e48]
    │   │   ├─ emit OptionBurnt(recipient: 0x0000000000000000000000000000001234567891, positionSize: 499999500 [4.999e8], tokenId: 1267651733651528342013150880167 [1.267e30], premiaByLeg: [0, 0, 0, 0])
>   │   │   ├─ [12744] 0xaD89B4fd8F609caa0CAa087c01dB16f51Cc32084::settleBurn(0x0000000000000000000000000000001234567891, 0, 499250100 [4.992e8], -499250100 [-4.992e8], 0, 88257235459544876001772053827994779808 [8.825e37])
    │   │   │   ├─ [12505] CollateralTracker::1d3e716e(00000000000000000000000000000000000000000000000000000012345678910000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001dc1f3b4ffffffffffffffffffffffffffffffffffffffffffffffffffffffffe23e0c4c0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000004265b9aa82bf202012711964000000a02304e805a66ee4627561f470475eaf9de34d85f70178bc008cd5dc9dd44bc3baad07e7b0568bcff19478bc008cd5dc9dd44bc3baad07e7b0568bcff194f674deda0f700f74e6675ad569cb089d0d0e9f613cd4e8a3644ddc8b16954a9f50fd0dc0185161ac00000000000000000000000000000000000000000001f4007c) [delegatecall]
    │   │   │   │   ├─ [4009] RiskEngine::updateInterestRate(100000000000000 [1e14], 6585396285046963155095886422008189160681051 [6.585e42]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 317217227 [3.172e8], 1268295158 [1.268e9]
    │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   └─ ← [Return] 0
>   │   │   ├─ [12564] 0x9aFAb8Ead9EC80cD40363C037C2DFbe12dDBa14D::settleBurn(0x0000000000000000000000000000001234567891, 0, 0, 0, 0, 88257235459544876001772053827994779808 [8.825e37])
    │   │   │   ├─ [12325] CollateralTracker::1d3e716e(00000000000000000000000000000000000000000000000000000012345678910000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000004265b9aa82bf202012711964000000a02304e805a66ee4627561f470475eaf9de34d85f700f674deda0f700f74e6675ad569cb089d0d0e9f6178bc008cd5dc9dd44bc3baad07e7b0568bcff194f674deda0f700f74e6675ad569cb089d0d0e9f613cd4e8a3644ddc8b16954a9f50fd0dc0185161ac00000000000000000000000000000000000000000001f4007c) [delegatecall]
    │   │   │   │   ├─ [4009] RiskEngine::updateInterestRate(0, 6585396243508588286817265393764218526920283 [6.585e42]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 317074541 [3.17e8], 1268295149 [1.268e9]
    │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000000
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [386] PanopticMath::hasNoDuplicateTokenIds([]) [delegatecall]
    │   │   │   └─ ← [Return] true
    │   │   ├─ [2529] SemiFungiblePositionManager::getCurrentTick(0x0000000000000000000000004364bc31e1a78a5368c422cd3dfb6fcedeb90f74) [staticcall]
    │   │   │   ├─ [696] 0x4364BC31e1A78a5368C422Cd3dfB6FceDEB90F74::slot0() [staticcall]
    │   │   │   │   └─ ← [Return] 79228162514264337593543950336 [7.922e28], 0, 1, 100, 100, 0, true
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [5715] RiskEngine::getSolvencyTicks(0, 74738209495215746795597743391008221985693311380285640030731442434076690821042 [7.473e76]) [staticcall]
    │   │   │   └─ ← [Return] [0], 0
    │   │   └─ ← [Stop]
    │   └─ ← [Return]
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.75s (100.30ms CPU time)

Ran 1 test suite in 2.04s (1.75s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The trace output above displays the last two interactions with `PanopticPool`, where Charlie settles his own premium, then burns the position.

In the first interaction (settle premium), the commission fee is computed to 0 for both `CollateralTracker0` and `CollateralTracker1` - This can be observed via the `CommissionPaid` event.

In the second interaction (burn), the commission fee is skipped - This can be confirmed due to the absence of the `CommissionPaid` event.

As a result, Charlie avoided paying any commission fees for his position.

## RiskEngine::_computeDelayedSwap() does not return the correct value

## Vulnerability Detail

```solidity
function _computeDelayedSwap(
    TokenId tokenId,
    uint128 positionSize,
    uint256 index,
    uint256 partnerIndex,
    int24 atTick
) internal view returns (uint256) {
    unchecked {
        // can only be called when partnerIndex is the credit
        LeftRightUnsigned amountsMoved = PanopticMath.getAmountsMoved(
            tokenId,
            positionSize,
            index,
            false
        );

        LeftRightUnsigned amountsMovedP = PanopticMath.getAmountsMoved(
            tokenId,
            positionSize,
            partnerIndex,
            false
        );

        uint256 loanAmount = tokenId.tokenType(index) == 0
            ? amountsMoved.rightSlot()
            : amountsMoved.leftSlot();
        uint256 required = Math.mulDivRoundingUp(
            loanAmount,
            SELLER_COLLATERAL_RATIO + DECIMALS,
            DECIMALS
        );

        uint256 creditAmount = tokenId.tokenType(partnerIndex) == 0
            ? amountsMovedP.rightSlot()
            : amountsMovedP.leftSlot();

        uint256 convertedCredit = tokenId.tokenType(partnerIndex) == 0
            ? PanopticMath.convert0to1RoundingUp(creditAmount, Math.getSqrtRatioAtTick(atTick))
            : PanopticMath.convert1to0RoundingUp(creditAmount, Math.getSqrtRatioAtTick(atTick));

>       if (required > convertedCredit) {
>           return required;
>       } else {
>           return convertedCredit;
>       }
    }
}
```

`RiskEngine::_computeDelayedSwap()` returns the required amount for delayed swaps.

These positions have a `riskPartner` that provides credit. 

When the converted credit amount is not able to fully cover the required amount, the difference between required and credit should be returned.

However, the current implementation returns the full required amount - The credits for the `riskPartner` are not included.

On the contrary, when `required < convertedCredit`, `convertedCredit` will be returned, which will be larger than `required` - This is incorrect as a user’s margin requirements should be capped at `required`.

(No issue when `required = convertedCredit`, which is also covered by else-branch).

```solidity
function _getRequiredCollateralSingleLegPartner(
    TokenId tokenId,
    uint256 index,
    uint128 positionSize,
    int24 atTick,
    int16 poolUtilization
) internal view returns (uint256) {
    // extract partner index (associated with another liquidity chunk)
    uint256 partnerIndex = tokenId.riskPartner(index);

    // In the following, we check whether the risk partner of this leg is itself
    // or another leg in this position.
    // Handles case where riskPartner(i) != i ==> leg i has a risk partner that is another leg
    // @dev In summary, the allowed risk partners:
    //
    // PURE OPTIONS
    // -Short Strangles/Straddles (short put + short call) = each leg's basic requirement is 50% less
    // -Vertical Spreads and Calendar Spreads (short put + long put) or (short call + long call) = requirement is max loss
    // -Synthetic Stocks (short put + long call) or (short call + long put) = requirement is short leg only
    //
    // FUNDED OPTIONS
    // -Prepaid long option (long put or call + credit) "Purchases pre-pays for the cost of the option" = requirement is max(long - credit, 1)
    // -Upfront short option (short put or call + loan) "Upfront payment to seller" = requirement is max(loan, short option)
    // -Option-protected loan (long put or call + loan) "Get a loan with an embedded long option for capital protection" = requirement is max(loan, short option)
    // -Cash-Secured Option (short put or call + credit) "Allocate collateral to that specific option" = requirement is max(short - credit, 1)
    //
    // TOKEN TRANSFERS
>   // - Delayed Swap (credit at one strike, loan at another; different amounts = effective swap) = requirement is max(loan0 - convert1to0(credit), 1) or max(loan1 - convert0to1(credit), 1)
    
    // SNIP
}
```

The code comments in `RiskEngine::_getRequiredCollateralSingleLegPartner()` mention that the required amount should be `max(loan0 - convert1to0(credit), 1)` or `max(loan1 - convert0to1(credit), 1)`.

Currently, the required amount is `max(loan0, convert1to0(credit))` or `max(loan1, convert0to1(credit))`.

## Impact

User’s margin requirements will be higher than intended when making delayed swaps due to omitting credits / returning incorrect value.

## Recommendation

```solidity
function _computeDelayedSwap(
    TokenId tokenId,
    uint128 positionSize,
    uint256 index,
    uint256 partnerIndex,
    int24 atTick
) internal view returns (uint256) {
    unchecked {
        // SNIP

        uint256 creditAmount = tokenId.tokenType(partnerIndex) == 0
            ? amountsMovedP.rightSlot()
            : amountsMovedP.leftSlot();

        uint256 convertedCredit = tokenId.tokenType(partnerIndex) == 0
            ? PanopticMath.convert0to1RoundingUp(creditAmount, Math.getSqrtRatioAtTick(atTick))
            : PanopticMath.convert1to0RoundingUp(creditAmount, Math.getSqrtRatioAtTick(atTick));

        if (required > convertedCredit) {
-           return required;
+           return required - convertedCredit;
        } else {
-           return convertedCredit;
+           return required;
        }
    }
}
```

## PoC
```solidity
// test/foundry/core/Misc.t.sol
// ❯ forge test --match-test test_computeDelayedSwapPoC -vvvv

function test_computeDelayedSwapPoC() public {
    swapperc = new SwapperC();
    vm.startPrank(Swapper);
    token0.mint(Swapper, type(uint128).max);
    token1.mint(Swapper, type(uint128).max);
    token0.approve(address(swapperc), type(uint128).max);
    token1.approve(address(swapperc), type(uint128).max);  

    {
        poolId = uint40(uint256(PoolId.unwrap(poolKey.toId()))) + uint64(uint256(vegoid) << 40);
        poolId += uint64(uint24(uniPool.tickSpacing())) << 48;
    }

    TokenId tokenId = TokenId.wrap(0).addPoolId(poolId)
        .addLeg(
            0,  // legIndex
            1,  // _optionRatio
            0,  // _asset
            0,  // _isLong
            0,  // _tokenType
            0,  // _riskPartner
            15, // _strike
            1   // _width
        ).addLeg(
            1,  // legIndex
            1,  // _optionRatio
            0,  // _asset
            1,  // _isLong
            1,  // _tokenType
            1,  // _riskPartner
            0, // _strike
            0   // _width
        );
    $posIdList.push(tokenId); 

    // @audit-info sanity check - option mint success
    vm.startPrank(Bob);
    mintOptions(
        pp,
        $posIdList,
        1_000_000,
        0,
        Constants.MIN_POOL_TICK,
        Constants.MAX_POOL_TICK,
        true
    );
    
    // ------------------------------------------------------------
    // @audit-info RiskEngine::_computeDelayedSwap() Raw Computation
    uint128 positionSize = 1_000_000;
    uint256 index = 0;
    uint256 partnerIndex = 1;
    int24 atTick = currentTick; 

    // Checks in _getRequiredCollateralSingleLegPartner() for _computeDelayedSwap() flow
    assertTrue(tokenId.asset(partnerIndex) == tokenId.asset(index));
    assertTrue(tokenId.optionRatio(partnerIndex) == tokenId.optionRatio(index));

    assertTrue(tokenId.width(index) > 0 != tokenId.width(partnerIndex) > 0);
    assertTrue(tokenId.tokenType(index) != tokenId.tokenType(partnerIndex));
    assertTrue(tokenId.isLong(index) != tokenId.isLong(partnerIndex));
    
    uint256 required = _computeDelayedSwapRequired(tokenId, positionSize, index);
    uint256 convertedCredit = _computeDelayedSwapCredits(tokenId, positionSize, partnerIndex, atTick);
    // --------------------------------------------------------------

    uint256 expectedAmount;
    if (required > convertedCredit) {
        expectedAmount = required - convertedCredit;
    } else {
        required;
    }

    // @audit-info sanity checks
    assertTrue(convertedCredit != 0);
    assertTrue(required > convertedCredit);

    // @audit-info current implementation does not subtract credits
    uint256 actualAmount = _computeDelayedSwap(tokenId, positionSize, index, partnerIndex, atTick);
    assertTrue(expectedAmount != actualAmount);
    assertEq(actualAmount, 1199999);
    assertEq(required, 1199999);
    assertEq(expectedAmount, 199999);
}

// @audit-info Helper method - Parsed required amount computation from RiskEngine::_computeDelayedSwap()
function _computeDelayedSwapRequired(TokenId tokenId, uint128 positionSize, uint256 index) internal returns (uint256 required) {
    LeftRightUnsigned amountsMoved = PanopticMath.getAmountsMoved(
        tokenId,
        positionSize,
        index,
        false
    );
    uint256 loanAmount = tokenId.tokenType(index) == 0 
            ? amountsMoved.rightSlot() // token0
            : amountsMoved.leftSlot(); // token1
    required = Math.mulDivRoundingUp(
        loanAmount,
        2_000_000 + 10_000_000,
        10_000_000 // DECIMALS
    ); 
}

// @audit-info Helper method - Parsed converted credits computation from RiskEngine::_computeDelayedSwap()
function _computeDelayedSwapCredits(TokenId tokenId, uint128 positionSize, uint256 partnerIndex, int24 atTick) internal returns (uint256 convertedCredit) {
    LeftRightUnsigned amountsMovedP = PanopticMath.getAmountsMoved(
        tokenId,
        positionSize,
        partnerIndex,
        false
    );
    uint256 creditAmount = tokenId.tokenType(partnerIndex) == 0
            ? amountsMovedP.rightSlot()
            : amountsMovedP.leftSlot(); 
    convertedCredit = tokenId.tokenType(partnerIndex) == 0
            ? PanopticMath.convert0to1RoundingUp(creditAmount, Math.getSqrtRatioAtTick(atTick))
            : PanopticMath.convert1to0RoundingUp(creditAmount, Math.getSqrtRatioAtTick(atTick));
}

// @audit-info Copied from RiskEngine::_computeDelayedSwap()
function _computeDelayedSwap(
    TokenId tokenId,
    uint128 positionSize,
    uint256 index,
    uint256 partnerIndex,
    int24 atTick
) internal view returns (uint256) {
    unchecked {
        // can only be called when partnerIndex is the credit
        LeftRightUnsigned amountsMoved = PanopticMath.getAmountsMoved(
            tokenId,
            positionSize,
            index,
            false
        );

        LeftRightUnsigned amountsMovedP = PanopticMath.getAmountsMoved(
            tokenId,
            positionSize,
            partnerIndex,
            false
        );

        uint256 loanAmount = tokenId.tokenType(index) == 0
            ? amountsMoved.rightSlot() // token0
            : amountsMoved.leftSlot(); // token1
        uint256 required = Math.mulDivRoundingUp(
            loanAmount,
            2_000_000 + 10_000_000,
            10_000_000
        ); 

        uint256 creditAmount = tokenId.tokenType(partnerIndex) == 0
            ? amountsMovedP.rightSlot()
            : amountsMovedP.leftSlot(); 

        uint256 convertedCredit = tokenId.tokenType(partnerIndex) == 0
            ? PanopticMath.convert0to1RoundingUp(creditAmount, Math.getSqrtRatioAtTick(atTick))
            : PanopticMath.convert1to0RoundingUp(creditAmount, Math.getSqrtRatioAtTick(atTick));

        if (required > convertedCredit) {
            return required; 
        } else {
            return convertedCredit;
        }
    }
}
```

## QA Report

## [L-01] `commissionPaidBuilder` value in `CommissionPaid` event is incorrectly emitted in `CollateralTracker::settleMint()` / `CollateralTracker::settleBurn()`

## Vulnerability Detail

```solidity
function settleBurn(
    address optionOwner,
    int128 longAmount,
    int128 shortAmount,
    int128 ammDeltaAmount,
    int128 realizedPremium,
    RiskParameters riskParameters
) external onlyPanopticPool returns (int128) {
    // SNIP

    if (realizedPremium != 0) {
        // SNIP

        uint256 sharesToBurn = Math.mulDivRoundingUp(commissionFee, _totalSupply, _totalAssets);

        if (riskParameters.feeRecipient() == 0) {
            _burn(optionOwner, sharesToBurn);
            emit CommissionPaid(optionOwner, address(0), commissionFee, 0);
        } else {
            unchecked {
                _transferFrom(
                    optionOwner,
                    address(riskEngine()),
                    (sharesToBurn * riskParameters.protocolSplit()) / DECIMALS
                );
                _transferFrom(
                    optionOwner,
                    address(uint160(riskParameters.feeRecipient())),
                    (sharesToBurn * riskParameters.builderSplit()) / DECIMALS
                );
                emit CommissionPaid( 
                    optionOwner,
                    address(uint160(riskParameters.feeRecipient())),
                    uint128((commissionFee * riskParameters.protocolSplit()) / DECIMALS),
>                   uint128((commissionFee * riskParameters.protocolSplit()) / DECIMALS) // @audit
                );
            }
        }
    }

    return tokenPaid;
}
```

The last value in the `CommissionPaid` event should refer to the builder split.

However, the ***protocol split*** is returned instead.

NOTE: This issue also occurs in `CollateralTracker::settleMint()`

## Recommendation

```solidity
function settleBurn(
    address optionOwner,
    int128 longAmount,
    int128 shortAmount,
    int128 ammDeltaAmount,
    int128 realizedPremium,
    RiskParameters riskParameters
) external onlyPanopticPool returns (int128) {
    // SNIP

    if (realizedPremium != 0) {
        // SNIP

        uint256 sharesToBurn = Math.mulDivRoundingUp(commissionFee, _totalSupply, _totalAssets);

        if (riskParameters.feeRecipient() == 0) {
            _burn(optionOwner, sharesToBurn);
            emit CommissionPaid(optionOwner, address(0), commissionFee, 0);
        } else {
            unchecked {
                _transferFrom(
                    optionOwner,
                    address(riskEngine()),
                    (sharesToBurn * riskParameters.protocolSplit()) / DECIMALS
                );
                _transferFrom(
                    optionOwner,
                    address(uint160(riskParameters.feeRecipient())),
                    (sharesToBurn * riskParameters.builderSplit()) / DECIMALS
                );
                emit CommissionPaid( 
                    optionOwner,
                    address(uint160(riskParameters.feeRecipient())),
                    uint128((commissionFee * riskParameters.protocolSplit()) / DECIMALS),
-                   uint128((commissionFee * riskParameters.protocolSplit()) / DECIMALS)
+                   uint128((commissionFee * riskParameters.builderSplit()) / DECIMALS) 
                );
            }
        }
    }

    return tokenPaid;
}
```

---

## [L-02] Incorrect event value emitted in `RiskEngine::unlockPool()`

## Vulnerability Detail

```solidity
function lockPool(PanopticPool pool) external onlyGuardian {
    emit GuardianSafeModeUpdated(true);
    pool.lockSafeMode();
}

/// @notice Removes the forced safe-mode lock on a PanopticPool.
/// @dev Restores the pool to using only the automatically computed safe-mode level.
/// @param pool The PanopticPool to unlock.
function unlockPool(PanopticPool pool) external onlyGuardian {
>   emit GuardianSafeModeUpdated(true); // @audit should be false
    pool.unlockSafeMode();
}
```

Whether the Guardian locks or unlocks safe mode, `GuardianSafeModeUpdated` will always emit `true`.

`GuardianSafeModeUpdated` should emit `false`, when safe mode is unlocked.

## Recommendation

```solidity
/// @notice Removes the forced safe-mode lock on a PanopticPool.
/// @dev Restores the pool to using only the automatically computed safe-mode level.
/// @param pool The PanopticPool to unlock.
function unlockPool(PanopticPool pool) external onlyGuardian {
-   emit GuardianSafeModeUpdated(true); 
+   emit GuardianSafeModeUpdated(false);
    pool.unlockSafeMode();
}
```

---

## [L-03] `BorrowRateUpdated` event is defined, but never emitted

## Vulnerability Detail

```solidity
contract RiskEngine {
    using Math for uint256;

    /// @notice Emitted when a borrow rate is updated.
    event BorrowRateUpdated(
        address indexed collateralToken,
        uint256 avgBorrowRate,
        uint256 rateAtTarget
    );
    
    // SNIP    
}
```

`RiskEngine` defines an event called `BorrowRateUpdated()`.

However, this event is never emitted anywhere in the system.

## Recommendation

Currently, `RiskEngine` does not have any state-changing logic, as all the methods are labeled `view` / `pure`. 

Consider moving the definition of `BorrowRateUpdated` to `CollateralTracker` and emit the event when borrow rate is updated in `CollateralTracker::_updateInterestRate()`.

---

## [L-04] `tickData` is always omitted by default in `OptionMinted` event

## Vulnerability Detail

```solidity
function _mintOptions(
    TokenId tokenId,
    uint128 positionSize,
    uint24 effectiveLiquidityLimit,
    address owner,
    int24[2] memory tickLimits,
    RiskParameters riskParameters
) internal returns (LeftRightSigned paidAmounts, int24 finalTick) {
    // SNIP

    {
        // update the users options balance of position `tokenId`
        // NOTE: user can't mint same position multiple times, so set the positionSize instead of adding
        PositionBalance balanceData = PositionBalanceLibrary.storeBalanceData(
            positionSize,    // positionSize
            poolUtilizations, // utilizations 
>           0 // @audit tickData
        );
>       s_positionBalance[owner][tokenId] = balanceData;

>       emit OptionMinted(owner, tokenId, balanceData); 
    }
}
```

In `PanopticPool::_mintOptions()`, `tickData` is never cached or formatted prior to writing `balanceData`.

Therefore, the user’s position data never stores any relevant `tickData` (`lastObservedTick`, `slowOracleTick`, `fastOracleTick`, `currentTick`).

The `OptionMinted` event will also emit empty `tickData`.

## Recommendation

Consider updating the logic in `_mintOptions()` to format and store the relevant `tickData`.

---

## [L-05] `IRiskEngine::getFeeRecipient()` return type inconsistency with implementation

## Vulnerability Detail

```solidity
 // contracts/interfaces/IRiskEngine.sol
 function getFeeRecipient(uint256 builderCode) external pure returns (uint128 feeRecipient);
 
 // contracts/RiskEngine.sol
 function getFeeRecipient(uint256 builderCode) external view returns (address feeRecipient) {
    feeRecipient = _computeBuilderWallet(builderCode);

     //Optional: enforce whitelist by checking that the contract actually exists
     if (builderCode != 0) {
        if (feeRecipient.code.length == 0) revert Errors.InvalidBuilderCode();
     }
}
```

The return value, `feeRecipient`, is of type `uint128` in `IRiskEngine`.

The return value, `feeRecipient`, is of type `address` in `RiskEngine`.

If a user / contract utilized `IRiskEngine::getFeeRecipient()`, the returned address would always be incorrect as the most significant 32 bits of the address would be truncated. 

## Recommendation

```solidity
-function getFeeRecipient(uint256 builderCode) external pure returns (uint128 feeRecipient);
+function getFeeRecipient(uint256 builderCode) external view returns (address feeRecipient);
```

---

## [QA-01] `MAX_OPEN_LEGS` is defined in both `PanopticPool` and `RiskEngine`

## Vulnerability Detail

`MAX_OPEN_LEGS` is defined in both `PanopticPool` and `RiskEngine` as constants. 

However, when this value is used within the system, it is always fetched from `RiskEngine::getRiskParameters()`.

## Recommendation

Consider removing `MAX_OPEN_LEGS` from `PanopticPool` , since this value is never read / used anywhere in the system and the `RiskEngine` value is always used instead. 

---

## [QA-02]  `TARGET_RATE_MASK` is defined in `CollateralTracker` but never used

## Description

`TARGET_RATE_MASK` is defined in both `types/MarketState.sol` and `contracts/CollateralTracker.sol`.

However, `TARGET_RATE_MASK` is only used in `MarketState::updateRateAtTarget()` - The `CollateralTracker` definition is outdated.

## Recommendation

Consider removing `TARGET_RATE_MASK` from `CollateralTracker.sol`

---

## [QA-03] `BUILDER_SALT` is defined in `RiskEngine` but never used

## Description

```solidity
bytes32 internal constant BUILDER_SALT = keccak256("panoptic.builder");
```

## Recommendation

Consider using this value in `RiskEngine::_computeBuilderWallet()` and `BuilderFactory::predictBuilderWallet()`.

```solidity
// BuilderFactory
-function predictBuilderWallet(uint48 builderCode) external view returns (address) {
+function predictBuilderWallet(uint128 builderCode, bytes32 _salt) external view returns (address) {
-   bytes32 salt = bytes32(uint256(builderCode));
+   bytes32 salt = keccak256(abi.encodePacked(bytes32(uint256(builderCode)), _salt));

    bytes32 initCodeHash = keccak256(
        abi.encodePacked(type(BuilderWallet).creationCode, abi.encode(address(this)))
    );

    bytes32 h = keccak256(abi.encodePacked(bytes1(0xff), address(this), salt, initCodeHash));

    return address(uint160(uint256(h)));
}

// RiskEngine
function _computeBuilderWallet(uint256 builderCode) internal view returns (address wallet) {
    if (builderCode == 0) return address(0);

-   bytes32 salt = bytes32(builderCode);
+   bytes32 salt = keccak256(abi.encodePacked(bytes32(uint256(builderCode)), BUILDER_SALT));

    bytes32 h = keccak256(
        abi.encodePacked(bytes1(0xff), BUILDER_FACTORY, salt, BUILDER_INIT_CODE_HASH)
    );

    wallet = address(uint160(uint256(h)));
}
```

---

## [QA-04] Missing input validation for `builderCode` in `PanopticPool::dispatch()`

## Vulnerability Detail

Currently, there is no input validation for `builderCode` in `PanopticPool::dispatch()`, which allows users to provide arbitrary builder codes that do not map to a valid `BuilderWallet` address.

## Recommendation

Consider checking if the computed `feeRecipient` address is valid.

```solidity
function getRiskParameters(
    int24 currentTick,
    OraclePack oraclePack,
    uint256 builderCode
) external view returns (RiskParameters) {
    uint8 safeMode = isSafeMode(currentTick, oraclePack);

    uint128 feeRecipient = uint256(uint160(_computeBuilderWallet(builderCode))).toUint128();
+   if (builderCode != 0) {
+       if (feeRecipient.code.length == 0) revert Errors.InvalidBuilderCode();
+   }

    return
        RiskParametersLibrary.storeRiskParameters(
            safeMode,
            NOTIONAL_FEE,
            PREMIUM_FEE,
            PROTOCOL_SPLIT,
            BUILDER_SPLIT,
            MAX_TWAP_DELTA_LIQUIDATION,
            MAX_SPREAD,
            BP_DECREASE_BUFFER,
            MAX_OPEN_LEGS,
            feeRecipient
        );
}
```
