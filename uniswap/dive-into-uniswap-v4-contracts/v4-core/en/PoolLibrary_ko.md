# Pool 라이브러리 (Pool Library)

Pool 라이브러리는 AMM(Automated Market Maker) 풀의 핵심 로직을 정의하며, [PoolManager](./PoolManager_ko.md) 및 [v3-core/Pool.sol](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol)의 로직과 유사합니다.

## 구조체 정의 (Struct Definitions)

### State

```solidity
struct State {
    Slot0 slot0;
    uint256 feeGrowthGlobal0X128;
    uint256 feeGrowthGlobal1X128;
    mapping(int128 => uint256) liquidity;
    mapping(int24 => TickInfo) ticks;
    mapping(int24 => mapping(int24 => Position.State)) positions;
}
```

State 구조체는 풀의 상태를 저장하며, 다음을 포함합니다:

* `slot0`: 풀의 전역 상태 저장
* `feeGrowthGlobal0X128` 및 `feeGrowthGlobal1X128`: 풀의 전역 수수료 증가분 저장
* `liquidity`: 각 틱의 현재 총 유동성
* `ticks`: 틱 정보
* `positions`: 포지션 정보

### Slot0

```solidity
struct Slot0 {
    // the current price
    uint160 sqrtPriceX96;
    // the current tick
    int24 tick;
    uint24 protocolFee;
    uint24 lpFee;
}
```

Slot0는 비용을 절약하기 위해 여러 변수를 하나의 슬롯에 압축하며, 다음을 포함합니다:

* `sqrtPriceX96`: 현재 $ \sqrt{P} $
* `tick`: 현재 틱, $ 1.0001^{tick} \approx P $ 만족
* `protocolFee`: 프로토콜 수수료. 상위 12비트는 token0의 수수료 비율, 하위 12비트는 token1의 수수료 비율을 나타냄
* `lpFee`: 유동성 공급자 수수료(LP fee). 24비트 숫자, 상위 4비트는 플래그, 하위 20비트는 수수료 비율을 나타내며, `1000000`은 `100%`를 의미

Uniswap v3와 비교했을 때, `Slot0`에는 다음과 같은 변경 사항이 있습니다:

* `observationIndex`, `observationCardinality`, `observationCardinalityNext` 제거: 오라클 기능이 [Hooks](./HooksLibrary_ko.md)로 이동되었습니다.
* `unlocked` 제거: [Flash Accounting](./CurrencyDeltaLibrary_ko.md)이 도입되어 재진입 잠금(reentrancy lock)이 [PoolManager](./PoolManager_ko.md)로 이동되었습니다.

### TickInfo

```solidity
struct TickInfo {
    // the total liquidity that references this tick
    uint128 liquidityGross;
    // amount of net liquidity added (subtracted) when tick is crossed from left to right (right to left),
    int128 liquidityNet;
    // fee growth per unit of liquidity on the _other_ side of this tick (relative to the current tick)
    // only has relative meaning, not absolute — the value depends on when the tick is initialized
    uint256 feeGrowthOutside0X128;
    uint256 feeGrowthOutside1X128;
}
```

TickInfo는 틱 정보를 저장하며, [v3-core/Tick.sol](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/Tick.sol)과 유사합니다. 다음을 포함합니다:

* `liquidityGross`: 이 틱을 참조하는 총 유동성
* `liquidityNet`: 틱을 교차할 때 추가되거나 제거되는 순 유동성
* `feeGrowthOutside0X128` 및 `feeGrowthOutside1X128`: 틱 외부의 수수료 증가분

v3와 비교했을 때, `secondsPerLiquidityOutsideX128`, `tickCumulativeOutside`, `secondsOutside`, `initialized` 필드가 제거되었습니다. 이는 오라클 기능이 Hooks로 이동되었고 `liquidityGross`가 틱의 초기화 여부를 나타내기에 충분하기 때문입니다.

## 함수 정의 (Function Definitions)

### initialize

풀을 초기화합니다.

