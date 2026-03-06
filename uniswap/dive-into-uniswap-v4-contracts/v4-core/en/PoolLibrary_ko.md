# Pool 라이브러리 (Pool Library)

Pool 라이브러리는 AMM(Automated Market Maker) 풀의 핵심 로직을 정의하며, [PoolManager](./PoolManager_ko.md) 및 [v3-core/Pool.sol](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol)의 로직과 유사합니다.

## 구조체 정의 (Struct Definitions)

### TickInfo

`TickInfo` 구조체는 풀의 각 `tick`에 대한 정보를 정의합니다. 여기에는 `liquidityGross`, `liquidityNet`, `feeGrowthOutside0X128`, `feeGrowthOutside1X128` 등이 포함됩니다.

```solidity
// info stored for each initialized individual tick
struct TickInfo {
    // the total position liquidity that references this tick
    uint128 liquidityGross;
    // amount of net liquidity added (subtracted) when tick is crossed from left to right (right to left),
    int128 liquidityNet;
    // fee growth per unit of liquidity on the _other_ side of this tick (relative to the current tick)
    // only has relative meaning, not absolute — the value depends on when the tick is initialized
    uint256 feeGrowthOutside0X128;
    uint256 feeGrowthOutside1X128;
}
```

Uniswap v3에서 설명한 것처럼:

* `liquidityGross`는 총 유동성을 나타내며, 해당 `tick`을 초기화해야 하는지 판단하는 데 사용됩니다.
    * `mint`면 유동성을 증가시키고, `burn`이면 유동성을 감소시킵니다.
    * 이 값은 포지션에서 그 `tick`이 하한인지 상한인지와는 무관하며, 오직 `mint` 또는 `burn`으로 얼마나 많은 유동성이 걸려 있는지만 반영합니다.
    * 어떤 `tick`이 서로 다른 포지션에서 `tickLower`와 `tickUpper`로 동시에 사용되면 `liquidityNet`은 0일 수 있지만, `liquidityGross`는 여전히 0보다 크므로 다시 초기화할 필요가 없습니다.

* `liquidityNet`은 순 유동성을 나타내며, `swap`이 `tick`을 통과할 때 전역 활성 유동성을 갱신하는 데 사용됩니다.
    * `tickLower`, 즉 하한(왼쪽 경계점)으로 사용되면 `liquidityDelta`를 더합니다.
    * `tickUpper`, 즉 상한(오른쪽 경계점)으로 사용되면 `liquidityDelta`를 뺍니다.

### State

`State` 구조체는 풀의 상태를 정의합니다:

```solidity
/// @notice The state of a pool
/// @dev Note that feeGrowthGlobal can be artificially inflated
/// For pools with a single liquidity position, actors can donate to themselves to freely inflate feeGrowthGlobal
/// atomically donating and collecting fees in the same unlockCallback may make the inflated value more extreme
struct State {
    Slot0 slot0;
    uint256 feeGrowthGlobal0X128;
    uint256 feeGrowthGlobal1X128;
    uint128 liquidity;
    mapping(int24 tick => TickInfo) ticks;
    mapping(int16 wordPos => uint256) tickBitmap;
    mapping(bytes32 positionKey => Position.State) positions;
}
```

여기서:

* `slot0`는 풀 수수료(LP 수수료, 프로토콜 수수료), 현재 가격 등 기본 정보를 정의합니다.
* `feeGrowthGlobal0X128`와 `feeGrowthGlobal1X128`는 각각 `token0`, `token1`의 전역 수수료 성장값을 나타냅니다.
* `liquidity`는 현재 풀의 총 활성 유동성을 나타냅니다.
* `ticks`는 각 `tick`의 정보를 저장하는 `mapping`입니다.
* `tickBitmap`은 각 `tick`의 비트맵 정보를 저장하는 `mapping`으로, 다음에 초기화된 `tick`을 빠르게 찾는 데 사용됩니다.
* `positions`는 각 포지션의 정보를 저장하는 `mapping`입니다.

