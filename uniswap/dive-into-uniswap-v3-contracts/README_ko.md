[English](./README.md) | [中文](./README_zh.md)

# Uniswap v3 스마트 컨트랙트 심층 분석 (1부)

###### 태그: `uniswap` `uniswap-v3` `smart contract` `solidity`

## 개요 (Overview)

Uniswap v2와 유사하게 Uniswap v3 컨트랙트는 두 가지 범주로 나뉩니다:

* [Uniswap-v3-core](#Uniswap-v3-core)
    - Uniswap v3의 핵심 코드로, 프로토콜에 정의된 모든 기능을 구현하며 외부 컨트랙트가 핵심 컨트랙트와 직접 상호 작용할 수 있습니다.
* [Uniswap-v3-periphery](../dive-into-uniswap-v3-contracts-2/README.md)
    - 포지션 관리, 다중 경로 토큰 스왑 등과 같은 사용 시나리오를 기반으로 캡슐화된 인터페이스입니다. Uniswap 인터페이스는 주변(periphery) 컨트랙트와 상호 작용합니다.

사용자 시나리오 관점에서 읽고 싶다면 포지션 생성, 포지션 유동성 수정, 토큰 스왑 등과 같은 일반적인 기능이 포함된 [Uniswap-v3-periphery](../dive-into-uniswap-v3-contracts-2/README.md)부터 시작하세요.

바닥부터 핵심 모듈로 시작하고 싶다면 [Uniswap-v3-core](#Uniswap-v3-core)부터 시작하세요.

## Uniswap-v3-core

### UniswapV3Factory.sol

팩토리 컨트랙트는 주로 세 가지 기능을 포함합니다:

* [createPool](#createPool): 거래 쌍 풀을 생성합니다.
* [setOwner](#setOwner): 팩토리 컨트랙트의 소유자를 설정합니다.
* [enableFeeAmount](#enableFeeAmount): 수수료 등급을 추가합니다.

#### createPool

Uniswap v3 거래 쌍 풀을 생성합니다. Uniswap v3는 0.05%, 0.30%, 1.00% 등과 같은 다양한 수수료 등급을 지원하므로, 거래 쌍 컨트랙트는 `tokenA`, `tokenB`, `fee`에 의해 고유하게 식별됩니다.

> 거래 쌍 컨트랙트를 계산하려면 팩토리 컨트랙트 주소와 컨트랙트 초기화 코드의 해시도 필요합니다.

```solidity
/// @inheritdoc IUniswapV3Factory
function createPool(
    address tokenA,
    address tokenB,
    uint24 fee
) external override noDelegateCall returns (address pool) {
    require(tokenA != tokenB);
    (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
    require(token0 != address(0));
    int24 tickSpacing = feeAmountTickSpacing[fee];
    require(tickSpacing != 0);
    require(getPool[token0][token1][fee] == address(0));
    pool = deploy(address(this), token0, token1, fee, tickSpacing);
    getPool[token0][token1][fee] = pool;
    // populate mapping in the reverse direction, deliberate choice to avoid the cost of comparing addresses
    getPool[token1][token0][fee] = pool;
    emit PoolCreated(token0, token1, fee, tickSpacing, pool);
}
```

`tokenA`와 `tokenB`는 순서가 없으므로 `tokenA < tokenB`를 보장하기 위해 먼저 `tokenA`, `tokenB`를 정렬합니다.

수수료 등급에 따라 그에 상응하는 `tickSpacing`을 얻습니다:

```solidity
int24 tickSpacing = feeAmountTickSpacing[fee];
```

"Uniswap v3 백서 심층 분석"에서 소개했듯이 `tickSpacing`은 중요한 역할을 합니다. 각 수수료 등급은 `tickSpacing`에 해당합니다. `tickSpacing`으로 나누어떨어지는 틱만 초기화할 수 있으며, `tickSpacing`이 클수록 틱당 유동성이 많아지고 틱 간의 슬리피지가 커지지만, 틱을 교차하는 작업에 대한 가스를 절약할 수 있습니다. 여기서 이는 `Pool`의 매개변수로 저장됩니다.

해당 수수료 등급의 거래 쌍이 생성되지 않았는지 확인합니다:

```solidity
require(getPool[token0][token1][fee] == address(0));
```

거래 쌍 컨트랙트를 생성(배포)합니다:

```solidity
pool = deploy(address(this), token0, token1, fee, tickSpacing);
```

배포 코드는 다음과 같습니다:

```solidity
/// @dev Deploys a pool with the given parameters by transiently setting the parameters storage slot and then
/// clearing it after deploying the pool.
/// @param factory The contract address of the Uniswap V3 factory
/// @param token0 The first token of the pool by address sort order
/// @param token1 The second token of the pool by address sort order
/// @param fee The fee collected upon every swap in the pool, denominated in hundredths of a bip
/// @param tickSpacing The spacing between usable ticks
function deploy(
    address factory,
    address token0,
    address token1,
    uint24 fee,
    int24 tickSpacing
) internal returns (address pool) {
    parameters = Parameters({factory: factory, token0: token0, token1: token1, fee: fee, tickSpacing: tickSpacing});
    pool = address(new UniswapV3Pool{salt: keccak256(abi.encode(token0, token1, fee))}());
    delete parameters;
}
```

Uniswap v2에서 언급했듯이 거래 쌍 컨트랙트 주소의 계산 가능성과 고유성을 보장하기 위해 Uniswap v2는 `CREATE2` 옵코드를 사용하여 거래 쌍 컨트랙트를 생성합니다. Solidity 0.6.2 버전([Github PR](https://github.com/ethereum/solidity/pull/8177))부터는 `new` 메서드에 `salt` 매개변수를 전달하여 `CREATE2` 기능을 구현합니다. `salt` 매개변수는 컨트랙트 주소의 고유성과 계산 가능성을 보장합니다. 코드에서 Uniswap v3 거래 쌍 컨트랙트는 token0, token1, fee를 사용하여 거래 쌍 컨트랙트를 고유하게 결정함을 알 수 있습니다. 예를 들어 ETH-USDC 0.05% 수수료(및 팩토리 컨트랙트 주소, 초기화 코드 해시) 등을 기반으로 거래 쌍 컨트랙트 주소를 계산할 수 있습니다.

마지막으로 거래 쌍 컨트랙트 주소를 `getPool` 변수에 저장합니다:

```solidity
getPool[token0][token1][fee] = pool;
// populate mapping in the reverse direction, deliberate choice to avoid the cost of comparing addresses
getPool[token1][token0][fee] = pool;
```

#### setOwner

팩토리 컨트랙트의 소유자를 설정합니다. 소유자는 다음과 같은 권한을 가집니다:

* [setOwner](#setOwner): 소유자를 수정합니다.
* [enableFeeAmount](#enableFeeAmount): 수수료 등급을 추가합니다.
* [setFeeProtocol](#setFeeProtocol): 특정 거래 쌍에 대한 프로토콜 수수료 비율을 수정합니다.
* [collectProtocol](#collectProtocol): 특정 거래 쌍에 대한 프로토콜 수수료를 징수합니다.

먼저 요청이 현재 소유자에 의해 시작되었는지 확인한 다음 소유자를 수정합니다:

```solidity
/// @inheritdoc IUniswapV3Factory
function setOwner(address _owner) external override {
    require(msg.sender == owner);
    emit OwnerChanged(owner, _owner);
    owner = _owner;
}
```

#### enableFeeAmount

Uniswap v3는 0.05%, 0.30%, 1.00%의 세 가지 기본 수수료 등급을 지원하며, 이는 각각 500, 3000, 10000의 수수료에 해당합니다. 수수료의 기본 단위는 베이시스 포인트의 100분의 1, 즉 0.01 bp = $10^{-6}$입니다.

수수료 백분율 계산 공식:

$$
f_{ratio} = \frac{fee}{1,000,000}
$$

```solidity
/// @inheritdoc IUniswapV3Factory
function enableFeeAmount(uint24 fee, int24 tickSpacing) public override {
    require(msg.sender == owner);
    require(fee < 1000000);
    // tick spacing is capped at 16384 to prevent the situation where tickSpacing is so large that
    // TickBitmap#nextInitializedTickWithinOneWord overflows int24 container from a valid tick
    // 16384 ticks represents a >5x price change with ticks of 1 bips
    require(tickSpacing > 0 && tickSpacing < 16384);
    require(feeAmountTickSpacing[fee] == 0);

    feeAmountTickSpacing[fee] = tickSpacing;
    emit FeeAmountEnabled(fee, tickSpacing);
}
```

### UniswapV3Pool.sol

이는 Uniswap v3의 메인 코드로, 거래 쌍 풀의 기능을 정의합니다:

* [initialize](#initialize): 거래 쌍을 초기화합니다.
* [mint](#mint): 유동성을 추가합니다.
* [burn](#burn): 유동성을 제거합니다.
* [swap](#swap): 토큰을 스왑합니다.
* [flash](#flash): 플래시 론을 실행합니다.
* [collect](#collect): 토큰을 출금합니다.
* [increaseObservationCardinalityNext](#increaseObservationCardinalityNext): 오라클을 위한 공간을 확장합니다.
* [observe](#observe): 오라클 데이터를 얻습니다.

또한 팩토리 소유자는 다음 두 가지 메서드를 호출할 수 있습니다:

* [setFeeProtocol](#setFeeProtocol): 특정 거래 쌍에 대한 프로토콜 수수료 비율을 수정합니다.
* [collectProtocol](#collectProtocol): 특정 거래 쌍에 대한 프로토콜 수수료를 징수합니다.

![](./assets/uniswapV3Pool.png)

#### initialize

거래 쌍을 생성한 후 컨트랙트를 정상적으로 사용하려면 `initialize` 메서드를 호출하여 초기화해야 합니다.

이 메서드는 `slot0` 변수를 초기화합니다:

```solidity
/// @inheritdoc IUniswapV3PoolActions
/// @dev not locked because it initializes unlocked
function initialize(uint160 sqrtPriceX96) external override {
    require(slot0.sqrtPriceX96 == 0, 'AI');

    int24 tick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);

    (uint16 cardinality, uint16 cardinalityNext) = observations.initialize(_blockTimestamp());

    slot0 = Slot0({
        sqrtPriceX96: sqrtPriceX96,
        tick: tick,
        observationIndex: 0,
        observationCardinality: cardinality,
        observationCardinalityNext: cardinalityNext,
        feeProtocol: 0,
        unlocked: true
    });

    emit Initialize(sqrtPriceX96, tick);
}
```

`slot0`는 다음과 같이 정의됩니다:

* `sqrtPriceX96`: 거래 쌍의 현재 제곱근 가격 $\sqrt{P}$.
* `tick`: $\sqrt{P}$에 해당하는 현재 틱, [getTickAtSqrtRatio](#gettickatsqrtratio)를 사용하여 계산됨.
* `observationIndex`: 가장 최근에 업데이트된 (오라클) 관측 배열 인덱스.
* `observationCardinality`: (오라클) 관측 배열의 용량, 최대 65536, 초기에 1로 설정됨.
* `observationCardinalityNext`: 다음 (오라클) 관측 배열 용량, 수동으로 확장된 경우 이 값이 업데이트됨, 초기에 1로 설정됨.
* `feeProtocol`: 프로토콜 수수료 비율, 프로토콜에 제공되는 `token0`과 `token1`의 거래 수수료 비율을 별도로 설정할 수 있음.
* `unlocked`: 현재 거래 쌍 컨트랙트가 잠금 해제되었는지 여부를 나타냄.

#### mint

이 메서드는 유동성 추가 기능을 구현합니다. 사실 유동성을 처음 추가할 때와 이후 유동성을 늘릴 때 모두 이 메서드를 사용합니다.

`mint` 메서드의 매개변수:

* `recipient`: 포지션의 수령인(소유자).
* `tickLower`: 유동성 범위의 하한.
* `tickUpper`: 유동성 범위의 상한.
* `amount`: 유동성의 양.
* `data`: 콜백 매개변수.

```solidity
/// @inheritdoc IUniswapV3PoolActions
/// @dev noDelegateCall is applied indirectly via _modifyPosition
function mint(
    address recipient,
    int24 tickLower,
    int24 tickUpper,
    uint128 amount,
    bytes calldata data
) external override lock returns (uint256 amount0, uint256 amount1) {
    require(amount > 0);
    (, int256 amount0Int, int256 amount1Int) =
        _modifyPosition(
            ModifyPositionParams({
                owner: recipient,
                tickLower: tickLower,
                tickUpper: tickUpper,
                liquidityDelta: int256(amount).toInt128()
            })
        );

    amount0 = uint256(amount0Int);
    amount1 = uint256(amount1Int);

    uint256 balance0Before;
    uint256 balance1Before;
    if (amount0 > 0) balance0Before = balance0();
    if (amount1 > 0) balance1Before = balance1();
    IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(amount0, amount1, data);
    if (amount0 > 0) require(balance0Before.add(amount0) <= balance0(), 'M0');
    if (amount1 > 0) require(balance1Before.add(amount1) <= balance1(), 'M1');

    emit Mint(msg.sender, recipient, tickLower, tickUpper, amount, amount0, amount1);
}
```

`mint`의 주요 로직은 [_modifyPosition](#_modifyPosition)에 있으며, 이는 `amount` 유동성을 추가할 경우 거래 쌍 컨트랙트로 전송해야 하는 `token0` 및 `token1` 토큰의 양을 나타내는 `amount0Int` 및 `amount1Int`를 반환합니다.

호출자는 `uniswapV3MintCallback`에서 토큰을 전송해야 합니다. `mint`를 호출하는 컨트랙트는 `IUniswapV3MintCallback` 인터페이스를 구현해야 하며, 이는 Uniswap v3 주변(periphery) 컨트랙트의 `NonfungiblePositionManager.sol`에 구현되어 있습니다.

> `mint` 호출자가 인터페이스 메서드를 구현해야 하므로, 개인 ETH 계정(EOA)은 이 메서드를 호출할 수 없습니다.

```solidity
IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(amount0, amount1, data);
```

##### _modifyPosition

`_modifyPosition`을 살펴보겠습니다:

```solidity
/// @dev Effect some changes to a position
/// @param params the position details and the change to the position's liquidity to effect
/// @return position a storage pointer referencing the position with the given owner and tick range
/// @return amount0 the amount of token0 owed to the pool, negative if the pool should pay the recipient
/// @return amount1 the amount of token1 owed to the pool, negative if the pool should pay the recipient
function _modifyPosition(ModifyPositionParams memory params)
    private
    noDelegateCall
    returns (
        Position.Info storage position,
        int256 amount0,
        int256 amount1
    )
{
    checkTicks(params.tickLower, params.tickUpper);

    Slot0 memory _slot0 = slot0; // SLOAD for gas optimization

    position = _updatePosition(
        params.owner,
        params.tickLower,
        params.tickUpper,
        params.liquidityDelta,
        _slot0.tick
    );
```

먼저 [_updatePosition](#_updatePosition)을 통해 포지션 정보를 업데이트하며, 이에 대해서는 다음 섹션에서 자세히 설명합니다.

```solidity
    if (params.liquidityDelta != 0) {
        if (_slot0.tick < params.tickLower) {
            // current tick is below the passed range; liquidity can only become in range by crossing from left to
            // right, when we'll need _more_ token0 (it's becoming more valuable) so user must provide it
            amount0 = SqrtPriceMath.getAmount0Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );
        } else if (_slot0.tick < params.tickUpper) {
            // current tick is inside the passed range
            uint128 liquidityBefore = liquidity; // SLOAD for gas optimization

            // write an oracle entry
            (slot0.observationIndex, slot0.observationCardinality) = observations.write(
                _slot0.observationIndex,
                _blockTimestamp(),
                _slot0.tick,
                liquidityBefore,
                _slot0.observationCardinality,
                _slot0.observationCardinalityNext
            );

            amount0 = SqrtPriceMath.getAmount0Delta(
                _slot0.sqrtPriceX96,
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );
            amount1 = SqrtPriceMath.getAmount1Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                _slot0.sqrtPriceX96,
                params.liquidityDelta
            );

            liquidity = LiquidityMath.addDelta(liquidityBefore, params.liquidityDelta);
        } else {
            // current tick is above the passed range; liquidity can only become in range by crossing from right to
            // left, when we'll need _more_ token1 (it's becoming more valuable) so user must provide it
            amount1 = SqrtPriceMath.getAmount1Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );
        }
    }
}
```

코드의 후반부는 주로 [getAmount0Delta](#getAmount0Delta) 및 [getAmount1Delta](#getAmount1Delta)를 통해 이 유동성에 필요한 `token0` 및 `token1`의 양인 `amount0` 및 `amount1`을 계산합니다.

구체적으로 제공하는 유동성 범위가 현재 `tick` $i_c$보다 큰 경우, `tick`의 크기는 $\sqrt{P}$ (즉, $\sqrt{\frac{y}{x}}$)에 비례하므로 $i_c$보다 큰 범위에서는 $x$가 더 가치 있으므로(더 적은 $x$ 필요) $x$ 토큰, 즉 `amount0`만큼의 `token0`를 제공해야 합니다. 그렇지 않으면 $y$ 토큰, 즉 `amount1`만큼의 `token1`을 제공합니다.

다음과 같이 표시됩니다:

$$
\begin{cases}i_c, ..., \overbrace{i_l, ..., i_u}^{amount0} & \text{if $i_c < i_l$}\\
\overbrace{i_l, ...}^{amount1}, i_c, \overbrace{..., i_u}^{amount0} & \text{if $i_l \leq i_c < i_u$}\\
\overbrace{i_l, ..., i_u}^{amount1}, ..., i_c & \text{if $i_u \leq i_c$}\end{cases}
$$

여기서 $i_l$, $i_u$는 유동성 가격 범위의 경계이고 $i_c$는 현재 가격에 해당하는 `tick`입니다.

현재 가격이 범위 내에 있는 경우, 즉 $i_l \leq i_c < i_u$인 경우, `_modifyPosition`은 (오라클) 관측 점 데이터를 기록합니다. 범위 내 유동성이 변경되었고 각 유동성의 지속 시간 `secondsPerLiquidityCumulativeX128`을 기록해야 하기 때문입니다:

```solidity
// write an oracle entry
(slot0.observationIndex, slot0.observationCardinality) = observations.write(
    _slot0.observationIndex,
    _blockTimestamp(),
    _slot0.tick,
    liquidityBefore,
    _slot0.observationCardinality,
    _slot0.observationCardinalityNext
);
```

`amount0` 및 `amount1`을 계산한 후 현재 거래 쌍의 전역 활성 유동성 `liquidity`를 업데이트합니다:

```solidity
liquidity = LiquidityMath.addDelta(liquidityBefore, params.liquidityDelta);
```

이 전역 유동성은 [swap](#swap)에서 사용됩니다.

##### _updatePosition

`_modifyPosition` 내의 `_updatePosition` 코드는 다음과 같습니다:

```solidity
/// @dev Gets and updates a position with the given liquidity delta
/// @param owner the owner of the position
/// @param tickLower the lower tick of the position's tick range
/// @param tickUpper the upper tick of the position's tick range
/// @param tick the current tick, passed to avoid sloads
function _updatePosition(
    address owner,
    int24 tickLower,
    int24 tickUpper,
    int128 liquidityDelta,
    int24 tick
) private returns (Position.Info storage position) {
    position = positions.get(owner, tickLower, tickUpper);

    uint256 _feeGrowthGlobal0X128 = feeGrowthGlobal0X128; // SLOAD for gas optimization
    uint256 _feeGrowthGlobal1X128 = feeGrowthGlobal1X128; // SLOAD for gas optimization
    // if we need to update the ticks, do it
    bool flippedLower;
    bool flippedUpper;
    if (liquidityDelta != 0) {
        uint32 time = _blockTimestamp();
        (int56 tickCumulative, uint160 secondsPerLiquidityCumulativeX128) =
            observations.observeSingle(
                time,
                0,
                slot0.tick,
                slot0.observationIndex,
                liquidity,
                slot0.observationCardinality
            );
```

`observations.observeSingle`은 마지막 관측 시점부터 현재까지의 누적 틱 `tickCumulative`과 유동성 단위당 누적 지속 시간 `secondsPerLiquidityCumulativeX128`을 계산합니다.

```solidity
        flippedLower = ticks.update(
            tickLower,
            tick,
            liquidityDelta,
            _feeGrowthGlobal0X128,
            _feeGrowthGlobal1X128,
            secondsPerLiquidityCumulativeX128,
            tickCumulative,
            time,
            false,
            maxLiquidityPerTick
        );
        flippedUpper = ticks.update(
            tickUpper,
            tick,
            liquidityDelta,
            _feeGrowthGlobal0X128,
            _feeGrowthGlobal1X128,
            secondsPerLiquidityCumulativeX128,
            tickCumulative,
            time,
            true,
            maxLiquidityPerTick
        );
        if (flippedLower) {
            tickBitmap.flipTick(tickLower, tickSpacing);
        }
        if (flippedUpper) {
            tickBitmap.flipTick(tickUpper, tickSpacing);
        }
    }
```

그런 다음 `ticks.update`를 사용하여 `tickLower`(가격 범위의 하한) 및 `tickUpper`(가격 범위의 상한)의 상태를 각각 업데이트합니다. [Tick.update](#update)를 참조하세요.

해당 `tick`의 유동성이 0에서 0이 아닌 값으로 또는 그 반대로 변경되면 `tick`을 뒤집어야(flip) 함을 나타냅니다. 해당 `tick`이 초기화된 것으로 표시되지 않은 경우 초기화된 것으로 표시하고, 그렇지 않으면 표시를 해제합니다. 여기서는 `tickBitmap.flipTick` 메서드가 사용됩니다. [TickBitmap.flipTick](#flipTick)을 참조하세요.

```solidity
    (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) =
        ticks.getFeeGrowthInside(tickLower, tickUpper, tick, _feeGrowthGlobal0X128, _feeGrowthGlobal1X128);
```

그런 다음 해당 가격 범위에 대한 유동성당 누적 수수료를 계산합니다.

```solidity
    position.update(liquidityDelta, feeGrowthInside0X128, feeGrowthInside1X128);
```

포지션 정보를 업데이트합니다. 주로 포지션의 수취 가능 수수료 `tokensOwed0` 및 `tokensOwed1`과 포지션 유동성 `liquidity`를 업데이트합니다. [Position.update](#update1)를 참조하세요.

```solidity
    // clear any tick data that is no longer needed
    if (liquidityDelta < 0) {
        if (flippedLower) {
            ticks.clear(tickLower);
        }
        if (flippedUpper) {
            ticks.clear(tickUpper);
        }
    }
}
```

유동성을 제거하는 중이고 `tick`이 뒤집힌 경우 [clear](#Tick.clear)를 호출하여 `tick` 상태를 지웁니다.

마지막으로 `mint` 메서드로 돌아가서, 호출자는 여기서 계산된 `amount0` 및 `amount1` 만큼의 `token0` 및 `token1` 토큰이 `uniswapV3MintCallback`에서 거래 쌍 컨트랙트로 전송되도록 보장해야 합니다.

요약하면 `mint` 메서드의 주요 작업은 다음과 같습니다:

1. 가격 범위의 끝점(하한, 상한)에 대한 정보 업데이트: `ticks.update`.
2. 틱 상태가 뒤집힌 경우 비트맵 표시기를 "초기화됨" 또는 "초기화되지 않음"으로 업데이트: `tickBitmap.flipTick`.
3. 포지션 정보 업데이트: `positions.update`.
4. 현재 틱이 가격 범위 내에 있는 경우:
    - 오라클 관측 점 쓰기: `observations.write`.
    - 전역 활성 유동성 업데이트: `liquidity`.
5. 호출자가 거래 쌍 컨트랙트로 전송: `uniswapV3MintCallback`.

#### burn

유동성 소각(`burn`) 로직은 유동성 추가(`mint`)와 거의 동일하며, 유일한 차이점은 `liquidityDelta`가 음수라는 것입니다.

```solidity
/// @inheritdoc IUniswapV3PoolActions
/// @dev noDelegateCall is applied indirectly via _modifyPosition
function burn(
    int24 tickLower,
    int24 tickUpper,
    uint128 amount
) external override lock returns (uint256 amount0, uint256 amount1) {
    (Position.Info storage position, int256 amount0Int, int256 amount1Int) =
        _modifyPosition(
            ModifyPositionParams({
                owner: msg.sender,
                tickLower: tickLower,
                tickUpper: tickUpper,
                liquidityDelta: -int256(amount).toInt128()
            })
        );

    amount0 = uint256(-amount0Int);
    amount1 = uint256(-amount1Int);

    if (amount0 > 0 || amount1 > 0) {
        (position.tokensOwed0, position.tokensOwed1) = (
            position.tokensOwed0 + uint128(amount0),
            position.tokensOwed1 + uint128(amount1)
        );
    }

    emit Burn(msg.sender, tickLower, tickUpper, amount, amount0, amount1);
}
```

여기서는 `mint`와 동일한 [_modifyPosition](#_modifyPosition) 메서드가 사용됩니다.

유동성을 (일부) 소각한 후 토큰은 호출자에게 다시 전송되지 않고 포지션(Position)에 미청구 토큰으로 기록된다는 점에 유의하세요.

#### swap

`swap` 메서드는 Uniswap v3 코드의 핵심으로, `token0`에서 `token1`으로 또는 그 반대로 두 토큰의 교환을 구현합니다.

Uniswap v2의 동질적인 유동성과 비교하여, 우리는 `swap` 과정에서 가격이 어떻게 변하고 유동성에 어떤 영향을 미치는지에 초점을 맞춥니다.

먼저 `swap` 메서드의 매개변수를 살펴보겠습니다:

* `recipient`: 거래 후 토큰의 수령인.
* `zeroForOne`: true이면 `token0`에서 `token1`으로, 그렇지 않으면 `token1`에서 `token0`로 스왑.
* `amountSpecified`: 지정된 토큰의 양. 양수이면 입력될 토큰의 양, 음수이면 출력될 토큰의 양.
* `sqrtPriceLimitX96`: 허용 가능한 최대 가격(또는 최소 가격), `Q64.96` 형식; `token0`에서 `token1`으로 스왑하는 경우 스왑 중 가격의 하한을 나타내고, `token1`에서 `token0`로 스왑하는 경우 가격의 상한을 나타냄. 가격이 이 값을 초과하면 스왑이 실패함.
* `data`: 콜백 매개변수.

```solidity
/// @inheritdoc IUniswapV3PoolActions
function swap(
    address recipient,
    bool zeroForOne,
    int256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    bytes calldata data
) external override noDelegateCall returns (int256 amount0, int256 amount1) {
    require(amountSpecified != 0, 'AS');

    Slot0 memory slot0Start = slot0;

    require(slot0Start.unlocked, 'LOK');
    require(
        zeroForOne
            ? sqrtPriceLimitX96 < slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 > TickMath.MIN_SQRT_RATIO
            : sqrtPriceLimitX96 > slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 < TickMath.MAX_SQRT_RATIO,
        'SPL'
    );

    slot0.unlocked = false;

    SwapCache memory cache =
        SwapCache({
            liquidityStart: liquidity,
            blockTimestamp: _blockTimestamp(),
            feeProtocol: zeroForOne ? (slot0Start.feeProtocol % 16) : (slot0Start.feeProtocol >> 4),
            secondsPerLiquidityCumulativeX128: 0,
            tickCumulative: 0,
            computedLatestObservation: false
        });

    bool exactInput = amountSpecified > 0;

    SwapState memory state =
        SwapState({
            amountSpecifiedRemaining: amountSpecified,
            amountCalculated: 0,
            sqrtPriceX96: slot0Start.sqrtPriceX96,
            tick: slot0Start.tick,
            feeGrowthGlobalX128: zeroForOne ? feeGrowthGlobal0X128 : feeGrowthGlobal1X128,
            protocolFee: 0,
            liquidity: cache.liquidityStart
        });
```

위의 코드는 주로 상태를 초기화합니다.

$\sqrt{P} = \sqrt{\frac{y}{x}}$이므로 `zeroForOne = true`일 때, 즉 `token0`에서 `token1`으로 스왑할 때, 스왑 과정에서 풀의 $x$는 증가하고 $y$는 감소하므로 $\sqrt{P}$는 점차 감소합니다. 따라서 지정된 가격 제한 `sqrtPriceLimitX96`은 현재 시장 가격 `sqrtPriceX96`보다 작아야 합니다.

또한 몇 가지 주요 데이터에 유의해야 합니다:

* 초기 거래 가격 `state.sqrtPriceX96`은 `slot0.sqrtPriceX96`입니다.
    - 여기서 `slot0.tick`은 초기 가격을 계산하는 데 사용되지 않습니다. `slot0.tick`에서 계산된 값이 `slot0.sqrtPriceX96`과 일치하지 않을 수 있기 때문입니다. 나중에 코드에서 `slot0.tick`이 현재 가격으로 사용될 수 없음을 보게 될 것입니다.
* 초기 사용 가능 유동성 `state.liquidity`는 `liquidity`이며, 이는 [mint](#mint) 또는 [burn](#burn)에서 업데이트하는 전역 사용 가능 유동성입니다.

`zeroForOne` 및 `exactInput`에 따라 `swap`의 네 가지 조합이 있을 수 있습니다:

|zeroForOne|exactInput|swap|
|---|---|---|
|true|true|고정된 양의 `token0` 입력, 최대 양의 `token1` 출력|
|true|false|최소 양의 `token0` 입력, 고정된 양의 `token1` 출력|
|false|true|고정된 양의 `token1` 입력, 최대 양의 `token0` 출력|
|false|false|최소 양의 `token1` 입력, 고정된 양의 `token0` 출력|

완전한 `swap`은 여러 `step`으로 구성될 수 있으며, 코드는 다음과 같습니다:

```solidity
// continue swapping as long as we haven't used the entire input/output and haven't reached the price limit
while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != sqrtPriceLimitX96) {
    StepComputations memory step;

    step.sqrtPriceStartX96 = state.sqrtPriceX96;

    (step.tickNext, step.initialized) = tickBitmap.nextInitializedTickWithinOneWord(
        state.tick,
        tickSpacing,
        zeroForOne
    );

    // ensure that we do not overshoot the min/max tick, as the tick bitmap is not aware of these bounds
    if (step.tickNext < TickMath.MIN_TICK) {
        step.tickNext = TickMath.MIN_TICK;
    } else if (step.tickNext > TickMath.MAX_TICK) {
        step.tickNext = TickMath.MAX_TICK;
    }

    // get the price for the next tick
    step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.tickNext);

    // compute values to swap to the target tick, price limit, or point where input/output amount is exhausted
    (state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
        state.sqrtPriceX96,
        (zeroForOne ? step.sqrtPriceNextX96 < sqrtPriceLimitX96 : step.sqrtPriceNextX96 > sqrtPriceLimitX96)
            ? sqrtPriceLimitX96
            : step.sqrtPriceNextX96,
        state.liquidity,
        state.amountSpecifiedRemaining,
        fee
    );

    if (exactInput) {
        state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
        state.amountCalculated = state.amountCalculated.sub(step.amountOut.toInt256());
    } else {
        state.amountSpecifiedRemaining += step.amountOut.toInt256();
        state.amountCalculated = state.amountCalculated.add((step.amountIn + step.feeAmount).toInt256());
    }

    // if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
    if (cache.feeProtocol > 0) {
        uint256 delta = step.feeAmount / cache.feeProtocol;
        step.feeAmount -= delta;
        state.protocolFee += uint128(delta);
    }

    // update global fee tracker
    if (state.liquidity > 0)
        state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);

    // shift tick if we reached the next price
    if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
        // if the tick is initialized, run the tick transition
        if (step.initialized) {
            // check for the placeholder value, which we replace with the actual value the first time the swap
            // crosses an initialized tick
            if (!cache.computedLatestObservation) {
                (cache.tickCumulative, cache.secondsPerLiquidityCumulativeX128) = observations.observeSingle(
                    cache.blockTimestamp,
                    0,
                    slot0Start.tick,
                    slot0Start.observationIndex,
                    cache.liquidityStart,
                    slot0Start.observationCardinality
                );
                cache.computedLatestObservation = true;
            }
            int128 liquidityNet =
                ticks.cross(
                    step.tickNext,
                    (zeroForOne ? state.feeGrowthGlobalX128 : feeGrowthGlobal0X128),
                    (zeroForOne ? feeGrowthGlobal1X128 : state.feeGrowthGlobalX128),
                    cache.secondsPerLiquidityCumulativeX128,
                    cache.tickCumulative,
                    cache.blockTimestamp
                );
            // if we're moving leftward, we interpret liquidityNet as the opposite sign
            // safe because liquidityNet cannot be type(int128).min
            if (zeroForOne) liquidityNet = -liquidityNet;

            state.liquidity = LiquidityMath.addDelta(state.liquidity, liquidityNet);
        }

        state.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
    } else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
        // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
        state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
    }
}
```

의사 코드(pseudo code)로 번역하면:

```python
loop if 남은 토큰 != 0 and 현재 가격 != 최저 (또는 최고) 가격:
    // step
    초기 가격 := 이전 단계의 가격
    다음 틱 := 현재 틱을 기준으로 가장 가까운 초기화된 틱, 또는 그룹의 마지막 미초기화 틱 찾기
    목표 가격 := 다음 틱을 기준으로 계산된 가격
    스왑 후 가격, 사용된 입력 토큰 양, 받은 출력 토큰 양, 거래 수수료 := 스왑 단계 하나 수행(초기 가격, 목표 가격, 사용 가능 유동성, 남은 토큰)

    남은 토큰 업데이트
    프로토콜 수수료 업데이트

    if 스왑 후 가격 == 목표 가격:
        if 틱이 초기화됨：
            틱 교차, 틱 관련 필드 업데이트
            틱 순 유동성으로 사용 가능 유동성 업데이트

        현재 틱 := 다음 틱 - 1
    else if 스왑 후 가격 != 초기 가격:
        현재 틱 := 스왑 후 가격을 기준으로 틱 계산
```

먼저 현재 `tick`을 기준으로 다음 `tick`, 즉 `tickNext`를 찾습니다. 구체적인 로직은 [tickBitmap.nextInitializedTickWithinOneWord](#nextInitializedTickWithinOneWord)를 참조하세요.

이 단계의 목표 가격을 계산합니다:

```solidity
// get the price for the next tick
step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.tickNext);
```

이 단계의 입력, 출력 및 수수료를 계산합니다(즉, 하나의 스왑 실행):

```solidity
// compute values to swap to the target tick, price limit, or point where input/output amount is exhausted
(state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
    state.sqrtPriceX96,
    (zeroForOne ? step.sqrtPriceNextX96 < sqrtPriceLimitX96 : step.sqrtPriceNextX96 > sqrtPriceLimitX96)
        ? sqrtPriceLimitX96
        : step.sqrtPriceNextX96,
    state.liquidity,
    state.amountSpecifiedRemaining,
    fee
);
```

`SwapMath.computeSwapStep`은 현재 가격, 목표 가격, 사용 가능 유동성, 사용 가능 입력 토큰 등을 기반으로 이 단계에서 거래할 수 있는 최대 입력 토큰 양(`amountIn`), 출력 토큰 양(`amountOut`), 거래 수수료(`feeAmount`) 및 스왑 후 가격(`sqrtRatioNextX96`)을 계산합니다. [computeSwapStep](#computeSwapStep)을 참조하세요.

이번 거래의 `amountIn` 및 `amountOut` 양을 저장합니다:

```solidity
if (exactInput) {
    state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
    state.amountCalculated = state.amountCalculated.sub(step.amountOut.toInt256());
} else {
    state.amountSpecifiedRemaining += step.amountOut.toInt256();
    state.amountCalculated = state.amountCalculated.add((step.amountIn + step.feeAmount).toInt256());
}
```

* 입력 토큰 양(`token0` 또는 `token1`)을 지정하는 경우
    - `amountSpecifiedRemaining`은 남은 사용 가능 입력 토큰 양을 나타냄 (수수료 차감 후)
    - `amountCalculated`는 출력 토큰 양을 나타냄 (음수 값임에 주의)
* 출력 토큰 양(`token0` 또는 `token1`)을 지정하는 경우
    - `amountSpecifiedRemaining`은 남은 필요 출력 토큰 양을 나타냄 (초기에 음수이므로, 0에 도달할 때까지 각 스왑마다 `+= step.amountOut` 필요)
    - `amountCalculated`는 (수수료 포함) 사용된 입력 토큰 양을 나타냄

프로토콜 수수료를 계산합니다:

```solidity
// if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
    // if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
    if (cache.feeProtocol > 0) {
        uint256 delta = step.feeAmount / cache.feeProtocol;
        step.feeAmount -= delta;
        state.protocolFee += uint128(delta);
    }
```

프로토콜 수수료가 활성화된 경우 거래 수수료에서 프로토콜 수수료를 공제합니다. 참고로 `feeProtocol` 값은 거래 수수료의 $\frac{1}{n}$을 나타냅니다.

```solidity
    // update global fee tracker
    if (state.liquidity > 0)
        state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);
```

유동성 당 누적 수수료를 전역적으로 계산합니다.

```solidity
    // shift tick if we reached the next price
    if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
        // if the tick is initialized, run the tick transition
        if (step.initialized) {
            // check for the placeholder value, which we replace with the actual value the first time the swap
            // crosses an initialized tick
            if (!cache.computedLatestObservation) {
                (cache.tickCumulative, cache.secondsPerLiquidityCumulativeX128) = observations.observeSingle(
                    cache.blockTimestamp,
                    0,
                    slot0Start.tick,
                    slot0Start.observationIndex,
                    cache.liquidityStart,
                    slot0Start.observationCardinality
                );
                cache.computedLatestObservation = true;
            }
            int128 liquidityNet =
                ticks.cross(
                    step.tickNext,
                    (zeroForOne ? state.feeGrowthGlobalX128 : feeGrowthGlobal0X128),
                    (zeroForOne ? feeGrowthGlobal1X128 : state.feeGrowthGlobalX128),
                    cache.secondsPerLiquidityCumulativeX128,
                    cache.tickCumulative,
                    cache.blockTimestamp
                );
            // if we're moving leftward, we interpret liquidityNet as the opposite sign
            // safe because liquidityNet cannot be type(int128).min
            if (zeroForOne) liquidityNet = -liquidityNet;

            state.liquidity = LiquidityMath.addDelta(state.liquidity, liquidityNet);
        }

        state.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
    } else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
        // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
        state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
    }
```

* 스왑 후 가격이 목표 가격(즉, 다음 `tick`을 기준으로 계산된 가격)에 도달한 경우:
    * 해당 `tick`이 초기화되어 있다면:
        - `ticks.cross`를 통해 `tick`을 교차하고 관련 `Outside` 변수를 반대로 설정합니다.
        - `tick` 순 유동성 `liquidityNet`을 사용하여 사용 가능 유동성 `state.liquidity`를 업데이트합니다.
            - [mint](#mint)에서 `tick` 초기화 중 `tickLower`의 `liquidityNet`은 양수( `liquidityDelta`와 동일)인 반면, `tickUpper`의 `liquidityNet`은 음수( `-liquidityDelta`와 동일)입니다. 따라서 `zeroForOne`의 값에 따라 `liquidityNet`의 부호를 조정해야 합니다.
            - `zeroForOne = true`일 때, 거래가 진행됨에 따라 풀의 $x$ 양은 증가하고 $y$는 감소하며 가격 $\sqrt{P}$는 점차 감소합니다. `tick`은 낮은 방향으로 이동합니다. `tickLower`를 교차하면 범위를 벗어나는 것을 의미하므로 유동성을 줄여야 합니다. 반대로 `tickUpper`를 교차하면 범위로 들어가는 것을 의미하므로 유동성을 늘려야 합니다. 두 경우 모두 `-liquidityNet`을 사용합니다.
            - `zeroForOne = false`일 때, 거래가 진행됨에 따라 풀의 $y$ 양은 증가하고 $x$는 감소하며 가격 $\sqrt{P}$는 점차 증가합니다. `tick`은 높은 방향으로 이동합니다. `tickUpper`를 교차하면 범위를 벗어나는 것을 의미하므로 유동성을 줄여야 합니다. 반대로 `tickLower`를 교차하면 범위로 들어가는 것을 의미하므로 유동성을 늘려야 합니다. 두 경우 모두 `liquidityNet`을 사용합니다.
    * 현재 `tick`을 다음 `tick`으로 이동합니다.
* 스왑 후 가격이 이 단계의 목표 가격에 도달하지 않았지만 초기 가격과 같지 않은 경우, 즉 거래가 완료되었음을 나타냄:
    * 스왑 후 가격을 기준으로 최신 `tick` 값을 계산합니다.

스왑이 완전히 완료될 때까지 위 단계를 반복합니다.

스왑 완료 후 전역 상태를 업데이트합니다:

```solidity
// update tick and write an oracle entry if the tick change
if (state.tick != slot0Start.tick) {
    (uint16 observationIndex, uint16 observationCardinality) =
        observations.write(
            slot0Start.observationIndex,
            cache.blockTimestamp,
            slot0Start.tick,
            cache.liquidityStart,
            slot0Start.observationCardinality,
            slot0Start.observationCardinalityNext
        );
    (slot0.sqrtPriceX96, slot0.tick, slot0.observationIndex, slot0.observationCardinality) = (
        state.sqrtPriceX96,
        state.tick,
        observationIndex,
        observationCardinality
    );
} else {
    // otherwise just update the price
    slot0.sqrtPriceX96 = state.sqrtPriceX96;
}
```

* 스왑 후 `tick`이 스왑 전 `tick`과 다른 경우:
    * `tickCumulative`가 변경되었으므로 (오라클) 관측 점 데이터를 기록합니다.
    * `slot0.sqrtPriceX96`, `slot0.tick` 등을 업데이트합니다. 이때 `sqrtPriceX96`과 `tick`이 대응하지 않을 수 있으며 `sqrtPriceX96`만이 현재 가격을 정확하게 반영할 수 있음에 유의하십시오.
* 스왑 전후 `tick` 값이 동일한 경우 가격만 수정합니다:
    * `slot0.sqrtPriceX96`을 업데이트합니다.

마찬가지로 전역 유동성이 변경된 경우 `liquidity`를 업데이트합니다:

```solidity
// update liquidity if it changed
if (cache.liquidityStart != state.liquidity) liquidity = state.liquidity;
```

누적 수수료 및 프로토콜 수수료를 업데이트합니다:

```solidity
// update fee growth global and, if necessary, protocol fees
// overflow is acceptable, protocol has to withdraw before it hits type(uint128).max fees
if (zeroForOne) {
    feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;
    if (state.protocolFee > 0) protocolFees.token0 += state.protocolFee;
} else {
    feeGrowthGlobal1X128 = state.feeGrowthGlobalX128;
    if (state.protocolFee > 0) protocolFees.token1 += state.protocolFee;
}
```

`token0`에서 `token1`으로 스왑하는 경우 `token0`만 수수료로 징수할 수 있으며, 반대의 경우 `token1`만 수수료로 징수할 수 있습니다.

```solidity
(amount0, amount1) = zeroForOne == exactInput
    ? (amountSpecified - state.amountSpecifiedRemaining, state.amountCalculated)
    : (state.amountCalculated, amountSpecified - state.amountSpecifiedRemaining);
```

이번 스왑에 필요한 구체적인 `amount0` 및 `amount1`을 계산합니다.

```solidity
// do the transfers and collect payment
if (zeroForOne) {
    if (amount1 < 0) TransferHelper.safeTransfer(token1, recipient, uint256(-amount1));

    uint256 balance0Before = balance0();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
    require(balance0Before.add(uint256(amount0)) <= balance0(), 'IIA');
} else {
    if (amount0 < 0) TransferHelper.safeTransfer(token0, recipient, uint256(-amount0));

    uint256 balance1Before = balance1();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
    require(balance1Before.add(uint256(amount1)) <= balance1(), 'IIA');
}

emit Swap(msg.sender, recipient, amount0, amount1, state.sqrtPriceX96, state.liquidity, state.tick);
slot0.unlocked = true;
```

컨트랙트는 출력 토큰을 `recipient`에게 전송하고, 동시에 호출자는 `uniswapV3SwapCallback`에서 입력 토큰을 거래 쌍 컨트랙트로 전송해야 합니다.

이로써 전체 `swap` 과정이 마무리됩니다.

#### flash

이 메서드는 Uniswap v3의 플래시 론 기능을 구현합니다.

메서드 매개변수:

* `recipient`: 플래시 론 수령인.
* `amount0`: 대출한 `token0`의 양.
* `amount1`: 대출한 `token1`의 양.
* `data`: 콜백 메서드 매개변수.

```solidity
/// @inheritdoc IUniswapV3PoolActions
function flash(
    address recipient,
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) external override lock noDelegateCall {
    uint128 _liquidity = liquidity;
    require(_liquidity > 0, 'L');

    uint256 fee0 = FullMath.mulDivRoundingUp(amount0, fee, 1e6);
    uint256 fee1 = FullMath.mulDivRoundingUp(amount1, fee, 1e6);
```

플래시 론 수수료는 `swap` 수수료와 동일하며 $\frac{fee}{10^6}$입니다.

```solidity
    uint256 balance0Before = balance0();
    uint256 balance1Before = balance1();

    if (amount0 > 0) TransferHelper.safeTransfer(token0, recipient, amount0);
    if (amount1 > 0) TransferHelper.safeTransfer(token1, recipient, amount1);

    IUniswapV3FlashCallback(msg.sender).uniswapV3FlashCallback(fee0, fee1, data);

    uint256 balance0After = balance0();
    uint256 balance1After = balance1();

    require(balance0Before.add(fee0) <= balance0After, 'F0');
    require(balance1Before.add(fee1) <= balance1After, 'F1');
```

대출한 토큰 양을 수령인에게 전송하며, `flash` 메서드의 호출자는 `IUniswapV3FlashCallback.uniswapV3FlashCallback` 인터페이스 메서드를 구현하고 해당 메서드에서 수수료를 포함한 토큰을 반환해야 합니다.

```solidity
    // sub is safe because we know balanceAfter is gt balanceBefore by at least fee
    uint256 paid0 = balance0After - balance0Before;
    uint256 paid1 = balance1After - balance1Before;

    if (paid0 > 0) {
        uint8 feeProtocol0 = slot0.feeProtocol % 16;
        uint256 fees0 = feeProtocol0 == 0 ? 0 : paid0 / feeProtocol0;
        if (uint128(fees0) > 0) protocolFees.token0 += uint128(fees0);
        feeGrowthGlobal0X128 += FullMath.mulDiv(paid0 - fees0, FixedPoint128.Q128, _liquidity);
    }
    if (paid1 > 0) {
        uint8 feeProtocol1 = slot0.feeProtocol >> 4;
        uint256 fees1 = feeProtocol1 == 0 ? 0 : paid1 / feeProtocol1;
        if (uint128(fees1) > 0) protocolFees.token1 += uint128(fees1);
        feeGrowthGlobal1X128 += FullMath.mulDiv(paid1 - fees1, FixedPoint128.Q128, _liquidity);
    }

    emit Flash(msg.sender, recipient, amount0, amount1, paid0, paid1);
}
```

징수된 수수료를 기반으로 프로토콜 수수료를 계산하고(`token0`과 `token1`의 프로토콜 수수료는 별도로 설정됨에 유의, [setFeeProtocol](#setFeeProtocol) 참조), 마지막으로 `protocolFees` 및 `feeGrowthGlobal1X128`을 업데이트합니다.

#### collect

이 메서드는 유동성 소각 및 수수료 토큰을 포함한 토큰 회수 기능을 구현합니다.

매개변수는 다음과 같습니다:

- `recipient`: 토큰 수령인
- `tickLower`: 포지션 최저점
- `tickUpper`: 포지션 최고점
- `amount0Requested`: 회수 요청한 `token0`의 양
- `amount1Requested`: 회수 요청한 `token1`의 양

```solidity
/// @inheritdoc IUniswapV3PoolActions
function collect(
    address recipient,
    int24 tickLower,
    int24 tickUpper,
    uint128 amount0Requested,
    uint128 amount1Requested
) external override lock returns (uint128 amount0, uint128 amount1) {
    // we don't need to checkTicks here, because invalid positions will never have non-zero tokensOwed{0,1}
    Position.Info storage position = positions.get(msg.sender, tickLower, tickUpper);

    amount0 = amount0Requested > position.tokensOwed0 ? position.tokensOwed0 : amount0Requested;
    amount1 = amount1Requested > position.tokensOwed1 ? position.tokensOwed1 : amount1Requested;

    if (amount0 > 0) {
        position.tokensOwed0 -= amount0;
        TransferHelper.safeTransfer(token0, recipient, amount0);
    }
    if (amount1 > 0) {
        position.tokensOwed1 -= amount1;
        TransferHelper.safeTransfer(token1, recipient, amount1);
    }

    emit Collect(msg.sender, recipient, tickLower, tickUpper, amount0, amount1);
}
```

위의 코드는 비교적 간단하므로 여기서 확장하지 않습니다. 모든 토큰을 회수하려면 `type(uint128).max`를 사용하는 등 `tokensOwned`보다 큰 숫자를 지정해야 합니다.

#### increaseObservationCardinalityNext

오라클 관측 점을 위한 쓰기 가능한 공간을 확장합니다. 이 메서드는 `Oracle.sol`의 [grow](#grow) 메서드를 호출하여 확장을 달성합니다.

```solidity
/// @inheritdoc IUniswapV3PoolActions
function increaseObservationCardinalityNext(uint16 observationCardinalityNext)
    external
    override
    lock
    noDelegateCall
{
    uint16 observationCardinalityNextOld = slot0.observationCardinalityNext; // for the event
    uint16 observationCardinalityNextNew =
        observations.grow(observationCardinalityNextOld, observationCardinalityNext);
    slot0.observationCardinalityNext = observationCardinalityNextNew;
    if (observationCardinalityNextOld != observationCardinalityNextNew)
        emit IncreaseObservationCardinalityNext(observationCardinalityNextOld, observationCardinalityNextNew);
}
```

#### observe

지정된 시간의 관측 점 데이터를 대량으로 검색합니다. 이 메서드는 `Oracle.sol`의 [observe](#observe2)를 호출하여 이를 구현합니다.

```solidity
/// @inheritdoc IUniswapV3PoolDerivedState
function observe(uint32[] calldata secondsAgos)
    external
    view
    override
    noDelegateCall
    returns (int56[] memory tickCumulatives, uint160[] memory secondsPerLiquidityCumulativeX128s)
{
    return
        observations.observe(
            _blockTimestamp(),
            secondsAgos,
            slot0.tick,
            slot0.observationIndex,
            liquidity,
            slot0.observationCardinality
        );
}
```

#### setFeeProtocol

프로토콜 수수료 비율을 설정합니다. 이 메서드는 팩토리 컨트랙트의 소유자만 실행할 수 있습니다.

`token0`과 `token1`의 프로토콜 수수료 비율은 별도로 설정해야 합니다. 비율은 거래 수수료의 일부이며, 유효한 값은 0(프로토콜 수수료 비활성화) 또는 $4 \leq n \leq 10$입니다. 즉, 프로토콜 수수료는 거래 수수료의 $\frac{1}{n}$로 설정될 수 있습니다.

```solidity
/// @inheritdoc IUniswapV3PoolOwnerActions
function setFeeProtocol(uint8 feeProtocol0, uint8 feeProtocol1) external override lock onlyFactoryOwner {
    require(
        (feeProtocol0 == 0 || (feeProtocol0 >= 4 && feeProtocol0 <= 10)) &&
            (feeProtocol1 == 0 || (feeProtocol1 >= 4 && feeProtocol1 <= 10))
    );
    uint8 feeProtocolOld = slot0.feeProtocol;
    slot0.feeProtocol = feeProtocol0 + (feeProtocol1 << 4);
    emit SetFeeProtocol(feeProtocolOld % 16, feeProtocolOld >> 4, feeProtocol0, feeProtocol1);
}
```

`slot0.feeProtocol`의 유형은 `uint8`이며, 상위 4비트는 `token1`, 하위 4비트는 `token0`에 대한 두 토큰의 프로토콜 수수료 비율을 저장합니다:

$$
slot0.feeProtocol = \overbrace{0000}^{fee1}\overbrace{0000}^{fee0}
$$

따라서 `feeProtocolOld % 16`은 `token0`에 대한 프로토콜 수수료 비율 `fee0`을 나타내고, `feeProtocolOld >> 4`는 `token1`에 대한 프로토콜 수수료 비율 `fee1`을 나타냅니다.

#### collectProtocol

프로토콜 수수료를 회수합니다. 이 메서드는 팩토리 컨트랙트의 소유자만 실행할 수 있습니다.

프로토콜 수수료는 두 가지 출처에서 발생합니다:

- `swap`에 의해 생성된 거래 수수료
- `flash`에 의해 생성된 플래시 론 수수료

```solidity
/// @inheritdoc IUniswapV3PoolOwnerActions
function collectProtocol(
    address recipient,
    uint128 amount0Requested,
    uint128 amount1Requested
) external override lock onlyFactoryOwner returns (uint128 amount0, uint128 amount1) {
    amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
    amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

    if (amount0 > 0) {
        if (amount0 == protocolFees.token0) amount0--; // ensure that the slot is not cleared, for gas savings
        protocolFees.token0 -= amount0;
        TransferHelper.safeTransfer(token0, recipient, amount0);
    }
    if (amount1 > 0) {
        if (amount1 == protocolFees.token1) amount1--; // ensure that the slot is not cleared, for gas savings
        protocolFees.token1 -= amount1;
        TransferHelper.safeTransfer(token1, recipient, amount1);
    }

    emit CollectProtocol(msg.sender, recipient, amount0, amount1);
}
```

위의 코드는 비교적 간단하므로 더 확장하지 않습니다.

### Tick.sol

Tick.sol은 Tick의 내부 상태를 관리합니다.

#### tickSpacingToMaxLiquidityPerTick

`tickSpacing`을 기반으로 `tick`당 최대 유동성을 계산합니다. `tickSpacing`으로 나누어떨어지는 `tick`만 유동성을 저장할 수 있습니다:

```solidity
/// @notice Derives max liquidity per tick from given tick spacing
/// @dev Executed within the pool constructor
/// @param tickSpacing The amount of required tick separation, realized in multiples of `tickSpacing`
///     e.g., a tickSpacing of 3 requires ticks to be initialized every 3rd tick i.e., ..., -6, -3, 0, 3, 6, ...
/// @return The max liquidity per tick
function tickSpacingToMaxLiquidityPerTick(int24 tickSpacing) internal pure returns (uint128) {
    int24 minTick = (TickMath.MIN_TICK / tickSpacing) * tickSpacing;
    int24 maxTick = (TickMath.MAX_TICK / tickSpacing) * tickSpacing;
    uint24 numTicks = uint24((maxTick - minTick) / tickSpacing) + 1;
    return type(uint128).max / numTicks;
}
```

#### getFeeGrowthInside

두 `tick` 범위 내부의 유동성 당 누적 수수료를 계산하며, 백서의 공식 6.17-6.19를 구현합니다:

$$
f_a(i) = \begin{cases} f_g - f_o(i) & \text{if $i_c \geq i$}\\
f_o(i) & \text{$i_c < i$} \end{cases} \quad \text{(6.17)}
$$

$$
f_b(i) = \begin{cases} f_o(i) & \text{if $i_c \geq i$}\\
f_g - f_o(i) & \text{$i_c < i$}\end{cases} \quad \text{(6.18)}
$$

$$
f_r = f_g - f_b(i_l) - f_a(i_u) \quad \text{(6.19)}
$$

코드는 다음과 같습니다:

```solidity
/// @notice Retrieves fee growth data
/// @param self The mapping containing all tick information for initialized ticks
/// @param tickLower The lower tick boundary of the position
/// @param tickUpper The upper tick boundary of the position
/// @param tickCurrent The current tick
/// @param feeGrowthGlobal0X128 The all-time global fee growth, per unit of liquidity, in token0
/// @param feeGrowthGlobal1X128 The all-time global fee growth, per unit of liquidity, in token1
/// @return feeGrowthInside0X128 The all-time fee growth in token0, per unit of liquidity, inside the position's tick boundaries
/// @return feeGrowthInside1X128 The all-time fee growth in token1, per unit of liquidity, inside the position's tick boundaries
function getFeeGrowthInside(
    mapping(int24 => Tick.Info) storage self,
    int24 tickLower,
    int24 tickUpper,
    int24 tickCurrent,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128
) internal view returns (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) {
    Info storage lower = self[tickLower];
    Info storage upper = self[tickUpper];

    // calculate fee growth below
    uint256 feeGrowthBelow0X128;
    uint256 feeGrowthBelow1X128;
    if (tickCurrent >= tickLower) {
        feeGrowthBelow0X128 = lower.feeGrowthOutside0X128;
        feeGrowthBelow1X128 = lower.feeGrowthOutside1X128;
    } else {
        feeGrowthBelow0X128 = feeGrowthGlobal0X128 - lower.feeGrowthOutside0X128;
        feeGrowthBelow1X128 = feeGrowthGlobal1X128 - lower.feeGrowthOutside1X128;
    }

    // calculate fee growth above
    uint256 feeGrowthAbove0X128;
    uint256 feeGrowthAbove1X128;
    if (tickCurrent < tickUpper) {
        feeGrowthAbove0X128 = upper.feeGrowthOutside0X128;
        feeGrowthAbove1X128 = upper.feeGrowthOutside1X128;
    } else {
        feeGrowthAbove0X128 = feeGrowthGlobal0X128 - upper.feeGrowthOutside0X128;
        feeGrowthAbove1X128 = feeGrowthGlobal1X128 - upper.feeGrowthOutside1X128;
    }

    feeGrowthInside0X128 = feeGrowthGlobal0X128 - feeGrowthBelow0X128 - feeGrowthAbove0X128;
    feeGrowthInside1X128 = feeGrowthGlobal1X128 - feeGrowthBelow1X128 - feeGrowthAbove1X128;
}
```

먼저 현재 `tickCurrent`를 기준으로 `tickLower` 및 `tickUpper`에 대한 $f_a$와 $f_b$를 계산하고, 마지막으로 범위 내 수수료 $f_r$을 계산합니다:

$$
\underbrace{\overbrace{..., i_l - 1}^{f_b(i_l)}, \overbrace{i_l, i_l + 1, ..., i_u - 1, i_u}^{f_r}, \overbrace{i_u + 1, ...}^{f_a(i_u)}}_{f_g}
$$

범위 내부의 유동성 당 누적 수수료를 계산하는 이유는 무엇일까요? 각 포지션(`Position`)이 `mint`/`burn` 시 이 값을 기반으로 수취 가능한 자체 수수료를 계산하기 때문입니다:

$$
liquidityDelta \cdot (feeGrowthInside - feeGrowthInsideLast)
$$

#### update

`tick` 상태를 업데이트하고 `tick`이 초기화됨에서 미초기화됨으로, 또는 그 반대로 뒤집혔는지 여부를 반환합니다:

```solidity
/// @notice Updates a tick and returns true if the tick was flipped from initialized to uninitialized, or vice versa
/// @param self The mapping containing all tick information for initialized ticks
/// @param tick The tick that will be updated
/// @param tickCurrent The current tick
/// @param liquidityDelta A new amount of liquidity to be added (subtracted) when tick is crossed from left to right (right to left)
/// @param feeGrowthGlobal0X128 The all-time global fee growth, per unit of liquidity, in token0
/// @param feeGrowthGlobal1X128 The all-time global fee growth, per unit of liquidity, in token1
/// @param secondsPerLiquidityCumulativeX128 The all-time seconds per max(1, liquidity) of the pool
/// @param tickCumulative The tick * time elapsed since the pool was first initialized
/// @param time The current block timestamp cast to a uint32
/// @param upper true for updating a position's upper tick, or false for updating a position's lower tick
/// @param maxLiquidity The maximum liquidity allocation for a single tick
/// @return flipped Whether the tick was flipped from initialized to uninitialized, or vice versa
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    int24 tickCurrent,
    int128 liquidityDelta,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128,
    uint160 secondsPerLiquidityCumulativeX128,
    int56 tickCumulative,
    uint32 time,
    bool upper,
    uint128 maxLiquidity
) internal returns (bool flipped) {
    Tick.Info storage info = self[tick];

    uint128 liquidityGrossBefore = info.liquidityGross;
    uint128 liquidityGrossAfter = LiquidityMath.addDelta(liquidityGrossBefore, liquidityDelta);

    require(liquidityGrossAfter <= maxLiquidity, 'LO');

    flipped = (liquidityGrossAfter == 0) != (liquidityGrossBefore == 0);
```

`tick`이 유동성이 없는 상태에서 있는 상태로, 또는 유동성이 있는 상태에서 없는 상태로 전환되면 해당 `tick`이 뒤집어져야 함(`flipped`)을 나타냅니다.

```solidity
    if (liquidityGrossBefore == 0) {
        // by convention, we assume that all growth before a tick was initialized happened _below_ the tick
        if (tick <= tickCurrent) {
            info.feeGrowthOutside0X128 = feeGrowthGlobal0X128;
            info.feeGrowthOutside1X128 = feeGrowthGlobal1X128;
            info.secondsPerLiquidityOutsideX128 = secondsPerLiquidityCumulativeX128;
            info.tickCumulativeOutside = tickCumulative;
            info.secondsOutside = time;
        }
        info.initialized = true;
    }
```

`tick`에 이전에 유동성이 없었다면 초기화합니다. 현재 `tickCurrent`보다 작은 틱의 경우 `Outside` 변수를 설정합니다.

```solidity
    info.liquidityGross = liquidityGrossAfter;
```

`liquidityGross`는 총 유동성을 나타내며 `tick`의 초기화 여부를 결정하는 데 사용됩니다:

- `mint` 작업의 경우 유동성이 증가하고, `burn` 작업의 경우 유동성이 감소합니다.
- 이 변수는 `tick`이 서로 다른 포지션에서 경계 저점(`tickLower`) 또는 고점(`tickUpper`)으로 사용되는지 여부와는 무관합니다. 오직 `mint` 또는 `burn` 작업과만 관련이 있습니다.
- `tick`이 `tickLower`와 `tickUpper`로 동시에 사용되는 경우 `liquidityNet`은 0일 수 있지만 `liquidityGross`는 여전히 0보다 큽니다. 따라서 재초기화가 필요하지 않습니다.

```solidity
    // when the lower (upper) tick is crossed left to right (right to left), liquidity must be added (removed)
    info.liquidityNet = upper
        ? int256(info.liquidityNet).sub(liquidityDelta).toInt128()
        : int256(info.liquidityNet).add(liquidityDelta).toInt128();
}
```

`liquidityNet`은 순 유동성을 나타내며, `swap`이 `tick`을 교차할 때 전역 사용 가능 유동성 `liquidity`를 업데이트하는 데 사용됩니다:

- `tickLower`, 즉 하한(왼쪽 경계)으로 작용하는 경우 `liquidityDelta`를 증가시킵니다(`mint`는 양수, `burn`은 음수).
- `tickUpper`, 즉 상한(오른쪽 경계)으로 작용하는 경우 `liquidityDelta`를 감소시킵니다(`mint`는 양수, `burn`은 음수).

#### clear

`tick`이 뒤집혔을 때 해당 `tick`과 관련된 유동성이 없으면, 즉 `liquidityGross = 0`이면 `tick` 상태를 지웁니다:

```solidity
/// @notice Clears tick data
/// @param self The mapping containing all initialized tick information for initialized ticks
/// @param tick The tick that will be cleared
function clear(mapping(int24 => Tick.Info) storage self, int24 tick) internal {
    delete self[tick];
}
```

#### cross

`tick`이 교차될 때 백서의 공식 6.20에 따라 `Outside` 변수의 방향을 뒤집어야 합니다:

$$
f_o(i) := f_g - f_o(i) \quad \text{(6.20)}
$$

이 변수들은 [getFeeGrowthInside](#getFeeGrowthInside)와 같은 메서드에서 사용됩니다.

```solidity
/// @notice Transitions to next tick as needed by price movement
/// @param self The mapping containing all tick information for initialized ticks
/// @param tick The destination tick of the transition
/// @param feeGrowthGlobal0X128 The all-time global fee growth, per unit of liquidity, in token0
/// @param feeGrowthGlobal1X128 The all-time global fee growth, per unit of liquidity, in token1
/// @param secondsPerLiquidityCumulativeX128 The current seconds per liquidity
/// @param tickCumulative The tick * time elapsed since the pool was first initialized
/// @param time The current block.timestamp
/// @return liquidityNet The amount of liquidity added (subtracted) when tick is crossed from left to right (right to left)
function cross(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128,
    uint160 secondsPerLiquidityCumulativeX128,
    int56 tickCumulative,
    uint32 time
) internal returns (int128 liquidityNet) {
    Tick.Info storage info = self[tick];
    info.feeGrowthOutside0X128 = feeGrowthGlobal0X128 - info.feeGrowthOutside0X128;
    info.feeGrowthOutside1X128 = feeGrowthGlobal1X128 - info.feeGrowthOutside1X128;
    info.secondsPerLiquidityOutsideX128 = secondsPerLiquidityCumulativeX128 - info.secondsPerLiquidityOutsideX128;
    info.tickCumulativeOutside = tickCumulative - info.tickCumulativeOutside;
    info.secondsOutside = time - info.secondsOutside;
    liquidityNet = info.liquidityNet;
}
```

### TickMath.sol

TickMath는 주로 두 가지 메서드를 포함합니다:

- [getSqrtRatioAtTick](#getSqrtRatioAtTick): 틱을 기반으로 제곱근 가격 $\sqrt{P}$를 계산합니다.
- [getTickAtSqrtRatio](#getTickAtSqrtRatio): 제곱근 가격 $\sqrt{P}$를 기반으로 틱을 계산합니다.

#### getSqrtRatioAtTick

이 메서드는 백서의 공식 6.2에 해당합니다:

$$
\sqrt{p}(i) = \sqrt{1.0001}^i = 1.0001^{\frac{i}{2}}
$$

여기서 $i$는 `tick`입니다.

Uniswap v3는 $[2^{-128}, 2^{128}]$의 가격 범위($\frac{token1}{token0}$)를 지원하므로 백서의 공식 6.1에 따라:

$$
p(i) = 1.0001^i
$$

해당하는 최대 틱(MAX_TICK)은 다음과 같습니다:

$$
i = \lfloor log_{1.0001}{p(i)} \rfloor = \lfloor log_{1.0001}{2^{128}} \rfloor = \lfloor 887272.7517970635 \rfloor = 887272
$$

최소 틱(MIN_TICK)은 다음과 같습니다:

$$
i = \lceil log_{1.0001}{2^{-128}} \rceil = \lceil -887272.7517970635 \rceil = -887272
$$

$i \geq 0$이라고 가정하면 주어진 틱 $i$에 대해 항상 이진수로 표현할 수 있으므로 다음 공식이 항상 성립합니다:

$$
\begin{cases} i = \sum_{n=0}^{19}{(x_n \cdot 2^n)} = x_0 \cdot 1 + x_1 \cdot 2 + x_2 \cdot 4 + ... + x_{19}\cdot 524288 \\
\forall x_n \in \{0, 1\} \end{cases} \quad \text{(1.1)}
$$

여기서 $x_n$은 $i$의 이진수 자릿수입니다. 예를 들어 $i=6$인 경우 이진 표현은 `000000000000000000000110`이므로 $x_1 = 1, x_2 = 1$이고 나머지 $x_n$은 0입니다.

마찬가지로 $i < 0$인 경우도 유사한 공식으로 표현할 수 있음을 유추할 수 있습니다.

먼저 $i < 0$인 경우를 살펴보겠습니다:

$i < 0$이면:

$$
\sqrt{p}(i) = 1.0001^{\frac{i}{2}} = 1.0001^{-\frac{|i|}{2}} = \frac{1}{1.0001^{\frac{|i|}{2}}} = \frac{1}{1.0001^{\frac{1}{2}(\sum_{n=0}^{19}{(x_n \cdot 2^n)})}} \\
= \frac{1}{1.0001^{\frac{1}{2} \cdot x_0}} \cdot \frac{1}{1.0001^{\frac{2}{2} \cdot x_1}} \cdot \frac{1}{1.0001^{\frac{4}{2} \cdot x_2}} \cdot ... \cdot \frac{1}{1.0001^{\frac{524288}{2} \cdot x_{19}}}
$$

이진수 자릿수 $x_n$의 값에 따라 다음과 같이 요약할 수 있습니다:

$$
\frac{1}{1.0001^{\frac{x_n \cdot 2^n}{2}}} \begin{cases} = 1 & \text{if $x_n = 0, n \geq 0, i < 0$}\\
< 1 & \text{if $x_n = 1, n \geq 0, i < 0$}
\end{cases}
$$

계산 중 정밀도 오류를 최소화하기 위해 `Q128.128`(128비트 고정 소수점 숫자)을 사용하여 중간 가격을 나타내며, 각 가격 $p$에 대해 128비트 왼쪽으로 시프트해야 합니다. $i < 0, x_n = 1$이면 $\frac{1}{1.0001^{\frac{x_n \cdot 2^n}{2}}} < 1$이므로 연속 곱셈 과정에서 오버플로 문제가 발생하지 않습니다.

$\sqrt{p}(i)$를 계산하는 방법은 다음과 같이 요약할 수 있습니다:

- 값 1로 시작하여 0번째 비트부터 시작하여 $i$의 이진 비트를 낮은 곳에서 높은 곳으로(오른쪽에서 왼쪽으로) 반복합니다.
- 비트가 0이 아니면 $\frac{2^{128}}{1.0001^{\frac{2^n}{2}}}$를 곱합니다. 여기서 $2^{128}$은 128비트 왼쪽 시프트를 나타냅니다.
- 비트가 0이면 1을 곱하므로 생략할 수 있습니다.

```solidity
/// @notice Calculates sqrt(1.0001^tick) * 2^96
/// @dev Throws if |tick| > max tick
/// @param tick The input tick for the above formula
/// @return sqrtPriceX96 A Fixed point Q64.96 number representing the sqrt of the ratio of the two assets (token1/token0)
/// at the given tick
function getSqrtRatioAtTick(int24 tick) internal pure returns (uint160 sqrtPriceX96) {
    uint256 absTick = tick < 0 ? uint256(-int256(tick)) : uint256(int256(tick));
    require(absTick <= uint256(MAX_TICK), 'T');

    // If the 0th bit is not 0, then ratio = 0xfffcb933bd6fad37aa2d162d1a594001, which is 2^128 / 1.0001^0.5
    uint256 ratio = absTick & 0x1 != 0 ? 0xfffcb933bd6fad37aa2d162d1a594001 : 0x100000000000000000000000000000000;
    // If the 1st bit is not 0, then multiply by 0xfff97272373d413259a46990580e213a, which is 2^128 / 1.0001^1, since both multipliers are Q128.128, the final result is multiplied by 2^128, so it needs to be right-shifted by 128
    if (absTick & 0x2 != 0) ratio = (ratio * 0xfff97272373d413259a46990580e213a) >> 128;
    // If the 2nd bit is not 0, then multiply by 0xfff2e50f5f656932ef12357cf3c7fdcc, which is 2^128 / 1.0001^2,
    if (absTick & 0x4 != 0) ratio = (ratio * 0xfff2e50f5f656932ef12357cf3c7fdcc) >> 128;
    // And so on
    if (absTick & 0x8 != 0) ratio = (ratio * 0xffe5caca7e10e4e61c3624eaa0941cd0) >> 128;
    if (absTick & 0x10 != 0) ratio = (ratio * 0xffcb9843d60f6159c9db58835c926644) >> 128;
    if (absTick & 0x20 != 0) ratio = (ratio * 0xff973b41fa98c081472e6896dfb254c0) >> 128;
    if (absTick & 0x40 != 0) ratio = (ratio * 0xff2ea16466c96a3843ec78b326b52861) >> 128;
    if (absTick & 0x80 != 0) ratio = (ratio * 0xfe5dee046a99a2a811c461f1969c3053) >> 128;
    if (absTick & 0x100 != 0) ratio = (ratio * 0xfcbe86c7900a88aedcffc83b479aa3a4) >> 128;
    if (absTick & 0x200 != 0) ratio = (ratio * 0xf987a7253ac413176f2b074cf7815e54) >> 128;
    if (absTick & 0x400 != 0) ratio = (ratio * 0xf3392b0822b70005940c7a398e4b70f3) >> 128;
    if (absTick & 0x800 != 0) ratio = (ratio * 0xe7159475a2c29b7443b29c7fa6e889d9) >> 128;
    if (absTick & 0x1000 != 0) ratio = (ratio * 0xd097f3bdfd2022b8845ad8f792aa5825) >> 128;
    if (absTick & 0x2000 != 0) ratio = (ratio * 0xa9f746462d870fdf8a65dc1f90e061e5) >> 128;
    if (absTick & 0x4000 != 0) ratio = (ratio * 0x70d869a156d2a1b890bb3df62baf32f7) >> 128;
    if (absTick & 0x8000 != 0) ratio = (ratio * 0x31be135f97d08fd981231505542fcfa6) >> 128;
    if (absTick & 0x10000 != 0) ratio = (ratio * 0x9aa508b5b7a84e1c677de54f3e99bc9) >> 128;
    if (absTick & 0x20000 != 0) ratio = (ratio * 0x5d6af8dedb81196699c329225ee604) >> 128;
    if (absTick & 0x40000 != 0) ratio = (ratio * 0x2216e584f5fa1ea926041bedfe98) >> 128;
    // If the 19th bit is not 0, since (2^19 = 0x80000=524288), then multiply by 0x2216e584f5fa1ea926041bedfe98, which is 2^128 / 1.0001^(524288/2)
    // The maximum value of tick is 887272, so its binary representation only needs up to 20 bits, starting from 0, the last bit is the 19th bit.
    if (absTick & 0x80000 != 0) ratio = (ratio * 0x48a170391f7dc42444e8fa2) >> 128;

    if (tick > 0) ratio = type(uint256).max / ratio;

    // this divides by 1<<32 rounding up to go from a Q128.128 to a Q128.96.
    // we then downcast because we know the result always fits within 160 bits due to our tick input constraint
    // we round up in the division so getTickAtSqrtRatio of the output price is always consistent
    sqrtPriceX96 = uint160((ratio >> 32) + (ratio % (1 << 32) == 0 ? 0 : 1));
}
```

$i > 0$이라고 가정하면:

$$
\sqrt{p_{Q128128}(i)} = 2^{128} \cdot \sqrt{p(i)} = 2^{128} \cdot 1.0001^{\frac{i}{2}} \\
= \frac{2^{128}}{1.0001^{-\frac{i}{2}}} = \frac{2^{256}}{2^{128} \cdot \sqrt{p(-i)}} = \frac{2^{256}}{\sqrt{p_{Q128128}(-i)}}
$$

따라서 $i < 0$에 대한 비율 값을 계산하고 $2^{256}$을 비율로 나누어 $i > 0$에 대한 비율 값을 `Q128.128` 표현으로 얻으면 됩니다:

```solidity
if (tick > 0) ratio = type(uint256).max / ratio;
```

코드의 마지막 줄은 비율을 32비트 오른쪽으로 시프트하여 `Q128.96` 형식으로 변환합니다:

```solidity
sqrtPriceX96 = uint160((ratio >> 32) + (ratio % (1 << 32) == 0 ? 0 : 1));
```

이것은 제곱근 가격 $\sqrt{p}$를 계산하는 것이며, 최대 가격 $p$가 $2^{128}$이므로 $\sqrt{p}$의 최대값은 $2^{64}$입니다. 즉, 정수 부분은 최대 64비트만 필요하므로 최종 sqrtPriceX96은 160비트(64+96, 즉 `Q64.96` 형식)로 확실히 표현할 수 있습니다.

#### getTickAtSqrtRatio

이 메서드는 백서의 공식 6.8에 해당합니다:

$$
i_c = \lfloor \log_{\sqrt{1.0001}} \sqrt{P} \rfloor
$$

이 메서드는 Solidity에서 로그 계산을 포함하며, 로그 공식을 기반으로 유추할 수 있습니다:

$$
\log_{\sqrt{1.0001}} \sqrt{P} = \frac{log_2{\sqrt{P}}}{log_2{\sqrt{1.0001}}} = log_2{\sqrt{P}}  \cdot log_{\sqrt{1.0001}}{2}
$$

$log_{\sqrt{1.0001}}{2}$는 상수이므로 $log_2{\sqrt{P}}$만 계산하면 됩니다.

입력 매개변수 $\sqrt{P}$를 $x$로 고려하면 문제는 $log_2{x}$를 계산하는 것이 됩니다.

결과를 정수 부분 $n$과 소수 부분 $m$으로 나누면:

$$
n \leq log_2{x} = n + m < n + 1
$$

##### 정수 부분

$n$에 대해, 다음 때문에:

$$
2^n \leq x < 2^{n+1}
$$

이진 검색을 사용하여 $n$을 찾을 수 있습니다:

- 256비트 숫자의 경우 $0 \leq n < 256$, 이는 8개의 이진 자릿수로 표현할 수 있습니다.
- 8번째 이진 자릿수(k=7)부터 1번째 자릿수(k=0)까지 (가장 높은 것부터 가장 낮은 것까지) $x$가 $2^{2^k} - 1$보다 큰지 비교합니다. 만약 그렇다면 해당 자릿수를 1로 표시하고 $2^k$ 비트만큼 오른쪽으로 시프트합니다. 그렇지 않으면 0으로 표시합니다.
- 표시된 8개의 이진 자릿수가 $n$의 값입니다.

Python 코드 설명은 다음과 같습니다:

```python
def find_msb(x):
    msb = 0
    for k in reversed(range(8)): # k = 7, 6, 5. 4, 3, 2, 1, 0
        if x > 2 ** (2 ** k) - 1:
            msb += 2 ** k # 이 자릿수를 1로 표시, 즉 2 ** k를 더함
            x /= 2 ** (2 ** k) # 2 ** k 비트만큼 오른쪽 시프트
    return msb
```

Uniswap v3의 Solidity 코드는 다음과 같습니다(코드 주석 참고):

```solidity
/// @notice Calculates the greatest tick value such that getRatioAtTick(tick) <= ratio
/// @dev Throws in case sqrtPriceX96 < MIN_SQRT_RATIO, as MIN_SQRT_RATIO is the lowest value getRatioAtTick may
/// ever return.
/// @param sqrtPriceX96 The sqrt ratio for which to compute the tick as a Q64.96
/// @return tick The greatest tick for which the ratio is less than or equal to the input ratio
function getTickAtSqrtRatio(uint160 sqrtPriceX96) internal pure returns (int24 tick) {
    // second inequality must be < because the price can never reach the price at the max tick
    require(sqrtPriceX96 >= MIN_SQRT_RATIO && sqrtPriceX96 < MAX_SQRT_RATIO, 'R');
    uint256 ratio = uint256(sqrtPriceX96) << 32; // Left-shift by 32 bits, converting to Q128.128 format

    uint256 r = ratio;
    uint256 msb = 0;

    assembly {
        // If greater than 2 ** (2 ** 7) - 1, save the temporary variable: 2 ** 7
        let f := shl(7, gt(r, 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF))
        // msb += 2 ** 7
        msb := or(msb, f)
        // r /= (2 ** (2 ** 7)), i.e., right-shift by 2 ** 7
        r := shr(f, r)
    }
    assembly {
        let f := shl(6, gt(r, 0xFFFFFFFFFFFFFFFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(5, gt(r, 0xFFFFFFFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(4, gt(r, 0xFFFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(3, gt(r, 0xFF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(2, gt(r, 0xF))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := shl(1, gt(r, 0x3))
        msb := or(msb, f)
        r := shr(f, r)
    }
    assembly {
        let f := gt(r, 0x1)
        msb := or(msb, f)
    }
```

##### 소수 부분

소수 부분 $m$에 대해서는 다음이 성립합니다:

$$
0 \leq m = log_2{x} - n = log_2{\frac{x}{2^n}} < 1 \quad \text{(1.2)}
$$

여기서 $n$은 앞에서 계산한 msb, 즉 정수 부분입니다.

먼저 $\frac{x}{2^n}$ 전체를 하나의 값 $r$로 두면:

$$
0 \leq log_2{r} < 1
$$

$$
1 \leq r = \frac{x}{2^n} < 2
$$

이제 목표는 $log_2{r}$를 구하는 것입니다. `log_2{r}`를 충분히 많은 소수 자릿수를 가진 수렴 급수로 표현할 수 있다면 그 값을 근사할 수 있습니다.

로그의 성질을 이용하면 다음 두 식을 얻을 수 있습니다:

$$
log_2{r} = \frac{2 \cdot log_2{r}}{2} = \frac{log_2{r^2}}{2} \quad \text{(1.3)}
$$

$$
log_2{r} = log_2{2 \cdot \frac{r}{2}} = 1 + log_2{\frac{r}{2}} \quad \text{(1.4)}
$$

위 두 공식을 반복 적용하면 다음과 같은 절차로 정리할 수 있습니다:

1. 처음에는 항상 $log_2{r} < 1$이므로 먼저 공식 1.3을 적용해 문제를 $log_2{r^2}$를 구하는 형태로 바꿉니다. 이 시점의 가중치는 $\frac{1}{2}$입니다.
   - 실제로는 1단계에 들어갈 때마다 가중치가 이전의 절반으로 줄어듭니다. 예를 들어 두 번째 진입에서는 $\frac{1}{4}$, 세 번째 진입에서는 $\frac{1}{8}$이 됩니다.
2. 만약 $r^2 \geq 2$라면 공식 1.4를 적용해 1을 분리하고, 문제를 $log_2{\frac{r^2}{2}}$를 구하는 것으로 바꿉니다.
   - 이 판단은 공식 1.3 이후에 일어나므로, 분리된 1에는 직전 1단계의 가중치를 곱해 기록해야 합니다. 첫 번째라면 $\frac{1}{2}$, 두 번째라면 $\frac{1}{4}$를 기록하는 식입니다.
   - 또한 $1 \leq r < 2$이고 $2 \leq r^2 < 4$이므로, $1 \leq \frac{r^2}{2} < 2$가 됩니다. 따라서 $\frac{r^2}{2}$를 다시 하나의 값 $r$로 두고 1단계로 돌아가면 됩니다.
3. 만약 $r^2 < 2$라면 1단계로 돌아가 계속 반복합니다.

이 과정을 다음 식으로 정리할 수 있습니다:

$$
log_2{r} = m_1 \cdot \frac{1}{2} + m_2 \cdot \frac{1}{4} + ... + m_n \cdot \frac{1}{2^n} = \sum^{\infty}_{i=1}(m_i \cdot \frac{1}{2^i}) \quad \text{(1.5)}
$$

여기서 $\forall m_i \in \{0, 1\}$입니다.

이는 사실상 소수의 이진 표현입니다. 첫 번째 이진 자리는 $2^{-1}$, 두 번째는 $2^{-2}$를 나타냅니다. 위 절차에서 2단계로 들어가면 해당 자리를 1로 기록하는 것과 같고, 3단계로 가면 0으로 기록하는 것과 같습니다.

이 과정을 반복하는 것은 소수 부분의 각 이진 비트를 높은 자리에서 낮은 자리로(왼쪽에서 오른쪽으로) 하나씩 결정하는 과정입니다. 반복 횟수가 많을수록 계산된 $log_2{r}$의 정밀도는 높아집니다.

이제 Uniswap v3에서 소수 부분을 계산하는 코드를 이어서 보겠습니다:

```solidity
        if (msb >= 128) r = ratio >> (msb - 127);
        else r = ratio << (127 - msb);
```

여기서 msb는 정수 부분 $n$입니다. ratio는 `Q128.128` 형식이므로 `msb >= 128`이면 `ratio >= 1`이고, 이 경우 정수 자릿수만큼 오른쪽으로 시프트해 소수 부분을 꺼내야 합니다. `-127`은 결과를 `Q129.127` 형식으로 맞추기 위해 127비트 왼쪽 정렬하는 의미입니다. 반대로 `msb < 128`이면 `ratio < 1`이므로 정수 부분이 없고, `127 - msb`만큼 왼쪽으로 시프트해 마찬가지로 `Q129.127` 형식으로 맞춥니다.

실제로 `ratio >> msb`는 공식 1.2의 $\frac{x}{2^n}$, 즉 1단계에서 사용한 $r$에 해당하며 이후의 반복 알고리즘(1~3단계)에 사용됩니다.

```solidity
    int256 log_2 = (int256(msb) - 128) << 64;
```
msb는 `Q128.128` ratio를 기반으로 계산되므로 `int256(msb) - 128`은 $n$의 실제 값을 나타냅니다. `<< 64`는 $n$을 나타내기 위해 `Q192.64`를 사용합니다.
이 코드 라인은 실제로 정수 부분의 값을 저장하기 위해 `Q192.64`를 사용하고 있습니다.

다음 코드는 반복문을 통해 소수 부분의 이진 표현의 처음 14자리를 계산합니다:

```solidity
    assembly {
        // 1단계에 따라 r^2를 계산합니다. 두 r 모두 Q129.127이므로 127만큼 오른쪽으로 시프트합니다.
        r := shr(127, mul(r, r))
        // 1 <= r^2 < 4이므로 r^2의 정수 부분을 나타내는 데 2자리만 필요합니다.
        // 따라서 오른쪽에서 세어 129번째와 128번째 자리가 r^2의 정수 부분을 나타냅니다.
        // 128자리만큼 오른쪽으로 시프트하여 129번째 자리만 남깁니다.
        // 이 값이 1이면 r >= 2를 나타내고, 0이면 r < 2를 나타냅니다.
        let f := shr(128, r)
        // f == 1이면 log_2 += Q192.64의 1/2
        log_2 := or(log_2, shl(63, f))
        // 2단계(즉, 공식 1.4)에 따라 r >= 2(즉, f == 1)이면 r /= 2; 그렇지 않으면 아무 작업도 수행하지 않음, 즉 3단계
        r := shr(f, r)
    }
    // 위 과정을 반복
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(62, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(61, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(60, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(59, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(58, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(57, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(56, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(55, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(54, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(53, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(52, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(51, f))
        r := shr(f, r)
    }
    assembly {
        r := shr(127, mul(r, r))
        let f := shr(128, r)
        log_2 := or(log_2, shl(50, f))
    }
```

계산된 log_2는 `Q192.64`로 표현된 $log_2{\sqrt{P}}$이며 정밀도는 $2^{-14}$입니다.

```solidity
    int256 log_sqrt10001 = log_2 * 255738958999603826347141; // 128.128 number
```

이유는 다음과 같습니다:

$$
\log_{\sqrt{1.0001}} \sqrt{P} = \frac{log_2{\sqrt{P}}}{log_2{\sqrt{1.0001}}} = log_2{\sqrt{P}}  \cdot log_{\sqrt{1.0001}}{2}
$$

여기서 `255738958999603826347141`은 $log_{\sqrt{1.0001}}{2} \cdot 2^{64}$이며, 두 `Q192.64` 숫자를 곱하면 `Q128.64` 결과가 나옵니다(오버플로 없음).

$log_2{\sqrt{P}}$의 정밀도가 $2^{-14}$이므로 `255738958999603826347141`을 곱하면 오류가 더 증폭되므로 결과가 주어진 가격에 가장 가까운 틱이 되도록 수정해야 합니다.

```solidity
    int24 tickLow = int24((log_sqrt10001 - 3402992956809132418596140100660247210) >> 128);
    int24 tickHi = int24((log_sqrt10001 + 291339464771989622907027621153398088495) >> 128);

    tick = tickLow == tickHi ? tickLow : getSqrtRatioAtTick(tickHi) <= sqrtPriceX96 ? tickHi : tickLow;
```

여기서 `3402992956809132418596140100660247210`은 `0.01000049749154292 << 128`을 나타내고, `291339464771989622907027621153398088495`는 `0.8561697375276566 << 128`을 나타냅니다.

[abdk의 이 기사](https://hackmd.io/@abdk/SkVJeHK9v)를 참조하면 $2^{-14}$의 정밀도로 가장 작은 틱 오류는 $-0.85617$이고 가장 큰 오류는 $0.0100005$입니다. 또한 이 기사는 정밀도가 $2^{-14}$과 같거나 클 때만 필요한 틱 값을 한 번에 계산할 수 있음을 이론적으로 증명합니다.

우리의 목표는 현재 조건을 만족하는 가장 큰 틱을 찾는 것입니다. 즉, 틱에 해당하는 $\sqrt{P}$가 주어진 값보다 작거나 같습니다. 따라서 보정된 tickHi가 요구 사항을 충족하면 우선적으로 사용되고, 그렇지 않으면 tickLow가 사용됩니다.

이 섹션의 참고 자료는 다음과 같습니다:

- [Solidity의 로그 계산](https://liaoph.com/logarithm-in-solidity/)
- [Solidity의 수학](https://medium.com/coinmonks/math-in-solidity-part-5-exponent-and-logarithm-9aef8515136e)
- [로그 근사 정밀도](https://hackmd.io/@abdk/SkVJeHK9v)

### TickBitmap.sol

TickBitmap은 비트맵(Bitmap)을 사용하여 틱의 초기화 상태를 저장하며 다음 메서드를 제공합니다:

- [position](#position): 틱을 기반으로 비트맵 인덱스 데이터를 반환합니다.
- [flipTick](#flipTick): 틱 상태를 뒤집습니다.
- [nextInitializedTickWithinOneWord](#nextInitializedTickWithinOneWord): 같은 그룹 내에서 다음 초기화된 틱을 찾습니다.

#### position

```solidity
/// @notice Computes the position in the mapping where the initialized bit for a tick lives
/// @param tick The tick for which to compute the position
/// @return wordPos The key in the mapping containing the word in which the bit is stored
/// @return bitPos The bit position in the word where the flag is stored
function position(int24 tick) private pure returns (int16 wordPos, uint8 bitPos) {
    wordPos = int16(tick >> 8);
    bitPos = uint8(tick % 256);
}
```

`tickSpacing`으로 나누어떨어지는 틱만 비트맵에 기록할 수 있으므로 여기서 매개변수는 $tick = \frac{tick}{tickSpacing}$입니다.

`tick`의 유형은 `int24`이며, 이진 형식으로 오른쪽에서 왼쪽으로, 낮은 곳에서 높은 곳으로 처음 8비트는 `bitPos`를 나타내고 다음 16비트는 `wordPos`를 나타냅니다. 다이어그램과 같이:

$$
\overbrace{23,...,8}^{wordPos},\overbrace{7,...,0}^{bitPos}
$$

`tick`의 비트맵 표현은 `self[wordPos] ^= 1 << bitPos`입니다.

#### flipTick

`tick`이 초기화 상태를 뒤집을 때 비트맵 값이 0이면 1로 변경해야 하고, 그렇지 않으면 0으로 변경해야 합니다. 즉, 해당 비트를 "토글"합니다.

```solidity
/// @notice Flips the initialized state for a given tick from false to true, or vice versa
/// @param self The mapping in which to flip the tick
/// @param tick The tick to flip
/// @param tickSpacing The spacing between usable ticks
function flipTick(
    mapping(int16 => uint256) storage self,
    int24 tick,
    int24 tickSpacing
) internal {
    require(tick % tickSpacing == 0); // ensure that the tick is spaced
    (int16 wordPos, uint8 bitPos) = position(tick / tickSpacing);
    uint256 mask = 1 << bitPos;
    self[wordPos] ^= mask;
}
```

먼저 `tick`에 해당하는 `wordPos`와 `bitPos`를 얻습니다. `bitPos`에 대한 최종 작업은 비트 단위 "xor"이기 때문입니다:

> 1을 임의의 값 b(0 또는 1)와 xor하면 ~b가 되고, 0을 임의의 값 b(0 또는 1)와 xor하면 b가 됩니다.

* `tick`에 해당하는 비트의 경우 마스크는 1입니다.
    - 이전 값이 1이면 `1^1=0`이므로 `tick` 상태가 "초기화됨"에서 "초기화되지 않음"으로 변경됩니다.
    - 이전 값이 0이면 `0^1=1`이므로 `tick` 상태가 "초기화되지 않음"에서 "초기화됨"으로 변경됩니다.
* `tick`에 해당하지 않는 비트의 경우 마스크는 0입니다.
    - 이전 값이 1이면 `1^0=1`이므로 상태가 변경되지 않습니다.
    - 이전 값이 0이면 `0^0=0`이므로 상태가 변경되지 않습니다.

따라서 위의 코드는 `tick` 비트를 토글하는 효과를 구현합니다.

#### nextInitializedTickWithinOneWord

`tick` 매개변수를 기반으로 비트맵에서 가장 가까운 초기화된 `tick`을 찾습니다. 찾지 못하면 그룹 내의 마지막 초기화되지 않은 틱을 반환합니다.

```solidity
/// @notice Returns the next initialized tick contained in the same word (or adjacent word) as the tick that is either
/// to the left (less than or equal to) or right (greater than) of the given tick
/// @param self The mapping in which to compute the next initialized tick
/// @param tick The starting tick
/// @param tickSpacing The spacing between usable ticks
/// @param lte Whether to search for the next initialized tick to the left (less than or equal to the starting tick)
/// @return next The next initialized or uninitialized tick up to 256 ticks away from the current tick
/// @return initialized Whether the next tick is initialized, as the function only searches within up to 256 ticks
function nextInitializedTickWithinOneWord(
    mapping(int16 => uint256) storage self,
    int24 tick,
    int24 tickSpacing,
    bool lte
) internal view returns (int24 next, bool initialized) {
    int24 compressed = tick / tickSpacing;
    if (tick < 0 && tick % tickSpacing != 0) compressed--; // round towards negative infinity

    if (lte) {
        (int16 wordPos, uint8 bitPos) = position(compressed);
        // all the 1s at or to the right of the current bitPos
        uint256 mask = (1 << bitPos) - 1 + (1 << bitPos);
        uint256 masked = self[wordPos] & mask;

        // if there are no initialized ticks to the right of or at the current tick, return rightmost in the word
        initialized = masked != 0;
        // overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
        next = initialized
            ? (compressed - int24(bitPos - BitMath.mostSignificantBit(masked))) * tickSpacing
            : (compressed - int24(bitPos)) * tickSpacing;
    } else {
        // start from the word of the next tick, since the current tick state doesn't matter
        (int16 wordPos, uint8 bitPos) = position(compressed + 1);
        // all the 1s at or to the left of the bitPos
        uint256 mask = ~((1 << bitPos) - 1);
        uint256 masked = self[wordPos] & mask;

        // if there are no initialized ticks to the left of the current tick, return leftmost in the word
        initialized = masked != 0;
        // overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
        next = initialized
            ? (compressed + 1 + int24(BitMath.leastSignificantBit(masked) - bitPos)) * tickSpacing
            : (compressed + 1 + int24(type(uint8).max - bitPos)) * tickSpacing;
    }
}
```

`lte == true`, 즉 현재 `tick`보다 작거나 같은 값을 찾는 경우 문제는 다음과 같이 변환됩니다: 하위 비트에 `1`이 있는지 찾습니다.

```solidity
if (lte) {
    (int16 wordPos, uint8 bitPos) = position(compressed);
    // all the 1s at or to the right of the current bitPos
    uint256 mask = (1 << bitPos) - 1 + (1 << bitPos);
    uint256 masked = self[wordPos] & mask;
}
```

여기서 mask 값은 `bitPos`보다 작거나 같은 모든 비트를 1로 설정합니다. 예를 들어 `bitPos = 7`이면 `mask in binary = 1111111`입니다. masked는 `bitPos`보다 작거나 같은 비트맵 값을 유지합니다. 예를 들어 `self[wordPos] = 110101011`이면 `masked = 110101011 & 1111111 = 000101011`입니다.

`masked != 0`이면 현재 방향(`lte`)의 해당 `wordPos`에 초기화된 `ticks`가 있음을 나타냅니다. 그렇지 않으면 그 방향의 모든 틱이 초기화되지 않았음을 의미합니다.

```solidity
// if there are no initialized ticks to the right of or at the current tick, return rightmost in the word
initialized = masked != 0;
```

초기화된 `ticks`가 있는 경우 `masked`에서 가장 높은 1의 비트를 찾습니다. 그렇지 않으면 현재 방향의 마지막 `tick`을 반환합니다.
```solidity
// overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
next = initialized
    ? (compressed - int24(bitPos - BitMath.mostSignificantBit(masked))) * tickSpacing
    : (compressed - int24(bitPos)) * tickSpacing;
```

`BitMath.mostSignificantBit(masked)`는 이진 검색 방법을 사용하여 `masked`에서 가장 높은 1의 비트를 찾습니다. 이 알고리즘에 대한 자세한 설명은 [TickMath 로그 계산](#정수-부분) 섹션을 참조하십시오. 간단히 말해서 `mostSignificantBit(masked)`는 다음을 만족하는 숫자 `n`을 반환합니다:

$$
2^n \leq masked < 2^{n+1}
$$

예를 들어 `masked = 000101011`인 경우, 가장 높은 1의 비트가 5번째 위치(0부터 시작)에 있으므로 `mostSignificantBit`는 5를 반환합니다: 000`1`01011.

`compressed - int24(bitPos)`는 현재 `wordPos`의 첫 번째 `tick`을 나타내고, `compressed - int24(bitPos - BitMath.mostSignificantBit(masked))`는 해당 가장 높은 `bitPos`에 해당하는 `tick`을 나타냅니다. `* tickSpacing`은 비트맵에 저장할 때 `tick`이 `tickSpacing`으로 나누어졌으므로 원래 `tick` 값으로 복원합니다.

마찬가지로 현재 `tick`보다 큰 방향에서 첫 번째 초기화된 `tick`을 검색할 때 방법은 유사하지만, `mostSignificantBit`를 `leastSignificantBit`로 대체해야 합니다. 즉, `tick` 비트(제외)부터 시작하여 위쪽으로 1인 첫 번째 `bitPos` 비트를 검색합니다.

```solidity
else {
    // start from the word of the next tick, since the current tick state doesn't matter
    (int16 wordPos, uint8 bitPos) = position(compressed + 1);
    // all the 1s at or to the left of the bitPos
    uint256 mask = ~((1 << bitPos) - 1);
    uint256 masked = self[wordPos] & mask;

    // if there are no initialized ticks to the left of the current tick, return leftmost in the word
    initialized = masked != 0;
    // overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
    next = initialized
        ? (compressed + 1 + int24(BitMath.leastSignificantBit(masked) - bitPos)) * tickSpacing
        : (compressed + 1 + int24(type(uint8).max - bitPos)) * tickSpacing;
}
```

### Position.sol

`Position.sol`은 포지션 관련 정보를 관리하며 다음 메서드를 포함합니다:

- [get](#get): 포지션 객체를 가져옵니다
- [update](#update1): 포지션을 업데이트합니다

포지션은 "소유자" `owner`, "하한 틱" `tickLower`, "상한 틱" `tickUpper`에 의해 고유하게 결정될 수 있습니다. 동일한 거래 쌍 풀에 대해 각 사용자는 서로 다른 가격 범위를 가진 여러 포지션을 생성할 수 있지만 동일한 가격 범위를 가진 포지션은 하나만 생성할 수 있습니다. 다른 사용자는 동일한 가격 범위로 포지션을 생성할 수 있습니다.

#### get

`owner`, `tickLower`, `tickUpper`를 기반으로 포지션 객체를 반환합니다:

```solidity
/// @notice Returns the Info struct of a position, given an owner and position boundaries
/// @param self The mapping containing all user positions
/// @param owner The address of the position owner
/// @param tickLower The lower tick boundary of the position
/// @param tickUpper The upper tick boundary of the position
/// @return position The position info struct of the given owners' position
function get(
    mapping(bytes32 => Info) storage self,
    address owner,
    int24 tickLower,
    int24 tickUpper
) internal view returns (Position.Info storage position) {
    position = self[keccak256(abi.encodePacked(owner, tickLower, tickUpper))];
}
```

#### update

포지션의 유동성 및 출금 가능한 토큰을 업데이트합니다. 참고로 이 메서드는 `mint` 및 `burn` 중에만 트리거되며 `swap`은 포지션 정보를 업데이트하지 않습니다.

```solidity
/// @notice Credits accumulated fees to a user's position
/// @param self The individual position to update
/// @param liquidityDelta The change in pool liquidity as a result of the position update
/// @param feeGrowthInside0X128 The all-time fee growth in token0, per unit of liquidity, inside the position's tick boundaries
/// @param feeGrowthInside1X128 The all-time fee growth in token1, per unit of liquidity, inside the position's tick boundaries
function update(
    Info storage self,
    int128 liquidityDelta,
    uint256 feeGrowthInside0X128,
    uint256 feeGrowthInside1X128
) internal {
    Info memory _self = self;

    uint128 liquidityNext;
    if (liquidityDelta == 0) {
        require(_self.liquidity > 0, 'NP'); // disallow pokes for 0 liquidity positions
        liquidityNext = _self.liquidity;
    } else {
        liquidityNext = LiquidityMath.addDelta(_self.liquidity, liquidityDelta);
    }
```

유동성 업데이트:

- `mint`인 경우 `liquidityDelta > 0`
- `burn`인 경우 `liquidityDelta < 0`

```solidity
    // calculate accumulated fees
    uint128 tokensOwed0 =
        uint128(
            FullMath.mulDiv(
                feeGrowthInside0X128 - _self.feeGrowthInside0LastX128,
                _self.liquidity,
                FixedPoint128.Q128
            )
        );
    uint128 tokensOwed1 =
        uint128(
            FullMath.mulDiv(
                feeGrowthInside1X128 - _self.feeGrowthInside1LastX128,
                _self.liquidity,
                FixedPoint128.Q128
            )
        );
```

마지막 업데이트 이후 포지션의 틱 경계 내 유동성 당 수수료 증가를 기반으로 `token0` 및 `token1`에 대한 수취 가능한 수수료를 계산합니다.

```solidity
    // update the position
    if (liquidityDelta != 0) self.liquidity = liquidityNext;
    self.feeGrowthInside0LastX128 = feeGrowthInside0X128;
    self.feeGrowthInside1LastX128 = feeGrowthInside1X128;
    if (tokensOwed0 > 0 || tokensOwed1 > 0) {
        // overflow is acceptable, have to withdraw before you hit type(uint128).max fees
        self.tokensOwed0 += tokensOwed0;
        self.tokensOwed1 += tokensOwed1;
    }
}
```

포지션 유동성, 이번 기간의 유동성 당 수수료, 출금 가능한 토큰 양을 업데이트합니다.

### SwapMath.sol

#### computeSwapStep

SwapMath는 단일 메서드 `computeSwapStep`만 가지고 있으며, 이는 단일 스왑 단계에 대한 입력 및 출력을 계산합니다.

이 메서드는 본질적으로 백서의 섹션 6.2.3에 설명된 개념을 구현합니다: 단일 틱 내에서 거래를 수행합니다. 이 시나리오에서 유동성 $L$은 일정하게 유지되지만 가격 $\sqrt{P}$만 변경됩니다.

매개변수는 다음과 같습니다:

- `sqrtRatioCurrentX96`: 현재 가격
- `sqrtRatioTargetX96`: 목표 가격
- `liquidity`: 사용 가능 유동성
- `amountRemaining`: 남은 입력 토큰
- `feePips`: 거래 수수료

반환 값:

- `sqrtRatioNextX96`: 스왑 후 가격
- `amountIn`: 이번 스왑에서 소비된 입력 토큰
- `amountOut`: 이번 스왑에서 생성된 출력 토큰
- `feeAmount`: 거래 수수료 (프로토콜 수수료 포함)

```solidity
function computeSwapStep(
    uint160 sqrtRatioCurrentX96,
    uint160 sqrtRatioTargetX96,
    uint128 liquidity,
    int256 amountRemaining,
    uint24 feePips
)
    internal
    pure
    returns (
        uint160 sqrtRatioNextX96,
        uint256 amountIn,
        uint256 amountOut,
        uint256 feeAmount
    )
{
    bool zeroForOne = sqrtRatioCurrentX96 >= sqrtRatioTargetX96;
    bool exactIn = amountRemaining >= 0;
```

[swap](#swap) 섹션에서 스왑 작업의 경우 `zeroForOne` 및 `exactIn`의 값에 따라 네 가지 조합이 있다고 언급했습니다:

| zeroForOne | exactInput | swap |
| --- | --- | --- |
| true | true | 고정된 양의 `token0` 입력, 최대 양의 `token1` 출력 |
| true | false | 최소 양의 `token0` 입력, 고정된 양의 `token1` 출력 |
| false | true | 고정된 양의 `token1` 입력, 최대 양의 `token0` 출력 |
| false | false | 최소 양의 `token1` 입력, 고정된 양의 `token0` 출력 |

```solidity
if (exactIn) {
    uint256 amountRemainingLessFee = FullMath.mulDiv(uint256(amountRemaining), 1e6 - feePips, 1e6);
    amountIn = zeroForOne
        ? SqrtPriceMath.getAmount0Delta(sqrtRatioTargetX96, sqrtRatioCurrentX96, liquidity, true)
        : SqrtPriceMath.getAmount1Delta(sqrtRatioCurrentX96, sqrtRatioTargetX96, liquidity, true);
    if (amountRemainingLessFee >= amountIn) sqrtRatioNextX96 = sqrtRatioTargetX96;
    else
        sqrtRatioNextX96 = SqrtPriceMath.getNextSqrtPriceFromInput(
            sqrtRatioCurrentX96,
            liquidity,
            amountRemainingLessFee,
            zeroForOne
        );
}
```

입력 금액이 고정된 경우, 입력 토큰에서 거래 수수료를 공제해야 합니다.

`feePips`의 단위는 베이시스 포인트의 100분의 1, 즉 $\frac{1}{10^6}$이므로 다음 공식에 따라 수수료를 공제합니다:

$$
amountRemaining \cdot (1 - \frac{feePips}{10^6})
$$

[getAmount0Delta](#getAmount0Delta) 또는 [getAmount1Delta](#getAmount1Delta)를 사용하여, 현재 가격, 목표 가격 및 사용 가능 유동성을 기반으로 필요한 입력 토큰 양 `amountIn`을 계산합니다:

- `token0`에서 `token1`으로 스왑하는 경우 입력 토큰은 `token0`이므로 `amount0`을 계산합니다.
- `token1`에서 `token0`으로 스왑하는 경우 입력 토큰은 `token1`이므로 `amount1`을 계산합니다.

참고로 마지막 매개변수 `roundUp`은 `amountIn`을 계산할 때 `roundUp = true`로 설정됩니다. 이는 컨트랙트가 더 많이 받을 수는 있어도 적게 받을 수는 없으며, 그렇지 않으면 컨트랙트가 자금을 잃게 됨을 의미합니다. 마찬가지로 `amountOut`을 계산할 때는 `roundUp = false`로 설정되어 컨트랙트가 덜 지불할 수는 있어도 더 지불할 수는 없습니다.

- 사용 가능한 토큰 양이 `amountIn`보다 크면 단계 거래를 완료할 수 있음을 나타내므로 스왑 후 가격은 목표 가격과 같습니다.
- 그렇지 않으면 현재 가격, 사용 가능 유동성 및 사용 가능 토큰 `amountRemainingLessFee`를 기반으로 스왑 후 가격을 계산합니다. 구체적인 계산 방법은 [SqrtPriceMath.getNextSqrtPriceFromInput](#getNextSqrtPriceFromInput)에서 소개합니다.

```solidity
else {
    amountOut = zeroForOne
        ? SqrtPriceMath.getAmount1Delta(sqrtRatioTargetX96, sqrtRatioCurrentX96, liquidity, false)
        : SqrtPriceMath.getAmount0Delta(sqrtRatioCurrentX96, sqrtRatioTargetX96, liquidity, false);
    if (uint256(-amountRemaining) >= amountOut) sqrtRatioNextX96 = sqrtRatioTargetX96;
    else
        sqrtRatioNextX96 = SqrtPriceMath.getNextSqrtPriceFromOutput(
            sqrtRatioCurrentX96,
            liquidity,
            uint256(-amountRemaining),
            zeroForOne
        );
}
```

입력 금액이 고정되지 않은 경우, 즉 출력 금액이 고정된 경우 `amountRemaining`을 출력 토큰 양으로 사용합니다:

- `token0`에서 `token1`으로 스왑하는 경우 출력 토큰은 `token1`이므로 `amount1`을 계산합니다.
- `token1`에서 `token0`으로 스왑하는 경우 출력 토큰은 `token0`이므로 `amount0`을 계산합니다.

먼저 현재 가격, 목표 가격 및 사용 가능 유동성을 기반으로 생산 가능한 출력 토큰 양 `amountOut`을 계산합니다. 여기서 위와 반대로 `token0`에서 `token1`으로 스왑하는 경우 `SqrtPriceMath.getAmount1Delta` 메서드를 사용하여 `token1` 양을 계산해야 함에 유의하십시오.

고정된 출력 토큰을 지정할 때 전달된 `amountRemaining`은 음수입니다:

- `amountOut`의 절대값이 필요한 출력 토큰 양보다 작으면 전체 스왑을 실행할 수 있음을 의미하므로 스왑 후 가격은 목표 가격과 같습니다.
- 그렇지 않으면 사용 가능한 출력 `amountRemaining`을 기반으로 스왑 후 가격을 계산합니다.

```solidity
    bool max = sqrtRatioTargetX96 == sqrtRatioNextX96;
```

스왑 후 가격이 목표 가격과 같으면 스왑이 완료되었음을 의미합니다.

```solidity
// get the input/output amounts
if (zeroForOne) {
    amountIn = max && exactIn
        ? amountIn
        : SqrtPriceMath.getAmount0Delta(sqrtRatioNextX96, sqrtRatioCurrentX96, liquidity, true);
    amountOut = max && !exactIn
        ? amountOut
        : SqrtPriceMath.getAmount1Delta(sqrtRatioNextX96, sqrtRatioCurrentX96, liquidity, false);
} else {
    amountIn = max && exactIn
        ? amountIn
        : SqrtPriceMath.getAmount1Delta(sqrtRatioCurrentX96, sqrtRatioNextX96, liquidity, true);
    amountOut = max && !exactIn
        ? amountOut
        : SqrtPriceMath.getAmount0Delta(sqrtRatioCurrentX96, sqrtRatioNextX96, liquidity, false);
}
```

이번 스왑에 필요한 입력 `amountIn`과 생성된 출력 `amountOut`을 계산합니다:

- `token0`에서 `token1`으로 스왑하는 경우
    - 필요한 입력 `amountIn`
        - 스왑이 완료되고 입력이 고정된 경우 필요한 입력은 `amountIn`입니다.
        - 그렇지 않은 경우(불완전한 스왑 또는 고정된 출력), `getAmount0Delta`를 사용하여 가격 범위와 유동성을 기반으로 필요한 입력을 계산합니다.
    - 생성된 출력 `amountOut`
        - 스왑이 완료되고 출력이 고정된 경우 생성된 출력은 `amountOut`입니다.
        - 그렇지 않은 경우(불완전한 스왑 또는 고정된 입력), `getAmount1Delta`를 사용하여 가격 범위와 유동성을 기반으로 생성된 출력을 계산합니다.
- `token1`에서 `token0`으로 스왑하는 경우
    - 필요한 입력 `amountIn`
        - 스왑이 완료되고 입력이 고정된 경우 필요한 입력은 `amountIn`입니다.
        - 그렇지 않은 경우(불완전한 스왑 또는 고정된 출력), `getAmount1Delta`를 사용하여 가격 범위와 유동성을 기반으로 필요한 입력을 계산합니다.
    - 생성된 출력 `amountOut`
        - 스왑이 완료되고 출력이 고정된 경우 생성된 출력은 `amountOut`입니다.
        - 그렇지 않은 경우(불완전한 스왑 또는 고정된 입력), `getAmount0Delta`를 사용하여 가격 범위와 유동성을 기반으로 생성된 출력을 계산합니다.

```solidity
// cap the output amount to not exceed the remaining output amount
if (!exactIn && amountOut > uint256(-amountRemaining)) {
    amountOut = uint256(-amountRemaining);
}
```

생성된 출력이 지정된 출력을 초과하지 않도록 합니다.

```solidity
if (exactIn && sqrtRatioNextX96 != sqrtRatioTargetX96) {
    // we didn't reach the target, so take the remainder of the maximum input as fee
    feeAmount = uint256(amountRemaining) - amountIn;
} else {
    feeAmount = FullMath.mulDivRoundingUp(amountIn, feePips, 1e6 - feePips);
}
```

거래 수수료(프로토콜 수수료 포함)를 계산합니다:

- 입력이 고정되어 있고 목표 가격에 도달하지 못한 경우, 즉 마지막 스왑에 해당하면 원래 입력 토큰 `amountRemaining`에서 이번 스왑에 필요한 입력 `amountIn`을 뺀 값이 수수료입니다.
- 그렇지 않으면 `amountIn`을 기반으로 수수료를 계산해야 합니다. 위에서 언급한 마지막 스왑 외에, 여기서 `amountIn` 및 `amountOut`에는 수수료가 포함되지 않으므로 수수료 계산 시 `1e6`이 아닌 `1e6 - feePips`로 나누어야 합니다.


### SqrtPriceMath.sol

#### getNextSqrtPriceFromAmount0RoundingUp

현재 가격, `liquidity`, $\Delta{x}$를 기반으로 목표 가격을 계산합니다.

백서 공식 6.16에 따르면:

$$
\Delta{x} = \Delta{\frac{1}{\sqrt{P}}} \cdot L
$$

$\sqrt{P_a} > \sqrt{P_b}$라고 가정하면 $x_a < x_b$:

$$
\Delta{x} = x_b - x_a
$$

$\sqrt{P_a}$가 주어지고 $\sqrt{P_b}$를 계산하는 경우:

$$
\frac{1}{\sqrt{P_b}} = \frac{\Delta{x}}{L} + \frac{1}{\sqrt{P_a}} = \frac{1}{L} \cdot (\Delta{x} + \frac{L}{\sqrt{P_a}}) \quad \text{(1.1)}
$$

$$
\sqrt{P_b} = \frac{L}{\Delta{x} + \frac{L}{\sqrt{P_a}}} \quad \text{(1.2)}
$$

$$
{\sqrt{P_b}} = \frac{L \cdot \sqrt{P_a}}{L + \Delta{x} \cdot \sqrt{P_a}}  \quad \text{(1.3)}
$$

$\sqrt{P_b}$가 주어지고 $\sqrt{P_a}$를 계산하는 경우:

$$
\frac{1}{\sqrt{P_a}} = \frac{1}{\sqrt{P_b}} - \frac{\Delta{x}}{L}  \quad \text{(1.4)}
$$

$$
{\sqrt{P_a}} = \frac{L \cdot \sqrt{P_b}}{L - \Delta{x} \cdot \sqrt{P_b}}  \quad \text{(1.5)}
$$

```solidity
/// @notice Gets the next sqrt price given a delta of token0
/// @dev Always rounds up, because in the exact output case (increasing price) we need to move the price at least
/// far enough to get the desired output amount, and in the exact input case (decreasing price) we need to move the
/// price less in order to not send too much output.
/// The most precise formula for this is liquidity * sqrtPX96 / (liquidity +- amount * sqrtPX96),
/// if this is impossible because of overflow, we calculate liquidity / (liquidity / sqrtPX96 +- amount).
/// @param sqrtPX96 The starting price, i.e. before accounting for the token0 delta
/// @param liquidity The amount of usable liquidity
/// @param amount How much of token0 to add or remove from virtual reserves
/// @param add Whether to add or remove the amount of token0
/// @return The price after adding or removing amount, depending on add
function getNextSqrtPriceFromAmount0RoundingUp(
    uint160 sqrtPX96,
    uint128 liquidity,
    uint256 amount,
    bool add
) internal pure returns (uint160) {
    // we short circuit amount == 0 because the result is otherwise not guaranteed to equal the input price
    if (amount == 0) return sqrtPX96;
    uint256 numerator1 = uint256(liquidity) << FixedPoint96.RESOLUTION;

    if (add) {
        uint256 product;
        if ((product = amount * sqrtPX96) / amount == sqrtPX96) {
            uint256 denominator = numerator1 + product;
            if (denominator >= numerator1)
                // always fits in 160 bits
                return uint160(FullMath.mulDivRoundingUp(numerator1, sqrtPX96, denominator));
        }

        return uint160(UnsafeMath.divRoundingUp(numerator1, (numerator1 / sqrtPX96).add(amount)));
    } else {
        uint256 product;
        // if the product overflows, we know the denominator underflows
        // in addition, we must check that the denominator does not underflow
        require((product = amount * sqrtPX96) / amount == sqrtPX96 && numerator1 > product);
        uint256 denominator = numerator1 - product;
        return FullMath.mulDivRoundingUp(numerator1, sqrtPX96, denominator).toUint160();
    }
}
```

- `add = true`일 때, 즉 $\sqrt{P_a}$가 주어지고 $\sqrt{P_b}$를 계산하는 경우
    - $\Delta{x} \cdot \sqrt{P_a}$가 오버플로하지 않으면 공식 1.3을 사용하여 $\sqrt{P_b}$를 계산합니다. 이 방법이 정밀도가 가장 높기 때문입니다.
    - 그렇지 않으면 공식 1.2를 사용하여 $\sqrt{P_b}$를 계산합니다.
- `add = false`일 때, 즉 $\sqrt{P_b}$가 주어지고 $\sqrt{P_a}$를 계산하는 경우 공식 1.5를 사용합니다.

#### getNextSqrtPriceFromAmount1RoundingDown

현재 가격, `liquidity`, $\Delta{y}$를 기반으로 목표 가격을 계산합니다.

백서 공식 6.13에 따르면:

$$
\Delta{\sqrt{P}} = \frac{\Delta{y}}{L} \quad \text{(6.13)}
$$

$\sqrt{P_a} > \sqrt{P_b}$라고 가정하면 $y_a > y_b$:

$$
\Delta{y} = y_a - y_b
$$

$\sqrt{P_b}$가 주어지고 $\sqrt{P_a}$를 계산하는 경우:

$$
\sqrt{P_a} = \sqrt{P_b} + \frac{\Delta{y}}{L} \quad \text{(1.6)}
$$

$\sqrt{P_a}$가 주어지고 $\sqrt{P_b}$를 계산하는 경우:

$$
\sqrt{P_b} = \sqrt{P_a} - \frac{\Delta{y}}{L} \quad \text{(1.7)}
$$

```solidity
/// @notice Gets the next sqrt price given a delta of token1
/// @dev Always rounds down, because in the exact output case (decreasing price) we need to move the price at least
/// far enough to get the desired output amount, and in the exact input case (increasing price) we need to move the
/// price less in order to not send too much output.
/// The formula we compute is within <1 wei of the lossless version: sqrtPX96 +- amount / liquidity
/// @param sqrtPX96 The starting price, i.e., before accounting for the token1 delta
/// @param liquidity The amount of usable liquidity
/// @param amount How much of token1 to add, or remove, from virtual reserves
/// @param add Whether to add, or remove, the amount of token1
/// @return The price after adding or removing `amount`
function getNextSqrtPriceFromAmount1RoundingDown(
    uint160 sqrtPX96,
    uint128 liquidity,
    uint256 amount,
    bool add
) internal pure returns (uint160) {
    // if we're adding (subtracting), rounding down requires rounding the quotient down (up)
    // in both cases, avoid a mulDiv for most inputs
    if (add) {
        uint256 quotient =
            (
                amount <= type(uint160).max
                    ? (amount << FixedPoint96.RESOLUTION) / liquidity
                    : FullMath.mulDiv(amount, FixedPoint96.Q96, liquidity)
            );

        return uint256(sqrtPX96).add(quotient).toUint160();
    } else {
        uint256 quotient =
            (
                amount <= type(uint160).max
                    ? UnsafeMath.divRoundingUp(amount << FixedPoint96.RESOLUTION, liquidity)
                    : FullMath.mulDivRoundingUp(amount, FixedPoint96.Q96, liquidity)
            );

        require(sqrtPX96 > quotient);
        // always fits 160 bits
        return uint160(sqrtPX96 - quotient);
    }
}
```

* `add = true`일 때, 즉 공식 1.6에 따라 $\sqrt{P_b}$를 기반으로 $\sqrt{P_a}$를 계산합니다.
* `add = false`일 때, 공식 1.7에 따라 $\sqrt{P_a}$를 기반으로 $\sqrt{P_b}$를 계산합니다.

#### getNextSqrtPriceFromInput

입력 토큰을 기반으로 다음 가격을 계산합니다. 즉, `amountIn`만큼의 토큰을 추가한 후의 가격입니다:

```solidity
/// @notice Gets the next sqrt price given an input amount of token0 or token1
/// @dev Throws if price or liquidity are 0, or if the next price is out of bounds
/// @param sqrtPX96 The starting price, i.e., before accounting for the input amount
/// @param liquidity The amount of usable liquidity
/// @param amountIn How much of token0, or token1, is being swapped in
/// @param zeroForOne Whether the amount in is token0 or token1
/// @return sqrtQX96 The price after adding the input amount to token0 or token1
function getNextSqrtPriceFromInput(
    uint160 sqrtPX96,
    uint128 liquidity,
    uint256 amountIn,
    bool zeroForOne
) internal pure returns (uint160 sqrtQX96) {
    require(sqrtPX96 > 0);
    require(liquidity > 0);

    // round to make sure that we don't pass the target price
    return
        zeroForOne
            ? getNextSqrtPriceFromAmount0RoundingUp(sqrtPX96, liquidity, amountIn, true)
            : getNextSqrtPriceFromAmount1RoundingDown(sqrtPX96, liquidity, amountIn, true);
}
```

#### getNextSqrtPriceFromOutput

출력 토큰을 기반으로 다음 가격을 계산합니다. 즉, `amountOut`만큼의 토큰을 제거한 후의 가격입니다:

```solidity
/// @notice Gets the next sqrt price given an output amount of token0 or token1
/// @dev Throws if price or liquidity are 0 or the next price is out of bounds
/// @param sqrtPX96 The starting price before accounting for the output amount
/// @param liquidity The amount of usable liquidity
/// @param amountOut How much of token0, or token1, is being swapped out
/// @param zeroForOne Whether the amount out is token0 or token1
/// @return sqrtQX96 The price after removing the output amount of token0 or token1
function getNextSqrtPriceFromOutput(
    uint160 sqrtPX96,
    uint128 liquidity,
    uint256 amountOut,
    bool zeroForOne
) internal pure returns (uint160 sqrtQX96) {
    require(sqrtPX96 > 0);
    require(liquidity > 0);

    // round to make sure that we pass the target price
    return
        zeroForOne
            ? getNextSqrtPriceFromAmount1RoundingDown(sqrtPX96, liquidity, amountOut, false)
            : getNextSqrtPriceFromAmount0RoundingUp(sqrtPX96, liquidity, amountOut, false);
}
```

#### getAmount0Delta

이 메서드는 백서의 공식 6.16을 계산합니다:

$$
\Delta{x} = \Delta{\frac{1}{\sqrt{P}}} \cdot L
$$

확장하면:

$$
amount0 = x_b - x_a = L \cdot (\frac{1}{\sqrt{P_b}} - \frac{1}{\sqrt{P_a}}) = L \cdot (\frac{\sqrt{P_a} - \sqrt{P_b}}{\sqrt{P_a} \cdot \sqrt{P_b}})
$$

```solidity
/// @notice Gets the amount0 delta between two prices
/// @dev Calculates liquidity / sqrt(lower) - liquidity / sqrt(upper),
/// i.e. liquidity * (sqrt(upper) - sqrt(lower)) / (sqrt(upper) * sqrt(lower))
/// @param sqrtRatioAX96 A sqrt price
/// @param sqrtRatioBX96 Another sqrt price
/// @param liquidity The amount of usable liquidity
/// @param roundUp Whether to round the amount up or down
/// @return amount0 Amount of token0 required to cover a position of size liquidity between the two passed prices
function getAmount0Delta(
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint128 liquidity,
    bool roundUp
) internal pure returns (uint256 amount0) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

    uint256 numerator1 = uint256(liquidity) << FixedPoint96.RESOLUTION;
    uint256 numerator2 = sqrtRatioBX96 - sqrtRatioAX96;

    require(sqrtRatioAX96 > 0);

    return
        roundUp
            ? UnsafeMath.divRoundingUp(
                FullMath.mulDivRoundingUp(numerator1, numerator2, sqrtRatioBX96),
                sqrtRatioAX96
            )
            : FullMath.mulDiv(numerator1, numerator2, sqrtRatioBX96) / sqrtRatioAX96;
}
```

#### getAmount1Delta

마찬가지로 이 메서드는 백서의 공식 6.7을 계산합니다:

$$
L = \frac{\Delta{Y}}{\Delta{\sqrt{P}}}
$$

확장하면:

$$
amount1 = y_b - y_a = L \cdot \Delta{\sqrt{P}} = L \cdot (\sqrt{P_b} - \sqrt{P_a})
$$

```solidity
/// @notice Gets the amount1 delta between two prices
/// @dev Calculates liquidity * (sqrt(upper) - sqrt(lower))
/// @param sqrtRatioAX96 A sqrt price
/// @param sqrtRatioBX96 Another sqrt price
/// @param liquidity The amount of usable liquidity
/// @param roundUp Whether to round the amount up, or down
/// @return amount1 Amount of token1 required to cover a position of size liquidity between the two passed prices
function getAmount1Delta(
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint128 liquidity,
    bool roundUp
) internal pure returns (uint256 amount1) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

    return
        roundUp
            ? FullMath.mulDivRoundingUp(liquidity, sqrtRatioBX96 - sqrtRatioAX96, FixedPoint96.Q96)
            : FullMath.mulDiv(liquidity, sqrtRatioBX96 - sqrtRatioAX96, FixedPoint96.Q96);
```

### Oracle.sol

이 컨트랙트는 오라클과 관련된 메서드를 정의합니다.

쌍(pair) 컨트랙트를 생성할 때 기본적으로 관측 데이터를 저장하기 위해 길이 1의 배열이 초기화됩니다. 모든 사용자는 `UniswapV3Pool.sol`의 [increaseObservationCardinalityNext](#increaseObservationCardinalityNext) 메서드를 호출하여 오라클의 관측 공간을 확장할 수 있지만, 공간 확장에 따른 비용은 부담해야 합니다.

오라클 관측에는 다음이 포함됩니다:

* `blockTimestamp`: 관측이 기록된 시간
* `tickCumulative`: 관측 시간 기준 누적 `tick`
* `secondsPerLiquidityCumulativeX128`: 관측 시간 기준 유동성 당 누적 초
* `initialized`: 관측 초기화 여부

관측 구조체는 다음과 같이 정의됩니다:

```solidity
struct Observation {
    // the block timestamp of the observation
    uint32 blockTimestamp;
    // the tick accumulator, i.e. tick * time elapsed since the pool was first initialized
    int56 tickCumulative;
    // the seconds per liquidity, i.e. seconds elapsed / max(1, liquidity) since the pool was first initialized
    uint160 secondsPerLiquidityCumulativeX128;
    // whether or not the observation is initialized
    bool initialized;
}
```

이 컨트랙트에는 다음 메서드가 포함됩니다:

* [transform](#transform): 이전을 기반으로 새 관측 객체를 반환합니다(단, 관측을 쓰지는 않음).
* [initialize](#initialize2): 관측 배열을 초기화합니다.
* [write](#write): 관측을 씁니다.
* [grow](#grow): 오라클 관측 공간을 확장합니다.
* [lte](#lte): 두 타임스탬프의 크기를 비교합니다.
* [binarySearch](#binarySearch): 지정된 시간에 관측 경계를 이진 검색합니다.
* [getSurroundingObservations](#getSurroundingObservations): 지정된 시간에 관측 경계를 가져옵니다.
* [observeSingle](#observeSingle): 지정된 시간에 관측 데이터를 검색합니다.
* [observe](#observe): 지정된 시간에 관측 데이터를 일괄 검색합니다.

#### transform

이전 관측을 기반으로 임시 관측 객체를 반환하지만 관측을 쓰지는 않습니다.

```solidity
function transform(
    Observation memory last,
    uint32 blockTimestamp,
    int24 tick,
    uint128 liquidity
) private pure returns (Observation memory) {
    uint32 delta = blockTimestamp - last.blockTimestamp;
    return
        Observation({
            blockTimestamp: blockTimestamp,
            tickCumulative: last.tickCumulative + int56(tick) * delta,
            secondsPerLiquidityCumulativeX128: last.secondsPerLiquidityCumulativeX128 +
                ((uint160(delta) << 128) / (liquidity > 0 ? liquidity : 1)),
            initialized: true
        });
}
```

먼저 현재 시간과 마지막 관측 시간 사이의 시간 차이를 계산합니다: `delta = blockTimestamp - last.blockTimestamp`.

그런 다음 주로 `tickCumulative` 및 `secondsPerLiquidityCumulativeX128`을 계산합니다.

여기서: `tickCumulative: last.tickCumulative + int56(tick) * delta`.

백서 공식 5.3-5.5에 따르면:

$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{\sum_{i=t_1}^{t_2} \log_{1.0001}(P_i)}{t_2 - t_1} \quad \text{(5.3)}
$$

$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{a_{t_2} - a_{t_1}}{t_2 - t_1} \quad \text{(5.4)}
$$

$$
P_{t_1,t_2} = 1.0001^{\frac{a_{t_2} - a_{t_1}}{t_2 - t_1}} \quad \text{(5.5)}
$$

여기서 저장된 `tickCumulative`는 $a_{t_n}$이며, 다음 공식에 해당합니다:

$$
tickCumulative = \sum_{i=0}^{t_n} \log_{1.0001}(P_i)
$$

마찬가지로 유동성 당 누적 초 `secondsPerLiquidityCumulative`는 다음과 같습니다:

$$
secondsPerLiquidityCumulative = \sum_{i=0}^{n} \frac{t_i}{L_i}
$$

#### initialize

오라클 저장 공간을 초기화하고 첫 번째 `slot`을 설정합니다.

```solidity
/// @notice Initialize the oracle array by writing the first slot. Called once for the lifecycle of the observations array
/// @param self The stored oracle array
/// @param time The time of the oracle initialization, via block.timestamp truncated to uint32
/// @return cardinality The number of populated elements in the oracle array
/// @return cardinalityNext The new length of the oracle array, independent of population
function initialize(Observation[65535] storage self, uint32 time)
    internal
    returns (uint16 cardinality, uint16 cardinalityNext)
{
    self[0] = Observation({
        blockTimestamp: time,
        tickCumulative: 0,
        secondsPerLiquidityCumulativeX128: 0,
        initialized: true
    });
    return (1, 1);
}
```

`cardinality` 및 `cardinalityNext`의 기본 반환 값은 모두 1이며, 이는 하나의 관측 점 데이터만 저장할 수 있음을 의미합니다.

이 메서드는 한 번만 호출할 수 있으며 실제로는 `UniswapV3Pool.sol`의 [initialize](#initialize) 메서드에서 호출됩니다:

```solidity
(uint16 cardinality, uint16 cardinalityNext) = observations.initialize(_blockTimestamp());
```

#### write

단일 관측 점 데이터를 씁니다. 이전 설명에 따르면 쓰기 작업은 `UniswapV3Pool`에서 `mint`, `burn` 및 `swap` 작업이 발생할 때만 트리거될 수 있습니다.

오라클 관측 점 배열은 순환 목록으로 볼 수 있으며, 쓰기 가능한 공간은 `cardinality` 및 `cardinalityNext`에 의해 결정됩니다. 배열 공간이 가득 차면 0번째 위치부터 덮어쓰기를 계속합니다.

이 메서드의 매개변수는 다음과 같습니다:

* `self`: 오라클 관측 점 배열
* `index`: 마지막으로 쓴 관측 점의 인덱스, 0부터 시작
* `blockTimestamp`: 추가할 관측 점의 시간
* `tick`: 추가할 관측 점의 `tick`
* `liquidity`: 관측 점 시점의 전역 사용 가능 유동성
* `cardinality`: 관측 점 배열의 현재 길이 (쓰기 가능한 공간)
* `cardinalityNext`: 관측 점 배열의 최신 길이 (확장됨)

```solidity
function write(
    Observation[65535] storage self,
    uint16 index,
    uint32 blockTimestamp,
    int24 tick,
    uint128 liquidity,
    uint16 cardinality,
    uint16 cardinalityNext
) internal returns (uint16 indexUpdated, uint16 cardinalityUpdated) {
    Observation memory last = self[index];

    // early return if we've already written an observation this block
    if (last.blockTimestamp == blockTimestamp) return (index, cardinality);
```

새 관측 점의 시간이 마지막 관측 점 시간과 같으면 바로 반환하여 블록당 하나의 관측 점만 기록되도록 합니다.

```solidity
    // if the conditions are right, we can bump the cardinality
    if (cardinalityNext > cardinality && index == (cardinality - 1)) {
        cardinalityUpdated = cardinalityNext;
    } else {
        cardinalityUpdated = cardinality;
    }
```

* `cardinalityNext > cardinality`이면 오라클 배열이 확장되었음을 의미합니다. `index == (cardinality - 1)`이면, 즉 마지막 쓰기 위치가 마지막 관측 점인 경우 이번에는 확장된 공간에 계속 써야 하므로 `cardinalityUpdated`는 확장된 배열 길이 `cardinalityNext`를 사용합니다.
* 그렇지 않으면 아직 채워지지 않았으므로 이전 배열 길이 `cardinality`를 계속 사용합니다.

```solidity
    indexUpdated = (index + 1) % cardinalityUpdated;
```

관측 점 배열에 대한 이번 쓰기 작업의 인덱스 `indexUpdated`를 업데이트합니다. `% cardinalityUpdated`는 순환 쓰기를 위한 인덱스를 계산하기 위한 것입니다.

```solidity
    self[indexUpdated] = transform(last, blockTimestamp, tick, liquidity);
```

[transform](#transform) 함수를 호출하여 최신 관측 지점 데이터를 계산하고, 관측 지점 배열의 `indexUpdated` 위치에 기록합니다.

#### grow

관측 지점 배열을 확장하여 기록 가능한 관측 지점의 수를 늘립니다. 컨트랙트는 기본적으로 1개의 관측 지점만 저장할 수 있으므로, 더 많은 관측 지점을 지원하려면 사용자가 수동으로 컨트랙트를 호출하여 관측 지점 배열을 확장해야 합니다.

> 참고: `grow` 함수는 `internal`로 표시되어 있으며, 사용자는 실제로 `UniswapV3Pool.sol`의 [increaseObservationCardinalityNext](#increaseObservationCardinalityNext) 함수를 통해 관측 지점 배열을 확장합니다.

확장 방식은 `SSTORE` 연산을 트리거하므로, 이 함수를 호출하는 사용자가 관련 가스 비용을 지불해야 합니다.

```solidity
/// @notice Prepares the oracle array to store up to `next` observations
/// @param self The stored oracle array
/// @param current The current next cardinality of the oracle array
/// @param next The proposed next cardinality which will be populated in the oracle array
/// @return next The next cardinality which will be populated in the oracle array
function grow(
    Observation[65535] storage self,
    uint16 current,
    uint16 next
) internal returns (uint16) {
    require(current > 0, 'I');
    // no-op if the passed next value isn't greater than the current next value
    if (next <= current) return current;
    // store in each slot to prevent fresh SSTOREs in swaps
    // this data will not be used because the initialized boolean is still false
    for (uint16 i = current; i < next; i++) self[i].blockTimestamp = 1;
    return next;
}
```

#### lte

두 타임스탬프를 비교합니다.

컨트랙트는 타임스탬프를 표현하기 위해 `uint32` 타입을 사용하므로, 최대값은 $2^{32}-1 = 4294967295$이며, 이는 `2106년 2월 7일 오전 6:28:15 GMT+00:00`에 해당합니다. 만약 두 시간 `a`와 `b` ($a < b$)가 `uint32`의 최대값 양쪽에 각각 위치한다면, `b`는 오버플로우되어 $a > b$가 되므로, 값을 직접 비교하면 잘못된 결과가 나옵니다. 따라서 통일된 시간 비교가 필요합니다.

> 참고: 실제로는 약 136년마다 오버플로우 문제가 발생합니다.

이 함수는 3개의 파라미터 `time`, `a`, `b`를 받습니다. 여기서 `time`은 기준 시간이며, `a`와 `b`는 논리적으로 `time`보다 작거나 같습니다. 함수는 `bool` 값을 반환하며, `a`가 논리적으로 `b`보다 작거나 같은지를 나타냅니다.

```solidity
/// @notice comparator for 32-bit timestamps
/// @dev safe for 0 or 1 overflows, a and b _must_ be chronologically before or equal to time
/// @param time A timestamp truncated to 32 bits
/// @param a A comparison timestamp from which to determine the relative position of `time`
/// @param b From which to determine the relative position of `time`
/// @return bool Whether `a` is chronologically <= `b`
function lte(
    uint32 time,
    uint32 a,
    uint32 b
) private pure returns (bool) {
    // if there hasn't been overflow, no need to adjust
    if (a <= time && b <= time) return a <= b;

    uint256 aAdjusted = a > time ? a : a + 2**32;
    uint256 bAdjusted = b > time ? b : b + 2**32;

    return aAdjusted <= bAdjusted;
}
```

* `a`와 `b`가 모두 `time`보다 작거나 같다면, 오버플로우가 발생하지 않았으므로 `a <= b`를 바로 반환합니다.
* `a`와 `b`는 모두 논리적으로 `time`보다 작거나 같으므로:
  * 만약 $a > time$이라면 오버플로우가 발생한 것이며, `aAdjusted`는 시간 `a`에 $2^{32}$를 더한 값입니다.
  * 마찬가지로 `bAdjusted`는 시간 `b`에 $2^{32}$를 더한 값입니다.
  * 마지막으로 조정된 두 시간 `aAdjusted`와 `bAdjusted`를 비교합니다.

따라서 이 함수는 `a`, `b`, `time`이 0에서 \(2^{32}\)의 시간 차이를 가질 때 오버플로우에 안전합니다.

> 두 번의 \(2^{32} - 1\)를 넘을 수는 없습니다.

#### binarySearch

지정된 목표 시간의 관측 지점을 찾기 위해 이진 탐색을 수행합니다.

파라미터는 다음과 같습니다:

* `self`: 관측 지점 배열
* `time`: 현재 블록 시간
* `target`: 목표 시간
* `index`: 마지막으로 기록된 관측 지점의 인덱스
* `cardinality`: 관측 지점 배열의 현재 길이 (기록 가능한 공간)

```solidity
/// @notice Fetches the observations beforeOrAt and atOrAfter a target, i.e. where [beforeOrAt, atOrAfter] is satisfied.
/// The result may be the same observation, or adjacent observations.
/// @dev The answer must be contained in the array, used when the target is located within the stored observation
/// boundaries: older than the most recent observation and younger, or the same age as, the oldest observation
/// @param self The stored oracle array
/// @param time The current block.timestamp
/// @param target The timestamp at which the reserved observation should be for
/// @param index The index of the observation that was most recently written to the observations array
/// @param cardinality The number of populated elements in the oracle array
/// @return beforeOrAt The observation recorded before, or at, the target
/// @return atOrAfter The observation recorded at, or after, the target
function binarySearch(
    Observation[65535] storage self,
    uint32 time,
    uint32 target,
    uint16 index,
    uint16 cardinality
) private view returns (Observation memory beforeOrAt, Observation memory atOrAfter) {
```

이진 탐색 알고리즘을 위해 먼저 왼쪽과 오른쪽 경계점을 확인해야 합니다.

```solidity
    uint256 l = (index + 1) % cardinality; // oldest observation
    uint256 r = l + cardinality - 1; // newest observation
```

마지막 인덱스가 `index`이므로, 오른쪽으로 한 칸 이동(모듈로 연산 적용)한 것이 왼쪽 경계점 `l` (가장 오래된 인덱스, 한 바퀴를 다 돌아서 기록된 경우)입니다. 오른쪽 경계점은 `l + cardinality - 1`입니다. 이진 탐색에서 오른쪽 노드가 왼쪽 노드보다 작으면 안 되기 때문에, 오른쪽 경계점 `r`에는 모듈로 연산을 적용하지 않는다는 점에 유의하세요.

```solidity
    uint256 i;
    while (true) {
        i = (l + r) / 2;

        beforeOrAt = self[i % cardinality];

        // we've landed on an uninitialized tick, keep searching higher (more recently)
        if (!beforeOrAt.initialized) {
            l = i + 1;
            continue;
        }

        atOrAfter = self[(i + 1) % cardinality];
```

계산된 중간 지점이 초기화되지 않은 경우(즉, 배열 공간이 가득 차지 않은 경우), 구간의 오른쪽 절반(더 최신 방향)을 사용하여 이진 탐색을 계속합니다.

`atOrAfter`는 바로 오른쪽의 관측 지점입니다.

```solidity
        bool targetAtOrAfter = lte(time, beforeOrAt.blockTimestamp, target);

        // check if we've found the answer!
        if (targetAtOrAfter && lte(time, target, atOrAfter.blockTimestamp)) break;

        if (!targetAtOrAfter) r = i - 1;
        else l = i + 1;
    }
```

* 목표 시간 `target`이 `beforeOrAt`과 `atOrAfter` 시간 사이라면, 이진 탐색을 종료하고 두 관측 지점 `beforeOrAt`과 `atOrAfter`를 반환합니다.
* 그렇지 않다면:
  * `beforeOrAt`의 시간이 목표 시간 `target`보다 크다면, 왼쪽 절반(더 작은 시간 방향)에서 이진 탐색을 계속합니다.
  * 목표 시간 `target`이 `atOrAfter`의 시간보다 크다면, 오른쪽 절반(더 큰 시간 방향)에서 탐색을 계속합니다.

`cardinality`가 10이고 `index`가 5라고 가정할 때, 여러 변수의 논리적 관계를 다음과 같이 그릴 수 있습니다:

![](./assets/binarySearch.jpg)

#### getSurroundingObservations

목표 시간 `target`에 대해 `target`이 `[beforeOrAt, atOrAfter]` 사이에 위치하는 관측 지점 데이터 `beforeOrAt`과 `atOrAfter`를 얻습니다.

```solidity
/// @notice Fetches the observations beforeOrAt and atOrAfter a given target, i.e. where [beforeOrAt, atOrAfter] is satisfied
/// @dev Assumes there is at least 1 initialized observation.
/// Used by observeSingle() to compute the counterfactual accumulator values as of a given block timestamp.
/// @param self The stored oracle array
/// @param time The current block.timestamp
/// @param target The timestamp at which the reserved observation should be for
/// @param tick The active tick at the time of the returned or simulated observation
/// @param index The index of the observation that was most recently written to the observations array
/// @param liquidity The total pool liquidity at the time of the call
/// @param cardinality The number of populated elements in the oracle array
/// @return beforeOrAt The observation which occurred at, or before, the given timestamp
/// @return atOrAfter The observation which occurred at, or after, the given timestamp
function getSurroundingObservations(
    Observation[65535] storage self,
    uint32 time,
    uint32 target,
    int24 tick,
    uint16 index,
    uint128 liquidity,
    uint16 cardinality
) private view returns (Observation memory beforeOrAt, Observation memory atOrAfter) {
    // optimistically set before to the newest observation
    beforeOrAt = self[index];

    // if the target is chronologically at or after the newest observation, we can early return
    if (lte(time, beforeOrAt.blockTimestamp, target)) {
        if (beforeOrAt.blockTimestamp == target) {
            // if newest observation equals target, we're in the same block, so we can ignore atOrAfter
            return (beforeOrAt, atOrAfter);
        } else {
            // otherwise, we need to transform
            return (beforeOrAt, transform(beforeOrAt, target, tick, liquidity));
        }
    }
```

먼저 `beforeOrAt`을 가장 최신 관측 지점으로 설정합니다:

`beforeOrAt` 시간이 목표 시간 `target`보다 작거나 같다면:
  * 목표 시간과 같다면, `beforeOrAt`을 바로 반환하고 `atOrAfter`는 무시합니다.
  * 목표 시간보다 작다면, `beforeOrAt`과 `target`을 기반으로 [transform](#transform)을 사용하여 임시로 관측 지점을 생성하고 이를 `atOrAfter`로 반환합니다.

```solidity
    // now, set before to the oldest observation
    beforeOrAt = self[(index + 1) % cardinality];
    if (!beforeOrAt.initialized) beforeOrAt = self[0];
```

그렇지 않다면, 마지막(최신) 관측 지점의 시간이 `target`보다 크다는 것을 확인할 수 있습니다.

`beforeOrAt`을 순환상 오른쪽으로 한 칸 이동한 관측 지점, 즉 가장 오래된 관측 지점으로 설정합니다. 만약 이 관측 지점이 초기화되지 않았다면, 배열이 꽉 차지 않은 것이므로 가장 이른 것은 0번 인덱스의 관측 지점이어야 합니다.

```solidity
    // ensure that the target is chronologically at or after the oldest observation
    require(lte(time, beforeOrAt.blockTimestamp, target), 'OLD');
```

`target` 시간이 가장 이른 관측 지점의 시간보다 크거나 같음을 확인합니다. 따라서 `target`은 분명히 가장 이른 관측 지점과 가장 늦은 관측 지점의 시간 구간 내에 있으므로 이진 탐색을 사용할 수 있습니다.

```solidity
    // if we've reached this point, we have to binary search
    return binarySearch(self, time, target, index, cardinality);
```

#### observeSingle

지정된 시간의 관측 지점 데이터를 얻습니다.

함수의 파라미터는 다음과 같습니다:

* `self`: 오라클 관측 지점 배열
* `time`: 현재 블록 시간
* `secondsAgo`: 현재 시간으로부터 몇 초 전의 관측 지점을 찾을지 지정
* `tick`: 목표 시간의 `tick` (가격)
* `index`: 마지막으로 기록된 관측 지점의 인덱스
* `liquidity`: 현재 전역 가용 유동성
* `cardinality`: 오라클 배열의 현재 길이 (기록 가능한 공간)

반환값:

* `tickCumulative`: 풀 페어 생성 이후 `secondsAgo`까지의 누적 `tick`
* `secondsPerLiquidityCumulativeX128`: 풀 페어 생성 이후 `secondsAgo`까지의 유동성 당 누적 지속 시간 (X128)

```solidity
/// @dev Reverts if an observation at or before the desired observation timestamp does not exist.
/// 0 may be passed as `secondsAgo' to return the current cumulative values.
/// If called with a timestamp falling between two observations, returns the counterfactual accumulator values
/// at exactly the timestamp between the two observations.
/// @param self The stored oracle array
/// @param time The current block timestamp
/// @param secondsAgo The amount of time to look back, in seconds, at which point to return an observation
/// @param tick The current tick
/// @param index The index of the observation that was most recently written to the observations array
/// @param liquidity The current in-range pool liquidity
/// @param cardinality The number of populated elements in the oracle array
/// @return tickCumulative The tick * time elapsed since the pool was first initialized, as of `secondsAgo`
/// @return secondsPerLiquidityCumulativeX128 The time elapsed / max(1, liquidity) since the pool was first initialized, as of `secondsAgo`
function observeSingle(
    Observation[65535] storage self,
    uint32 time,
    uint32 secondsAgo,
    int24 tick,
    uint16 index,
    uint128 liquidity,
    uint16 cardinality
) internal view returns (int56 tickCumulative, uint160 secondsPerLiquidityCumulativeX128) {
    if (secondsAgo == 0) {
        Observation memory last = self[index];
        if (last.blockTimestamp != time) last = transform(last, time, tick, liquidity);
        return (last.tickCumulative, last.secondsPerLiquidityCumulativeX128);
    }
```

`secondsAgo == 0`이면 현재 시간의 관측 지점을 가져온다는 의미입니다. 현재 시간이 마지막으로 기록된 관측 지점 시간과 같지 않다면, [transform](#transform)을 사용하여 임시 관측 지점을 생성하고(참고: 이 관측 지점은 기록되지 않음) 관련 데이터를 반환합니다.

```solidity
    uint32 target = time - secondsAgo;

    (Observation memory beforeOrAt, Observation memory atOrAfter) =
        getSurroundingObservations(self, time, target, tick, index, liquidity, cardinality);
```

`secondsAgo`를 기반으로 목표 시간 `target`을 계산하고, [getSurroundingObservations](#getSurroundingObservations) 함수를 사용하여 목표 시간과 가장 가까운 경계 관측 지점 `beforeOrAt`과 `atOrAfter`를 찾습니다.

```solidity
    if (target == beforeOrAt.blockTimestamp) {
        // we're at the left boundary
        return (beforeOrAt.tickCumulative, beforeOrAt.secondsPerLiquidityCumulativeX128);
    } else if (target == atOrAfter.blockTimestamp) {
        // we're at the right boundary
        return (atOrAfter.tickCumulative, atOrAfter.secondsPerLiquidityCumulativeX128);
    } else {
        // we're in the middle
        uint32 observationTimeDelta = atOrAfter.blockTimestamp - beforeOrAt.blockTimestamp;
        uint32 targetDelta = target - beforeOrAt.blockTimestamp;
        return (
            beforeOrAt.tickCumulative +
                ((atOrAfter.tickCumulative - beforeOrAt.tickCumulative) / observationTimeDelta) *
                targetDelta,
            beforeOrAt.secondsPerLiquidityCumulativeX128 +
                uint160(
                    (uint256(
                        atOrAfter.secondsPerLiquidityCumulativeX128 - beforeOrAt.secondsPerLiquidityCumulativeX128
                    ) * targetDelta) / observationTimeDelta
                )
        );
    }
```

[getSurroundingObservations](#getSurroundingObservations) 함수에 따라 왼쪽 경계점 `beforeOrAt`을 우선 사용합니다:

* 목표 시간이 `beforeOrAt` 시간과 같다면, 해당 관측 지점의 관련 데이터를 바로 반환합니다.
* 목표 시간이 `atOrAfter` 시간과 같다면, 역시 관련 데이터를 반환합니다.
* 목표 시간이 `beforeOrAt`과 `atOrAfter` 사이에 있다면, 시간 비례(가중치)에 따라 관련 값을 계산해야 합니다:
  * `observationTimeDelta`는 `beforeOrAt`과 `atOrAfter`의 시간 차이(아래 식의 $\Delta{t}$)이고, `targetDelta`는 `beforeOrAt`과 `target`의 시간 차이입니다.
  * $\Delta{tickCumulative} = tick \cdot \Delta{t}$이므로, `target`까지의 값은 $\frac{\Delta{tickCumulative}}{\Delta{t}} \cdot targetDelta$가 됩니다.
  * 마찬가지로 $\Delta{secondsPerLiquidityCumulativeX128} = \frac{\Delta{t}}{liquidity}$이므로, `target`까지의 값은 $\frac{\Delta{secondsPerLiquidityCumulativeX128}}{\Delta{t}} \cdot targetDelta$가 됩니다.

#### observe

지정된 시간들의 관측 지점 데이터를 배치로 얻습니다.

이 함수는 주로 [observeSingle](#observeSingle)을 호출하여 단일 지정 시간의 관측 지점 데이터를 얻은 다음, 이를 배치로 반환합니다.

```solidity
/// @notice Returns the accumulator values as of each time seconds ago from the given time in the array of `secondsAgos`
/// @dev Reverts if `secondsAgos` > oldest observation
/// @param self The stored oracle array
/// @param time The current block.timestamp
/// @param secondsAgos Each amount of time to look back, in seconds, at which point to return an observation
/// @param tick The current tick
/// @param index The index of the observation that was most recently written to the observations array
/// @param liquidity The current in-range pool liquidity
/// @param cardinality The number of populated elements in the oracle array
/// @return tickCumulatives The tick * time elapsed since the pool was first initialized, as of each `secondsAgo`
/// @return secondsPerLiquidityCumulativeX128s The cumulative seconds / max(1, liquidity) since the pool was first initialized, as of each `secondsAgo`
function observe(
    Observation[65535] storage self,
    uint32 time,
    uint32[] memory secondsAgos,
    int24 tick,
    uint16 index,
    uint128 liquidity,
    uint16 cardinality
) internal view returns (int56[] memory tickCumulatives, uint160[] memory secondsPerLiquidityCumulativeX128s) {
    require(cardinality > 0, 'I');

    tickCumulatives = new int56[](secondsAgos.length);
    secondsPerLiquidityCumulativeX128s = new uint160[](secondsAgos.length);
    for (uint256 i = 0; i < secondsAgos.length; i++) {
        (tickCumulatives[i], secondsPerLiquidityCumulativeX128s[i]) = observeSingle(
            self,
            time,
            secondsAgos[i],
            tick,
            index,
            liquidity,
            cardinality
        );
    }
```