```solidity
/// @notice Initializes the pool state
/// @param self The pool state state to initialize
/// @param sqrtPriceX96 The initial square root price
/// @param lpFee The initial LP fee
/// @return tick The initial tick
function initialize(State storage self, uint160 sqrtPriceX96, uint24 lpFee) internal returns (int24 tick) {
    if (self.slot0.sqrtPriceX96() != 0) PoolAlreadyInitialized.selector.revertWith();

    tick = TickMath.getTickAtSqrtPrice(sqrtPriceX96);

    // the initial protocolFee is 0 so doesn't need to be set
    self.slot0 = Slot0.wrap(bytes32(0)).setSqrtPriceX96(sqrtPriceX96).setTick(tick).setLpFee(lpFee);
}
```

이 함수는 다음과 같은 작업을 수행합니다:
1. 풀이 이미 초기화되었는지 확인합니다.
2. 초기 가격 `sqrtPriceX96`에 해당하는 `tick`을 계산합니다.
3. `slot0`를 초기화합니다.

### checkTick

틱의 유효성을 확인합니다.

```solidity
function checkTicks(int24 tickLower, int24 tickUpper) private pure {
    if (tickLower >= tickUpper) TicksMisordered.selector.revertWith();
    if (tickLower < TickMath.MIN_TICK) TickLowerOutOfBounds.selector.revertWith();
    if (tickUpper > TickMath.MAX_TICK) TickUpperOutOfBounds.selector.revertWith();
}
```

### modifyLiquidity

유동성을 수정합니다.

```solidity
struct ModifyLiquidityParams {
    address owner;
    int24 tickLower;
    int24 tickUpper;
    int128 liquidityDelta;
    int24 tickSpacing;
    bytes32 salt;
}

/// @notice Effect some changes to a position
/// @param self The pool state to update
/// @param params The parameters for modifying the liquidity
/// @return delta The balance delta of the pool
/// @return feeDelta The fee delta of the pool
function modifyLiquidity(State storage self, ModifyLiquidityParams memory params)
    internal
    returns (BalanceDelta delta, BalanceDelta feeDelta)
{
    int128 liquidityDelta = params.liquidityDelta;
    int24 tickLower = params.tickLower;
    int24 tickUpper = params.tickUpper;
    checkTicks(tickLower, tickUpper);

    {
        ModifyLiquidityState memory state;

        // if we need to update the ticks, do it
        if (liquidityDelta != 0) {
            (state.flippedLower, state.liquidityGrossAfterLower) =
                updateTick(self, tickLower, liquidityDelta, false);
            (state.flippedUpper, state.liquidityGrossAfterUpper) = updateTick(self, tickUpper, liquidityDelta, true);

            if (state.liquidityGrossAfterLower > TickMath.MAX_LIQUIDITY_GROSS_PER_TICK) {
                TickLiquidityOverflow.selector.revertWith(tickLower);
            }
            if (state.liquidityGrossAfterUpper > TickMath.MAX_LIQUIDITY_GROSS_PER_TICK) {
                TickLiquidityOverflow.selector.revertWith(tickUpper);
            }
        }
```

`modifyLiquidity` 함수는 다음과 같은 작업을 수행합니다:

1. `checkTicks`를 호출하여 틱 범위의 유효성을 확인합니다.
2. `liquidityDelta`가 0이 아닌 경우:
    * `updateTick`을 호출하여 `tickLower` 및 `tickUpper`의 `TickInfo`를 업데이트하고, 해당 틱이 초기화되거나 초기화 해제되었는지(`flipped`) 확인합니다.
    * 틱당 최대 유동성을 초과하지 않는지 확인합니다.

(계속)

```solidity
        (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) =
            getFeeGrowthInside(self, tickLower, tickUpper);

        (uint256 feesOwed0, uint256 feesOwed1) = self.positions.get(
            params.owner, tickLower, tickUpper, params.salt
        ).update(liquidityDelta, feeGrowthInside0X128, feeGrowthInside1X128);

        // if the ticks are flipped, we need to update the tick map
        if (liquidityDelta != 0) {
            if (state.flippedLower) {
                self.ticks.updateTick(tickLower, params.tickSpacing);
            }
            if (state.flippedUpper) {
                self.ticks.updateTick(tickUpper, params.tickSpacing);
            }
        }
        
        // ...
    }
}
```

(계속)

3. `getFeeGrowthInside`를 호출하여 지정된 틱 범위 내의 수수료 증가분을 계산합니다.
4. `self.positions.update`를 호출하여 포지션 정보를 업데이트하고 누적된 수수료(`feesOwed0`, `feesOwed1`)를 계산합니다.
    * `Position.update`는 포지션의 유동성과 수수료 성장 값을 업데이트합니다.