#### Slot0

Uniswap v3에서 `Slot0`는 `struct`였지만, Uniswap v4에서는 가스 비용을 줄이기 위해 `slot0` 정보를 담는 `bytes32`로 바뀌었습니다.

`Slot0`의 레이아웃은 다음과 같습니다:

```
24 bits empty | 24 bits lpFee | 12 bits protocolFee 1->0 | 12 bits protocolFee 0->1 | 24 bits tick | 160 bits sqrtPriceX96
```

왼쪽에서 오른쪽으로(상위 비트에서 하위 비트 순으로) 보면:

* 24 bits empty: 패딩용 빈 공간으로, 현재는 사용되지 않습니다.
* 24 bits lpFee: 유동성 공급자 수수료
* 12 bits protocolFee 1->0: `token1 -> token0` 방향의 프로토콜 수수료
* 12 bits protocolFee 0->1: `token0 -> token1` 방향의 프로토콜 수수료
* 24 bits tick: 현재 가격에 대응하는 `tick`
* 160 bits sqrtPriceX96: 현재 가격의 `sqrtPriceX96`

`Slot0Library`는 `slot0`의 각 비트를 읽고 설정하는 메서드를 제공합니다.

### ModifyLiquidityParams

`ModifyLiquidityParams`의 정의는 다음과 같습니다:

```solidity
struct ModifyLiquidityParams {
    // the address that owns the position
    address owner;
    // the lower and upper tick of the position
    int24 tickLower;
    int24 tickUpper;
    // any change in liquidity
    int128 liquidityDelta;
    // the spacing between ticks
    int24 tickSpacing;
    // used to distinguish positions of the same owner, at the same tick range
    bytes32 salt;
}
```

구성 요소는 다음과 같습니다:

* `owner`: 포지션 소유자 주소
* `tickLower` 및 `tickUpper`: 포지션의 틱 범위
* `liquidityDelta`: 유동성 변화량. 유동성 추가는 양수, 제거는 음수
* `tickSpacing`: 틱 간격
* `salt`: 동일한 소유자와 동일한 틱 범위를 갖는 포지션을 구분하기 위한 값

### SwapParams

`SwapParams`의 정의는 다음과 같습니다:

```solidity
struct SwapParams {
    int256 amountSpecified;
    int24 tickSpacing;
    bool zeroForOne;
    uint160 sqrtPriceLimitX96;
    uint24 lpFeeOverride;
}
```

구성 요소는 다음과 같습니다:

* `amountSpecified`: 지정한 토큰 수량
  > 참고: Uniswap v4에서는 `amountSpecified`가 음수면 원하는 입력 토큰 수량(`exactInput`), 양수면 원하는 출력 토큰 수량(`exactOutput`)을 의미합니다.
* `tickSpacing`: 틱 간격
* `zeroForOne`: 스왑 방향. `true`면 `token0 -> token1`, `false`면 `token1 -> token0`
* `sqrtPriceLimitX96`: 스왑 가격 제한. 최댓값 또는 최솟값이 될 수 있음
* `lpFeeOverride`: LP 수수료. 풀의 기본 LP 수수료를 덮어쓸 때 사용

### SwapResult

`SwapResult`의 정의는 다음과 같습니다:

```solidity
// Tracks the state of a pool throughout a swap, and returns these values at the end of the swap
struct SwapResult {
    // the current sqrt(price)
    uint160 sqrtPriceX96;
    // the tick associated with the current price
    int24 tick;
    // the current liquidity in range
    uint128 liquidity;
}
```

구성 요소는 다음과 같습니다:

* `sqrtPriceX96`: 현재 가격
* `tick`: 현재 가격에 대응하는 틱
* `liquidity`: 현재 범위 내 활성 유동성

### StepComputations

`StepComputations`의 정의는 다음과 같습니다:

```solidity
struct StepComputations {
    // the price at the beginning of the step
    uint160 sqrtPriceStartX96;
    // the next tick to swap to from the current tick in the swap direction
    int24 tickNext;
    // whether tickNext is initialized or not
    bool initialized;
    // sqrt(price) for the next tick (1/0)
    uint160 sqrtPriceNextX96;
    // how much is being swapped in in this step
    uint256 amountIn;
    // how much is being swapped out
    uint256 amountOut;
    // how much fee is being paid in
    uint256 feeAmount;
    // the global fee growth of the input token. updated in storage at the end of swap
    uint256 feeGrowthGlobalX128;
}
```

구성 요소는 다음과 같습니다:

* `sqrtPriceStartX96`: 현재 단계 시작 시점의 가격
* `tickNext`: 현재 스왑 방향으로 진행할 때 다음에 도달할 틱
* `initialized`: `tickNext`가 초기화되어 있는지 여부
* `sqrtPriceNextX96`: 다음 틱의 가격
* `amountIn`: 현재 단계에서 입력되는 토큰 수량
* `amountOut`: 현재 단계에서 출력되는 토큰 수량
* `feeAmount`: 현재 단계에서 지불하는 수수료 수량
* `feeGrowthGlobalX128`: 입력 토큰의 전역 수수료 성장값. 스왑 종료 시 storage에 반영됨

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

### checkTicks

틱의 유효성을 확인합니다.

```solidity
function checkTicks(int24 tickLower, int24 tickUpper) private pure {
    if (tickLower >= tickUpper) TicksMisordered.selector.revertWith(tickLower, tickUpper);
    if (tickLower < TickMath.MIN_TICK) TickLowerOutOfBounds.selector.revertWith(tickLower);
    if (tickUpper > TickMath.MAX_TICK) TickUpperOutOfBounds.selector.revertWith(tickUpper);
}
```

`tickLower`가 `tickUpper`보다 작은지, 그리고 두 값이 모두 허용 가능한 틱 범위 안에 있는지 검사합니다.

Uniswap v3에서 설명한 것처럼, Uniswap v3가 지원하는 가격 범위( $\frac{token1}{token0}$ )는 $[2^{-128}, 2^{128}]$이며, 백서의 공식 6.1은 다음과 같습니다:

$$
p(i) = 1.0001^i
$$

따라서 최대 틱(`MAX_TICK`)은 다음과 같습니다:

$$
i = \lfloor log_{1.0001}{2^{128}} \rfloor = \lfloor 887272.7517970635 \rfloor = 887272
$$

최소 틱(`MIN_TICK`)은 다음과 같습니다:

$$
i = \lceil log_{1.0001}{2^{-128}} \rceil = \lceil -887272.7517970635 \rceil = -887272
$$

### setProtocolFee

프로토콜 수수료를 설정합니다.

```solidity
function setProtocolFee(State storage self, uint24 protocolFee) internal {
    self.checkPoolInitialized();
    self.slot0 = self.slot0.setProtocolFee(protocolFee);
}
```

풀이 초기화되었는지 먼저 확인한 뒤, `slot0`의 프로토콜 수수료 필드를 갱신합니다.

### setLPFee

LP 수수료를 설정합니다. 동적 수수료 풀만 LP 수수료를 변경할 수 있습니다.