5. 틱이 `flipped`된 경우, `self.ticks.updateTick`(TickBitmap 업데이트)을 호출하여 비트맵을 업데이트합니다.

```solidity
        // ...
        delta = toBalanceDelta(
            feesOwed0.toInt128(), // feeDelta0
            feesOwed1.toInt128()  // feeDelta1
        );
        feeDelta = delta;

        if (liquidityDelta != 0) {
            (uint256 amount0, uint256 amount1) = SqrtPriceMath.getAmountsForLiquidity(
                self.slot0.sqrtPriceX96(),
                TickMath.getSqrtRatioAtTick(tickLower),
                TickMath.getSqrtRatioAtTick(tickUpper),
                uint128(liquidityDelta.abs())
            );

            if (liquidityDelta > 0) {
                delta = delta - toBalanceDelta(amount0.toInt128(), amount1.toInt128());
            } else {
                delta = delta + toBalanceDelta(amount0.toInt128(), amount1.toInt128());
            }

            // update global liquidity
            // try to update the liquidity of the current tick, if it fails, revert
            // only need to update the global liquidity if the current tick is in the range
            if (self.slot0.tick() >= tickLower && self.slot0.tick() < tickUpper) {
                self.liquidity.update(self.slot0.tick(), liquidityDelta);
            }
        }
    }
```

(계속)

6. `feeDelta`를 계산합니다. 이는 사용자가 수취할 수 있는 수수료입니다.
7. `liquidityDelta`가 0이 아닌 경우:
    * `SqrtPriceMath.getAmountsForLiquidity`를 사용하여 유동성 변경에 필요한 토큰 양(`amount0`, `amount1`)을 계산합니다.
    * `liquidityDelta`의 부호에 따라 `delta`를 조정합니다. 유동성을 추가하면 `delta`가 감소(사용자가 예치)하고, 제거하면 `delta`가 증가(사용자가 인출)합니다.
    * 현재 틱이 포지션 범위 내에 있다면, 현재 활성 유동성(`self.liquidity`)을 업데이트합니다.

### swap

토큰을 스왑합니다.

```solidity
struct SwapParams {
    int24 tickSpacing;
    bool zeroForOne;
    int256 amountSpecified;
    uint160 sqrtPriceLimitX96;
    uint24 lpFeeOverride;
}

/// @notice Executes a swap against the state
/// @param self The pool state to update
/// @param params The parameters for the swap
/// @return delta The balance delta of the pool, considering the swap and the protocol fees
function swap(State storage self, PoolId id, SwapParams memory params, Currency inputCurrency)
    internal
    returns (BalanceDelta delta)
{
```

Uniswap v3와 유사하게 `swap` 함수는 while 루프를 사용하여 거래가 완료되거나 가격 제한에 도달할 때까지 틱을 하나씩 교차하며 거래를 실행합니다.

```solidity
    // ...
    Slot0 slot0Start = self.slot0;
    // ...
    // initialize state struct
    SwapState memory state = SwapState({
        amountSpecifiedRemaining: params.amountSpecified,
        amountCalculated: 0,
        sqrtPriceX96: slot0Start.sqrtPriceX96(),
        tick: slot0Start.tick(),
        feeGrowthGlobalX128: params.zeroForOne ? self.feeGrowthGlobal0X128 : self.feeGrowthGlobal1X128,
        protocolFee: slot0Start.protocolFee(),
        lpFee: params.lpFeeOverride.isOverride() ? params.lpFeeOverride.removeOverride() : slot0Start.lpFee(),
        liquidity: self.liquidity[slot0Start.tick()]
    });
    // ...
```

먼저 현재 풀 상태(`Slot0`)와 스왑 상태(`SwapState`)를 초기화합니다. `SwapState`는 스왑 실행 중 변경되는 상태를 추적합니다.

```solidity
    // ...
    while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != params.sqrtPriceLimitX96) {
        StepComputations memory step;

        step.sqrtPriceStartX96 = state.sqrtPriceX96;

        (step.tickNext, step.initialized) =
            self.ticks.nextInitializedTickWithinOneWord(state.tick, params.tickSpacing, params.zeroForOne);
        
        // ...
        
        (state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
            state.sqrtPriceX96,
            (
                params.zeroForOne
                    ? step.sqrtPriceNextX96 < params.sqrtPriceLimitX96
                    : step.sqrtPriceNextX96 > params.sqrtPriceLimitX96
            ) ? params.sqrtPriceLimitX96 : step.sqrtPriceNextX96,
            state.liquidity,
            state.amountSpecifiedRemaining,
            state.lpFee
        );
        
        // ...
    }
    // ...
```

while 루프 내에서:
1. `TickBitmap.nextInitializedTickWithinOneWord`를 사용하여 다음 초기화된 틱(`step.tickNext`)을 찾습니다.
2. `SwapMath.computeSwapStep`을 호출하여 현재 가격에서 다음 틱 가격까지(또는 제한 가격까지) 스왑을 실행합니다.
    * 이 함수는 새로운 가격, 입력/출력 금액, 수수료 금액을 계산합니다.

```solidity
        if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
            // if the tick is initialized, run the tick transition
            if (step.initialized) {
                int128 liquidityNet = crossTick(
                    self,
                    step.tickNext,
                    (params.zeroForOne ? state.feeGrowthGlobalX128 : self.feeGrowthGlobal0X128),
                    (params.zeroForOne ? self.feeGrowthGlobal1X128 : state.feeGrowthGlobalX128)
                );
                // if we're moving leftward, we interpret liquidityNet as the opposite sign
                // safe because liquidityNet cannot be type.int128.min
                if (params.zeroForOne) liquidityNet = -liquidityNet;

                state.liquidity = LiquidityMath.addDelta(state.liquidity, liquidityNet);
            }

            state.tick = params.zeroForOne ? step.tickNext - 1 : step.tickNext;
        } else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
            // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
            state.tick = TickMath.getTickAtSqrtPrice(state.sqrtPriceX96);
        }
```

3. 틱을 교차하는 경우(`state.sqrtPriceX96 == step.sqrtPriceNextX96`):
    * `crossTick`을 호출하여 틱 외부 수수료 증가분을 업데이트하고 `liquidityNet`을 가져옵니다.
    * 현재 유동성(`state.liquidity`)을 업데이트합니다.
    * `state.tick`을 업데이트합니다.
4. 모든 스왑이 완료되면 루프를 종료하고 최종 상태를 `slot0`에 저장합니다.
5. 마지막으로 `delta`를 반환합니다.

### donate

풀에 토큰을 기부합니다. 기부된 토큰은 풀의 유동성 공급자에게 분배됩니다.

```solidity
/// @notice Donates the given amount of currency to the pool
function donate(State storage self, uint256 amount0, uint256 amount1)
    internal
    returns (BalanceDelta delta)
{
    // ...
    if (amount0 > 0) {
        self.feeGrowthGlobal0X128 += FullMath.mulDiv(amount0, FixedPoint128.Q128, liquidity);
    }
    if (amount1 > 0) {
        self.feeGrowthGlobal1X128 += FullMath.mulDiv(amount1, FixedPoint128.Q128, liquidity);
    }

    return toBalanceDelta(-(amount0.toInt128()), -(amount1.toInt128()));
}
```

`donate`는 현재 활성 유동성이 0이 아니어야 합니다. 기부된 금액(`amount0`, `amount1`)은 전역 수수료 증가분(`feeGrowthGlobal`)에 더해져 유동성 공급자에게 분배됩니다. 반환되는 `delta`는 호출자가 지불해야 할 금액이므로 음수입니다.

### getFeeGrowthInside

특정 틱 범위 내의 수수료 증가분을 계산합니다.

```solidity
function getFeeGrowthInside(State storage self, int24 tickLower, int24 tickUpper)
    internal
    view
    returns (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128)
{
    // ...
}
```

Uniswap v3와 마찬가지로 `getFeeGrowthInside`는 현재 틱(`tickCurrent`)과 틱 범위(`tickLower`, `tickUpper`)의 위치 관계를 기반으로 범위 내 수수료를 계산합니다.

수수료 계산 로직: $ f_g $를 전역 수수료 증가분, $ f_o(i) $를 틱 $ i $의 외부 수수료 증가분이라고 할 때, 틱 $ i $ 아래의 수수료 $ f_b(i) $와 틱 $ i $ 위의 수수료 $ f_a(i) $는 다음과 같습니다.