```solidity
/// @notice Only dynamic fee pools may update the lp fee.
function setLPFee(State storage self, uint24 lpFee) internal {
    self.checkPoolInitialized();
    self.slot0 = self.slot0.setLpFee(lpFee);
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

스왑 작업을 실행하여 `token0`을 `token1`로, 혹은 그 반대로 교환합니다.

```solidity
/// @notice Executes a swap against the state, and returns the amount deltas of the pool
/// @dev PoolManager checks that the pool is initialized before calling
function swap(State storage self, SwapParams memory params)
    internal
    returns (BalanceDelta swapDelta, uint256 amountToProtocol, uint24 swapFee, SwapResult memory result)
{
```

전체 `swap` 로직은 Uniswap v3의 [swap](../../../dive-into-uniswap-v3-contracts/README_ko.md#swap)과 거의 동일합니다.

입력 파라미터는 다음과 같습니다:

* [State](#state) `self`: 풀의 상태
* [SwapParams](#swapparams) `params`: 스왑 수량, 방향, 가격 제한 등을 포함한 스왑 파라미터

반환값은 다음과 같습니다:

* `BalanceDelta swapDelta`: 스왑 후 풀의 토큰 잔액 변화
* `uint256 amountToProtocol`: 프로토콜 수수료
* `uint24 swapFee`: 스왑 수수료
* [SwapResult](#swapresult) `result`: 스왑 후 가격, `tick`, 유동성을 포함하는 결과 값

코드는 다음과 같습니다:

```solidity
    Slot0 slot0Start = self.slot0;
    bool zeroForOne = params.zeroForOne;

    uint256 protocolFee =
        zeroForOne ? slot0Start.protocolFee().getZeroForOneFee() : slot0Start.protocolFee().getOneForZeroFee();

    // the amount remaining to be swapped in/out of the input/output asset. initially set to the amountSpecified
    int256 amountSpecifiedRemaining = params.amountSpecified;
    // the amount swapped out/in of the output/input asset. initially set to 0
    int256 amountCalculated = 0;
    // initialize to the current sqrt(price)
    result.sqrtPriceX96 = slot0Start.sqrtPriceX96();
    // initialize to the current tick
    result.tick = slot0Start.tick();
    // initialize to the current liquidity
    result.liquidity = self.liquidity;
```

관련 파라미터를 초기화합니다.

```solidity
    // if the beforeSwap hook returned a valid fee override, use that as the LP fee, otherwise load from storage
    // lpFee, swapFee, and protocolFee are all in pips
    {
        uint24 lpFee = params.lpFeeOverride.isOverride()
            ? params.lpFeeOverride.removeOverrideFlagAndValidate()
            : slot0Start.lpFee();

        swapFee = protocolFee == 0 ? lpFee : uint16(protocolFee).calculateSwapFee(lpFee);
    }
```

훅이 새로운 `lpFee`를 반환하면 그 값을 사용하고, 그렇지 않으면 풀 생성 시 설정된 `lpFee`를 사용합니다.

프로토콜 수수료가 0이면 `lpFee`를 그대로 스왑 수수료 `swapFee`로 사용합니다. 그렇지 않으면 다음 식으로 스왑 수수료를 계산합니다. 여기서 `fee`의 단위는 BIP의 100분의 1, 즉 ${10}^{6}$입니다:

$$
swapFee = protocolFee + lpFee \cdot (1 - \frac{protocolFee}{{10}^{6}})
$$

`swapFee`는 `protocolFee`와 `lpFee`를 모두 포함하며, `lpFee`는 프로토콜 수수료를 제외한 뒤의 값입니다.

```solidity
    // a swap fee totaling MAX_SWAP_FEE (100%) makes exact output swaps impossible since the input is entirely consumed by the fee
    if (swapFee >= SwapMath.MAX_SWAP_FEE) {
        // if exactOutput
        if (params.amountSpecified > 0) {
            InvalidFeeForExactOut.selector.revertWith();
        }
    }

    // swapFee is the pool's fee in pips (LP fee + protocol fee)
    // when the amount swapped is 0, there is no protocolFee applied and the fee amount paid to the protocol is set to 0
    if (params.amountSpecified == 0) return (BalanceDeltaLibrary.ZERO_DELTA, 0, swapFee, result);
```

`swapFee`의 유효성을 검사합니다.
입력 토큰 수량이 0이면 즉시 종료합니다.

```solidity
    if (zeroForOne) {
        if (params.sqrtPriceLimitX96 >= slot0Start.sqrtPriceX96()) {
            PriceLimitAlreadyExceeded.selector.revertWith(slot0Start.sqrtPriceX96(), params.sqrtPriceLimitX96);
        }
        // Swaps can never occur at MIN_TICK, only at MIN_TICK + 1, except at initialization of a pool
        // Under certain circumstances outlined below, the tick will preemptively reach MIN_TICK without swapping there
        if (params.sqrtPriceLimitX96 <= TickMath.MIN_SQRT_PRICE) {
            PriceLimitOutOfBounds.selector.revertWith(params.sqrtPriceLimitX96);
        }
    } else {
        if (params.sqrtPriceLimitX96 <= slot0Start.sqrtPriceX96()) {
            PriceLimitAlreadyExceeded.selector.revertWith(slot0Start.sqrtPriceX96(), params.sqrtPriceLimitX96);
        }
        if (params.sqrtPriceLimitX96 >= TickMath.MAX_SQRT_PRICE) {
            PriceLimitOutOfBounds.selector.revertWith(params.sqrtPriceLimitX96);
        }
    }
```

`sqrtPriceLimitX96`의 유효성을 검사합니다:

* `zeroForOne`이 `true`라면, 즉 `token0 -> token1` 스왑이라면 스왑 후에는 `token0`( $x$ )는 증가하고 `token1`( $y$ )는 감소하므로 $ \sqrt{P} = \sqrt{\frac{y}{x}} $가 감소합니다. 따라서 목표 가격 `sqrtPriceLimitX96`는 현재 가격보다 작아야 하고 `MIN_SQRT_PRICE`보다 작아질 수는 없습니다.
* 반대로 `zeroForOne`이 `false`라면 목표 가격은 현재 가격보다 커야 하고 `MAX_SQRT_PRICE`보다 클 수는 없습니다.

[Uniswap v3 백서](../../../dive-into-uniswap-v3-whitepaper/README_ko.md#621-price-and-liquidity)에서 설명했듯이 `swap`의 핵심 로직은 다음과 같습니다:

* 어느 시점에서든 유동성 $L$과 가격 $\sqrt{P}$ 중 하나만 변합니다.
* `swap` 중에는:
  * 현재 가격 구간 안의 모든 유동성이 총 유동성 `liquidity`를 이룹니다.
  * 각 `swap`은 여러 개의 `step`으로 나뉘며, 각 `step`은 하나의 `tick` 구간에서만 스왑을 수행합니다. 따라서:
    * `swap` step 안에서는 $L$은 고정되고 $\sqrt{P}$만 변합니다.
    * 가격이 `cross tick`을 통과하면 총 유동성 $L$을 수정하고, $\sqrt{P}$는 그대로 둔 채 다음 `swap step`으로 넘어갑니다.
* 유동성 추가/제거에서는 $L$이 변하고 $\sqrt{P}$는 변하지 않습니다.

아래 `swap` 코드는 이 로직을 구현합니다.

```solidity
    StepComputations memory step;
    step.feeGrowthGlobalX128 = zeroForOne ? self.feeGrowthGlobal0X128 : self.feeGrowthGlobal1X128;
    // continue swapping as long as we haven't used the entire input/output and haven't reached the price limit
    while (!(amountSpecifiedRemaining == 0 || result.sqrtPriceX96 == params.sqrtPriceLimitX96)) {
        step.sqrtPriceStartX96 = result.sqrtPriceX96;

        (step.tickNext, step.initialized) =
            self.tickBitmap.nextInitializedTickWithinOneWord(result.tick, params.tickSpacing, zeroForOne);

        // ensure that we do not overshoot the min/max tick, as the tick bitmap is not aware of these bounds
        if (step.tickNext <= TickMath.MIN_TICK) {
            step.tickNext = TickMath.MIN_TICK;
        }
        if (step.tickNext >= TickMath.MAX_TICK) {
            step.tickNext = TickMath.MAX_TICK;
        }

        // get the price for the next tick
        step.sqrtPriceNextX96 = TickMath.getSqrtPriceAtTick(step.tickNext);

        // compute values to swap to the target tick, price limit, or point where input/output amount is exhausted
        (result.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
            result.sqrtPriceX96,
            SwapMath.getSqrtPriceTarget(zeroForOne, step.sqrtPriceNextX96, params.sqrtPriceLimitX96),
            result.liquidity,
            amountSpecifiedRemaining,
            swapFee
        );

        // if exactOutput
        if (params.amountSpecified > 0) {
            unchecked {
                amountSpecifiedRemaining -= step.amountOut.toInt256();
            }
            amountCalculated -= (step.amountIn + step.feeAmount).toInt256();
        } else {
            // safe because we test that amountSpecified > amountIn + feeAmount in SwapMath
            unchecked {
                amountSpecifiedRemaining += (step.amountIn + step.feeAmount).toInt256();
            }
            amountCalculated += step.amountOut.toInt256();
        }

        // if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
        if (protocolFee > 0) {
            unchecked {
                // step.amountIn does not include the swap fee, as it's already been taken from it,
                // so add it back to get the total amountIn and use that to calculate the amount of fees owed to the protocol
                // cannot overflow due to limits on the size of protocolFee and params.amountSpecified
                // this rounds down to favor LPs over the protocol
                uint256 delta = (swapFee == protocolFee)
                    ? step.feeAmount // lp fee is 0, so the entire fee is owed to the protocol instead
                    : (step.amountIn + step.feeAmount) * protocolFee / ProtocolFeeLibrary.PIPS_DENOMINATOR;
                // subtract it from the total fee and add it to the protocol fee
                step.feeAmount -= delta;
                amountToProtocol += delta;
            }
        }

        // update global fee tracker
        if (result.liquidity > 0) {
            unchecked {
                // FullMath.mulDiv isn't needed as the numerator can't overflow uint256 since tokens have a max supply of type(uint128).max
                step.feeGrowthGlobalX128 +=
                    UnsafeMath.simpleMulDiv(step.feeAmount, FixedPoint128.Q128, result.liquidity);
            }
        }

        // Shift tick if we reached the next price, and preemptively decrement for zeroForOne swaps to tickNext - 1.
        // If the swap doesn't continue (if amountRemaining == 0 or sqrtPriceLimit is met), slot0.tick will be 1 less
        // than getTickAtSqrtPrice(slot0.sqrtPrice). This doesn't affect swaps, but donation calls should verify both
        // price and tick to reward the correct LPs.
        if (result.sqrtPriceX96 == step.sqrtPriceNextX96) {
            // if the tick is initialized, run the tick transition
            if (step.initialized) {
                (uint256 feeGrowthGlobal0X128, uint256 feeGrowthGlobal1X128) = zeroForOne
                    ? (step.feeGrowthGlobalX128, self.feeGrowthGlobal1X128)
                    : (self.feeGrowthGlobal0X128, step.feeGrowthGlobalX128);
                int128 liquidityNet =
                    Pool.crossTick(self, step.tickNext, feeGrowthGlobal0X128, feeGrowthGlobal1X128);
                // if we're moving leftward, we interpret liquidityNet as the opposite sign
                // safe because liquidityNet cannot be type(int128).min
                unchecked {
                    if (zeroForOne) liquidityNet = -liquidityNet;
                }

                result.liquidity = LiquidityMath.addDelta(result.liquidity, liquidityNet);
            }

            unchecked {
                result.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
            }
        } else if (result.sqrtPriceX96 != step.sqrtPriceStartX96) {
            // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
            result.tick = TickMath.getTickAtSqrtPrice(result.sqrtPriceX96);
        }
    }
```

전체 `swap` 작업은 여러 개의 `swap step`으로 나뉩니다.
`while` 루프 안에서는 각 `step`마다 다음 순서로 `swap step`을 수행합니다:

1. `tickBitmap.nextInitializedTickWithinOneWord`를 통해 최대 하나의 초기화된 `tick`을 찾고, 그 `tick`에 대응하는 가격 `sqrtPriceNextX96`를 계산합니다.
2. [SwapMath.computeSwapStep](../../../dive-into-uniswap-v3-contracts/README_ko.md#computeswapstep)를 사용해 현재 `tick` 구간 안에서 스왑을 계산합니다. 이 단계에서는 유동성 $L$은 고정되고 가격 $\sqrt{P}$만 바뀝니다. 결과로 종료 가격 `sqrtPriceX96`, 입력량 `amountIn`, 출력량 `amountOut`, 수수료 `feeAmount`를 얻으며, 이 값들은 모두 음수가 아닙니다.
3. `amountSpecifiedRemaining`과 `amountCalculated`를 갱신합니다.
   * `params.amountSpecified > 0`, 즉 `exactOutput`이면 `amountSpecifiedRemaining`은 출력 토큰을 의미하므로 `amountOut`만큼 차감해야 하고, `amountCalculated`는 입력 토큰을 의미하므로 사용자가 예치해야 하는 `amountIn + feeAmount`를 음수로 기록합니다.
   * 반대로 `exactInput`이면 `amountSpecifiedRemaining`은 입력 토큰을 의미하며 음수 값이므로 `amountIn + feeAmount`만큼 증가시키고, `amountCalculated`는 사용자가 인출할 수 있는 출력 토큰 `amountOut`을 양수로 기록합니다.
4. `protocolFee > 0`이면 현재 `step`의 프로토콜 수수료 `amountToProtocol`을 계산하고, 그만큼 `feeAmount`에서 차감합니다. 계산식은 다음과 같습니다:

    $$
    protocol fee = (amountIn + feeAmount) \cdot \frac{protocolFee}{10^6}
    $$

5. 현재 전역 유동성 `result.liquidity`가 0보다 크면, 이번 `step`에서 모은 수수료(`protocolFee` 차감 후)를 이용해 유동성 단위당 전역 수수료 성장값 `feeGrowthGlobalX128`를 갱신합니다.
6. `swap step` 이후의 가격 `result.sqrtPriceX96`가 다음 `tick`의 가격 `step.sqrtPriceNextX96`와 같다면 현재 `tick` 구간을 모두 소진했다는 뜻입니다. `nextInitializedTickWithinOneWord`는 초기화되지 않은 `tick`도 반환할 수 있으므로, 현재 `tick`이 실제로 초기화되어 있을 때만 [crossTick](#crosstick)을 실행합니다. 그런 다음 `tick`이 보유한 `liquidityNet`을 기준으로 전역 유동성 `result.liquidity`를 갱신합니다.
   * `tick`을 처음 초기화할 때는 기본적으로 `tickLower`의 `liquidityNet`은 양수, `tickUpper`의 `liquidityNet`은 음수로 저장합니다.
   * `zeroForOne`이 `true`, 즉 `token0 -> token1` 스왑이면 스왑이 진행될수록 `token0`는 늘고 `token1`은 줄어들어 $\sqrt{P} = \sqrt{\frac{y}{x}}$가 감소합니다. 즉 `tick`이 오른쪽에서 왼쪽으로 이동합니다.
      * `tickLower`를 지나면 포지션 범위를 벗어나므로 전역 유동성을 줄여야 합니다.
      * `tickUpper`를 지나면 포지션 범위에 진입하므로 전역 유동성을 늘려야 합니다.
      * 따라서 `zeroForOne`이 `true`일 때는 `liquidityNet`의 부호를 뒤집어야 합니다.
   * `zeroForOne`이 `false`, 즉 `token1 -> token0` 스왑이면 `tick`은 왼쪽에서 오른쪽으로 이동합니다.
      * `tickLower`를 지나면 포지션 범위에 진입하므로 전역 유동성을 늘려야 합니다.
      * `tickUpper`를 지나면 포지션 범위를 벗어나므로 전역 유동성을 줄여야 합니다.
      * 따라서 `zeroForOne`이 `false`일 때는 `liquidityNet`의 부호를 뒤집을 필요가 없습니다.
7. `tick`을 다음 값으로 갱신합니다. `zeroForOne`이 `true`이면 `tick`이 감소하므로 다음 `tick`은 `step.tickNext - 1`이고, 그렇지 않으면 `step.tickNext`입니다.
8. `swap step` 이후의 가격 `result.sqrtPriceX96`가 다음 `tick`의 가격 `step.sqrtPriceNextX96`에 도달하지 못했다면 `amountRemaining`이 0이 되었거나 가격이 `sqrtPriceLimit`에 도달한 것입니다. 현재 가격이 `sqrtPriceStartX96`와 다르면 `tick`을 다시 계산합니다.

```solidity
    self.slot0 = slot0Start.setTick(result.tick).setSqrtPriceX96(result.sqrtPriceX96);

    // update liquidity if it changed
    if (self.liquidity != result.liquidity) self.liquidity = result.liquidity;

    // update fee growth global
    if (!zeroForOne) {
        self.feeGrowthGlobal1X128 = step.feeGrowthGlobalX128;
    } else {
        self.feeGrowthGlobal0X128 = step.feeGrowthGlobalX128;
    }
```

여기까지 오면 `swap`이 종료된 상태입니다. 즉 `amountRemaining`이 0이 되었거나 가격이 `sqrtPriceLimit`에 도달했습니다. 이 시점에서 `slot0`, 전역 유동성 `liquidity`, 전역 수수료 성장값 `feeGrowthGlobalX128`를 갱신합니다.

```solidity
    unchecked {
        // "if currency1 is specified"
        if (zeroForOne != (params.amountSpecified < 0)) {
            swapDelta = toBalanceDelta(
                amountCalculated.toInt128(), (params.amountSpecified - amountSpecifiedRemaining).toInt128()
            );
        } else {
            swapDelta = toBalanceDelta(
                (params.amountSpecified - amountSpecifiedRemaining).toInt128(), amountCalculated.toInt128()
            );
        }
    }
}
```

`params.amountSpecified < 0`은 `exactInput`을 뜻합니다. 이를 `zeroForOne`과 조합하면 다음 네 가지 경우가 생깁니다:

| zeroForOne | exactInput | 설명 | amountSpecified | amountCalculated |
| --- | --- | --- | --- | --- |
| true | true | `token0 -> token1` 스왑, 입력 토큰 수량은 알고 출력 토큰 수량은 모름 | 입력 `token0` 수량, 음수 | 출력 `token1` 수량 |
| true | false | `token0 -> token1` 스왑, 출력 토큰 수량은 알고 입력 토큰 수량은 모름 | 출력 `token1` 수량, 양수 | 입력 `token0` 수량 |
| false | true | `token1 -> token0` 스왑, 입력 토큰 수량은 알고 출력 토큰 수량은 모름 | 입력 `token1` 수량, 음수 | 출력 `token0` 수량 |
| false | false | `token1 -> token0` 스왑, 출력 토큰 수량은 알고 입력 토큰 수량은 모름 | 출력 `token0` 수량, 양수 | 입력 `token1` 수량 |

따라서 `zeroForOne != (params.amountSpecified < 0)`일 때 `amountSpecified`는 항상 `token1`의 수량을 의미하고, `params.amountSpecified - amountSpecifiedRemaining`은 실제로 필요한 `token1`의 수량입니다. `amountCalculated`는 항상 `token0`의 수량을 의미합니다.

최종적으로 계산된 `swapDelta`는 이번 `swap` 작업의 토큰 잔액 변화입니다. 앞 128비트는 `token0`의 잔액 변화, 뒤 128비트는 `token1`의 잔액 변화를 나타냅니다. 음수면 사용자가 예치해야 하는 수량, 양수면 사용자가 인출할 수 있는 수량입니다.

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