$$
f_a(i) = \begin{cases} f_g - f_o(i) & \text{$i_c \geq i$}\\
f_o(i) & \text{$i_c < i$}\end{cases}
$$

$$
f_b(i) = \begin{cases} f_o(i) & \text{$i_c \geq i$}\\
f_g - f_o(i) & \text{$i_c < i$}\end{cases}
$$

따라서 두 틱 사이(하한 $ i_l $, 상한 $ i_u $)의 범위 내 수수료 $ f_r $은 다음과 같습니다:

$$
f_r = f_g - f_b(i_l) - f_a(i_u)
$$

이를 현재 틱 $ i_c $와 경계 틱 $ i_l, i_u $의 관계에 따라 정리하면 다음과 같습니다:

$$
f_r = \begin{cases} 
f_o(i_l) - f_o(i_u) & \text{$i_c < i_l < i_u$}\\
f_g - f_o(i_l) - f_o(i_u) & \text{$i_l \leq i_c < i_u$}\\
f_o(i_u) - f_o(i_l) & \text{$i_l < i_u \leq i_c$} \end{cases}
$$

`getFeeGrowthInside` 코드는 이 로직을 구현합니다.

### updateTick

틱의 유동성을 업데이트합니다.

```solidity
function updateTick(State storage self, int24 tick, int128 liquidityDelta, bool upper)
    internal
    returns (bool flipped, uint128 liquidityGrossAfter)
{
    // ...
    // update liquidityGross
    liquidityGrossAfter = LiquidityMath.addDelta(liquidityGrossBefore, liquidityDelta);
    
    flipped = (liquidityGrossAfter == 0) != (liquidityGrossBefore == 0);

    if (liquidityGrossBefore == 0) {
        // initialize feeGrowthOutside if tick is being initialized
        if (tick <= self.slot0.tick()) {
            info.feeGrowthOutside0X128 = self.feeGrowthGlobal0X128;
            info.feeGrowthOutside1X128 = self.feeGrowthGlobal1X128;
        }
    }
    
    // update liquidityNet
    int128 liquidityNet = upper ? liquidityNetBefore - liquidityDelta : liquidityNetBefore + liquidityDelta;
    // ...
}
```

* `liquidityGross`를 업데이트합니다.
* 틱이 `flipped`되었는지(초기화 상태 변경 여부) 확인합니다.
* 틱이 새로 초기화되는 경우(`liquidityGrossBefore == 0`), 현재 틱보다 작거나 같으면 `feeGrowthOutside`를 전역 수수료로 초기화합니다.
    > $$
    > f_o := \begin{cases} f_g & \text{$i_c \geq i$}\\
    > 0 & \text{$i_c < i$} \end{cases}
    > $$
* `liquidityNet`을 업데이트합니다. `upper` 틱인 경우 `liquidityDelta`를 빼고, `lower` 틱인 경우 더합니다.
    * 틱이 왼쪽에서 오른쪽으로 이동할 때: `lower` 틱을 지나면 유동성이 추가(범위 진입)되고, `upper` 틱을 지나면 유동성이 제거(범위 이탈)됩니다.
    * 틱이 오른쪽에서 왼쪽으로 이동할 때: `upper` 틱을 지나면 유동성이 추가되고, `lower` 틱을 지나면 유동성이 제거됩니다.

### crossTick

틱을 교차합니다.

```solidity
function crossTick(State storage self, int24 tick, uint256 feeGrowthGlobal0X128, uint256 feeGrowthGlobal1X128)
    internal
    returns (int128 liquidityNet)
{
    unchecked {
        TickInfo storage info = self.ticks[tick];
        info.feeGrowthOutside0X128 = feeGrowthGlobal0X128 - info.feeGrowthOutside0X128;
        info.feeGrowthOutside1X128 = feeGrowthGlobal1X128 - info.feeGrowthOutside1X128;
        liquidityNet = info.liquidityNet;
    }
}
```

틱을 교차할 때 `feeGrowthOutside`를 업데이트합니다. Uniswap v3 백서에 소개된 바와 같이 업데이트 공식은 다음과 같습니다:

$$
f_o(i) := f_g - f_o(i)
$$

그리고 해당 틱의 `liquidityNet`을 반환하여 현재 활성 유동성을 업데이트하는 데 사용합니다.
