[English](./README.md) | [中文](./README_zh.md)

# Uniswap v3 스마트 컨트랙트 심층 분석 (2부)

###### tags: `uniswap` `uniswap-v3` `smart contract` `solidity`

## Uniswap-v3-periphery

[Uniswap-v3-core](../dive-into-uniswap-v3-contracts/README.md) 컨트랙트가 기본적인 메서드들을 정의한다면, Uniswap-v3-periphery 컨트랙트는 우리가 직접 상호작용하는 컨트랙트입니다.

예를 들어, Uniswap v3 포지션이 NFT라는 것은 잘 알려져 있습니다. 이러한 NFT는 코어 컨트랙트에는 NFT의 개념이 없으며, 페리퍼리(periphery) 컨트랙트에서 생성되고 관리됩니다.

### NonfungiblePositionManager.sol

포지션 관리 컨트랙트로, 전역적으로 유일하며, 모든 거래 쌍(trading pair)의 포지션을 관리할 책임이 있습니다. 주로 다음 메서드들을 포함합니다:

* [createAndInitializePoolIfNecessary](#createAndInitializePoolIfNecessary): 컨트랙트 생성 및 초기화
* [mint](#mint): 포지션 생성
* [increaseLiquidity](#increaseLiquidity): 유동성 증가
* [decreaseLiquidity](#decreaseLiquidity): 유동성 감소
* [burn](#burn): 포지션 소각
* [collect](#collect): 토큰 수령

이 컨트랙트는 `ERC721`을 상속받아 NFT를 발행(mint)할 수 있다는 점이 중요합니다. 각 Uniswap v3 포지션(`owner`, `tickLower`, `tickUpper`에 의해 결정됨)은 고유하기 때문에 NFT로 표현하기에 매우 적합합니다.

#### createAndInitializePoolIfNecessary

[Uniswap-v3-core](../dive-into-uniswap-v3-contracts/README.md)에서 언급했듯이, 거래 쌍 컨트랙트가 생성된 후에는 사용하기 전에 초기화가 필요합니다.

이 메서드는 일련의 작업을 하나로 결합하여 거래 쌍을 생성하고 초기화합니다.

```solidity
/// @inheritdoc IPoolInitializer
function createAndInitializePoolIfNecessary(
    address token0,
    address token1,
    uint24 fee,
    uint160 sqrtPriceX96
) external payable override returns (address pool) {
    require(token0 < token1);
    pool = IUniswapV3Factory(factory).getPool(token0, token1, fee);

    if (pool == address(0)) {
        pool = IUniswapV3Factory(factory).createPool(token0, token1, fee);
        IUniswapV3Pool(pool).initialize(sqrtPriceX96);
    } else {
        (uint160 sqrtPriceX96Existing, , , , , , ) = IUniswapV3Pool(pool).slot0();
        if (sqrtPriceX96Existing == 0) {
            IUniswapV3Pool(pool).initialize(sqrtPriceX96);
        }
    }
}
```

먼저 거래 쌍 토큰(`token0`와 `token1`) 및 수수료(`fee`)를 기반으로 `pool` 객체를 가져옵니다:

* 존재하지 않는 경우, Uniswap-v3-core 팩토리 컨트랙트의 `createPool`을 호출하여 거래 쌍을 생성하고 초기화합니다.
* 이미 존재하는 경우, 추가적인 `slot0`를 기반으로 초기화 여부(가격)를 확인합니다. 초기화되지 않은 경우 Uniswap-v3-core의 `initialize` 메서드를 호출하여 초기화합니다.

#### mint

새로운 포지션을 생성합니다. 파라미터는 다음과 같습니다:

* `token0`: 토큰 0
* `token1`: 토큰 1
* `fee`: 수수료 등급 (팩토리 컨트랙트에 정의된 수수료 등급과 일치해야 함)
* `tickLower`: 가격 하한
* `tickUpper`: 가격 상한
* `amount0Desired`: 예치하고자 하는 토큰 0의 양
* `amount1Desired`: 예치하고자 하는 토큰 1의 양
* `amount0Min`: 예치할 `token0`의 최소량 (프론트러닝 방지)
* `amount1Min`: 예치할 `token1`의 최소량 (프론트러닝 방지)
* `recipient`: 포지션 수령자
* `deadline`: 마감 기한 (이 시간 이후 요청은 무효) (리플레이 공격 방지)

반환값:

* `tokenId`: 각 포지션에 할당된 고유한 `tokenId` (NFT를 나타냄)
* `liquidity`: 포지션의 유동성
* `amount0`: `token0`의 양
* `amount1`: `token1`의 양

```solidity
/// @inheritdoc INonfungiblePositionManager
function mint(MintParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (
        uint256 tokenId,
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    IUniswapV3Pool pool;
    (liquidity, amount0, amount1, pool) = addLiquidity(
        AddLiquidityParams({
            token0: params.token0,
            token1: params.token1,
            fee: params.fee,
            recipient: address(this),
            tickLower: params.tickLower,
            tickUpper: params.tickUpper,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min
        })
    );
```

먼저 [addLiquidity](#addLiquidity) 메서드를 통해 유동성을 추가하고, 실제 `liquidity`, 소모된 `amount0`, `amount1`, 그리고 거래 쌍 `pool`을 얻습니다.

```solidity
    _mint(params.recipient, (tokenId = _nextId++));
```

`ERC721` 컨트랙트의 `_mint` 메서드를 통해 수령자 `recipient`에게 NFT를 발행합니다. `tokenId`는 1부터 증가합니다.

```solidity
    bytes32 positionKey = PositionKey.compute(address(this), params.tickLower, params.tickUpper);
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

    // idempotent set
    uint80 poolId =
        cachePoolKey(
            address(pool),
            PoolAddress.PoolKey({token0: params.token0, token1: params.token1, fee: params.fee})
        );

    _positions[tokenId] = Position({
        nonce: 0,
        operator: address(0),
        poolId: poolId,
        tickLower: params.tickLower,
        tickUpper: params.tickUpper,
        liquidity: liquidity,
        feeGrowthInside0LastX128: feeGrowthInside0LastX128,
        feeGrowthInside1LastX128: feeGrowthInside1LastX128,
        tokensOwed0: 0,
        tokensOwed1: 0
    });

    emit IncreaseLiquidity(tokenId, liquidity, amount0, amount1);
}
```

마지막으로 포지션 정보를 `_positions`에 저장합니다.

#### increaseLiquidity

포지션에 유동성을 추가합니다. 포지션의 토큰 양은 변경할 수 있지만 가격 범위는 변경할 수 없다는 점에 유의하세요.

파라미터는 다음과 같습니다:

* `tokenId`: 포지션 생성 시 반환된 `tokenId` (NFT의 `tokenId`)
* `amount0Desired`: 추가하고자 하는 `token0`의 양
* `amount1Desired`: 추가하고자 하는 `token1`의 양
* `amount0Min`: 추가할 `token0`의 최소량 (프론트러닝 방지)
* `amount1Min`: 추가할 `token1`의 최소량 (프론트러닝 방지)
* `deadline`: 마감 기한 (이 시간 이후 요청은 무효) (리플레이 공격 방지)

```solidity
/// @inheritdoc INonfungiblePositionManager
function increaseLiquidity(IncreaseLiquidityParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    Position storage position = _positions[params.tokenId];

    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];

    IUniswapV3Pool pool;
    (liquidity, amount0, amount1, pool) = addLiquidity(
        AddLiquidityParams({
            token0: poolKey.token0,
            token1: poolKey.token1,
            fee: poolKey.fee,
            tickLower: position.tickLower,
            tickUpper: position.tickUpper,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min,
            recipient: address(this)
        })
    );
```

먼저 `tokenId`를 기반으로 포지션 정보를 가져옵니다. [mint](#mint) 메서드와 마찬가지로, 여기에서도 [addLiquidity](#addLiquidity)를 통해 유동성을 추가하고, 추가된 `liquidity`, 소모된 `amount0`와 `amount1`, 그리고 거래 쌍 컨트랙트 `pool`을 반환합니다.

```solidity
    bytes32 positionKey = PositionKey.compute(address(this), position.tickLower, position.tickUpper);

    // this is now updated to the current transaction
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

    position.tokensOwed0 += uint128(
        FullMath.mulDiv(
            feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
            position.liquidity,
            FixedPoint128.Q128
        )
    );
    position.tokensOwed1 += uint128(
        FullMath.mulDiv(
            feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
            position.liquidity,
            FixedPoint128.Q128
        )
    );

    position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
    position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
    position.liquidity += liquidity;

    emit IncreaseLiquidity(params.tokenId, liquidity, amount0, amount1);
}
```

`pool` 객체의 최신 포지션 정보에 따라 컨트랙트의 포지션 상태를 업데이트합니다. 예를 들어 수령할 `token0`와 `token1`인 `tokensOwed0`와 `tokensOwed1`, 그리고 포지션의 현재 유동성 등을 업데이트합니다.

#### decreaseLiquidity

유동성을 제거합니다. 일부 또는 전체를 제거할 수 있습니다. 제거 후 토큰은 수령 대기 상태로 기록되며, 토큰을 실제로 수령하려면 [collect](#collect) 메서드를 다시 호출해야 합니다.

파라미터는 다음과 같습니다:

* `tokenId`: 포지션 생성 시 반환된 `tokenId` (NFT의 `tokenId`)
* `liquidity`: 제거할 유동성의 양
* `amount0Min`: 제거할 `token0`의 최소량 (프론트러닝 방지)
* `amount1Min`: 제거할 `token1`의 최소량 (프론트러닝 방지)
* `deadline`: 마감 기한 (이 시간 이후 요청은 무효) (리플레이 공격 방지)

```solidity
/// @inheritdoc INonfungiblePositionManager
function decreaseLiquidity(DecreaseLiquidityParams calldata params)
    external
    payable
    override
    isAuthorizedForToken(params.tokenId)
    checkDeadline(params.deadline)
    returns (uint256 amount0, uint256 amount1)
{
```

여기서 `isAuthorizedForToken` modifier가 사용된 것에 유의하세요:

```solidity
modifier isAuthorizedForToken(uint256 tokenId) {
    require(_isApprovedOrOwner(msg.sender, tokenId), 'Not approved');
    _;
}
```

현재 사용자가 `tokenId`를 조작할 권한이 있는지 확인합니다. 그렇지 않으면 제거가 금지됩니다.

```solidity
    require(params.liquidity > 0);
    Position storage position = _positions[params.tokenId];

    uint128 positionLiquidity = position.liquidity;
    require(positionLiquidity >= params.liquidity);
```

포지션 유동성이 제거하려는 유동성보다 크거나 같은지 확인합니다.

```solidity
    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];
    IUniswapV3Pool pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));
    (amount0, amount1) = pool.burn(position.tickLower, position.tickUpper, params.liquidity);

    require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
```

Uniswap-v3-core의 [burn](../dive-into-uniswap-v3-contracts/README.md#burn) 메서드를 호출하여 유동성을 소각하고, 해당하는 `token0` 및 `token1` 토큰 양인 `amount0`와 `amount1`을 반환받습니다. 그리고 이들이 `amount0Min` 및 `amount1Min` 제한을 충족하는지 확인합니다.

```solidity
    bytes32 positionKey = PositionKey.compute(address(this), position.tickLower, position.tickUpper);
    // this is now updated to the current transaction
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

    position.tokensOwed0 +=
        uint128(amount0) +
        uint128(
            FullMath.mulDiv(
                feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                positionLiquidity,
                FixedPoint128.Q128
            )
        );
    position.tokensOwed1 +=
        uint128(amount1) +
        uint128(
            FullMath.mulDiv(
                feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                positionLiquidity,
                FixedPoint128.Q128
            )
        );

    position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
    position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
    // subtraction is safe because we checked positionLiquidity is gte params.liquidity
    position.liquidity = positionLiquidity - params.liquidity;

    emit DecreaseLiquidity(params.tokenId, params.liquidity, amount0, amount1);
}
```

[increaseLiquidity](#increaseLiquidity)와 유사하게, 여기서 포지션의 수령 대기 토큰 등을 계산합니다.

#### burn

포지션 NFT를 소각합니다. 포지션의 유동성이 0이고 수령 대기 토큰이 0인 경우에만 NFT를 소각할 수 있습니다.

마찬가지로 이 메서드를 호출하려면 현재 사용자가 `tokenId`를 소유하고 있는지 확인해야 합니다.

```solidity
/// @inheritdoc INonfungiblePositionManager
function burn(uint256 tokenId) external payable override isAuthorizedForToken(tokenId) {
    Position storage position = _positions[tokenId];
    require(position.liquidity == 0 && position.tokensOwed0 == 0 && position.tokensOwed1 == 0, 'Not cleared');
    delete _positions[tokenId];
    _burn(tokenId);
}
```

#### collect

수령 대기 중인 토큰을 회수합니다.

파라미터는 다음과 같습니다:

* `tokenId`: 포지션 생성 시 반환된 `tokenId` (NFT의 `tokenId`)
* `recipient`: 토큰 수령자
* `amount0Max`: 수령할 `token0` 토큰의 최대량
* `amount1Max`: 수령할 `token1` 토큰의 최대량

```solidity
/// @inheritdoc INonfungiblePositionManager
function collect(CollectParams calldata params)
    external
    payable
    override
    isAuthorizedForToken(params.tokenId)
    returns (uint256 amount0, uint256 amount1)
{
    require(params.amount0Max > 0 || params.amount1Max > 0);
    // allow collecting to the nft position manager address with address 0
    address recipient = params.recipient == address(0) ? address(this) : params.recipient;

    Position storage position = _positions[params.tokenId];

    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];

    IUniswapV3Pool pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));

    (uint128 tokensOwed0, uint128 tokensOwed1) = (position.tokensOwed0, position.tokensOwed1);
```

수령할 토큰의 수량을 얻습니다.

```solidity
    // trigger an update of the position fees owed and fee growth snapshots if it has any liquidity
    if (position.liquidity > 0) {
        pool.burn(position.tickLower, position.tickUpper, 0);
        (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) =
            pool.positions(PositionKey.compute(address(this), position.tickLower, position.tickUpper));

        tokensOwed0 += uint128(
            FullMath.mulDiv(
                feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                position.liquidity,
                FixedPoint128.Q128
            )
        );
        tokensOwed1 += uint128(
            FullMath.mulDiv(
                feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                position.liquidity,
                FixedPoint128.Q128
            )
        );

        position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
        position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
    }
```

포지션에 유동성이 포함되어 있다면, 포지션 상태 업데이트를 트리거합니다. 여기서는 0 유동성을 사용하여 트리거합니다. 이는 Uniswap-v3-core가 `mint`와 `burn` 중에만 포지션 상태를 업데이트하기 때문이며, `collect` 메서드는 `swap` 후에 호출될 수 있어 포지션 상태가 최신이 아닐 수 있기 때문입니다.

```solidity
    // compute the arguments to give to the pool#collect method
    (uint128 amount0Collect, uint128 amount1Collect) =
        (
            params.amount0Max > tokensOwed0 ? tokensOwed0 : params.amount0Max,
            params.amount1Max > tokensOwed1 ? tokensOwed1 : params.amount1Max
        );

    // the actual amounts collected are returned
    (amount0, amount1) = pool.collect(
        recipient,
        position.tickLower,
        position.tickUpper,
        amount0Collect,
        amount1Collect
    );

    // sometimes there will be a few less wei than expected due to rounding down in core, but we just subtract the full amount expected
    // instead of the actual amount so we can burn the token
    (position.tokensOwed0, position.tokensOwed1) = (tokensOwed0 - amount0Collect, tokensOwed1 - amount1Collect);

    emit Collect(params.tokenId, recipient, amount0Collect, amount1Collect);
}
```

Uniswap-v3-core의 `collect` 메서드를 호출하여 토큰을 회수하고, 포지션의 수령 대기 토큰을 업데이트합니다.

### SwapRouter.sol

토큰 스왑(교환)을 수행하며, 다음 메서드들을 포함합니다:

* [exactInputSingle](#exactInputSingle): 단일 단계 스왑, 입력 토큰 양을 지정하여 가능한 많은 출력 토큰을 얻음
* [exactInput](#exactInput): 다중 단계 스왑, 입력 토큰 양을 지정하여 가능한 많은 출력 토큰을 얻음
* [exactOutputSingle](#exactOutputSingle): 단일 단계 스왑, 출력 토큰 양을 지정하여 가능한 적은 입력 토큰을 제공함
* [exactOutput](#exactOutput): 다중 단계 스왑, 출력 토큰 양을 지정하여 가능한 적은 입력 토큰을 제공함

또한 컨트랙트는 다음도 구현합니다:

* [uniswapV3SwapCallback](#uniswapV3SwapCallback): 스왑 콜백 메서드
* [exactInputInternal](#exactInputInternal): 단일 단계 스왑, 내부 메서드, 입력 토큰 양을 지정하여 가능한 많은 출력 토큰을 얻음
* [exactOutputInternal](#exactOutputInternal): 단일 단계 스왑, 내부 메서드, 출력 토큰 양을 지정하여 가능한 적은 입력 토큰을 제공함

#### exactInputSingle

단일 단계 스왑으로, 입력 토큰 양을 지정하여 가능한 많은 출력 토큰을 얻습니다.

파라미터는 다음과 같습니다:

* `tokenIn`: 입력 토큰 주소
* `tokenOut`: 출력 토큰 주소
* `fee`: 수수료 등급
* `recipient`: 출력 토큰 수령자
* `deadline`: 마감 기한, 이 시간 이후 요청은 무효
* `amountIn`: 입력 토큰 양
* `amountOutMinimum`: 받을 최소 출력 토큰 양
* `sqrtPriceLimitX96`: (최대 또는 최소) 제한 가격

반환값:

* `amountOut`: 출력 토큰 양

```solidity
/// @inheritdoc ISwapRouter
function exactInputSingle(ExactInputSingleParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountOut)
{
    amountOut = exactInputInternal(
        params.amountIn,
        params.recipient,
        params.sqrtPriceLimitX96,
        SwapCallbackData({path: abi.encodePacked(params.tokenIn, params.fee, params.tokenOut), payer: msg.sender})
    );
    require(amountOut >= params.amountOutMinimum, 'Too little received');
}
```

이 메서드는 실제로 [exactInputInternal](#exactInputInternal)을 호출하며, 마지막으로 출력 토큰 양 `amountOut`이 최소 요구량 `amountOutMinimum`을 충족하는지 확인합니다.

참고로, `SwapCallbackData`의 `path`는 [Path.sol](#Pathsol)에 정의된 형식에 따라 인코딩됩니다.

#### exactInput

다중 단계 스왑으로, 입력 토큰 양을 지정하여 가능한 많은 출력 토큰을 얻습니다.

파라미터는 다음과 같습니다:

* `path`: 스왑 경로, 형식은 [Path.sol](#Pathsol) 참조
* `recipient`: 출력 토큰 수령자
* `deadline`: 거래 마감 기한
* `amountIn`: 입력 토큰 양
* `amountOutMinimum`: 최소 출력 토큰 양

반환값:

* `amountOut`: 출력 토큰 양

```solidity
/// @inheritdoc ISwapRouter
function exactInput(ExactInputParams memory params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountOut)
{
    address payer = msg.sender; // msg.sender pays for the first hop

    while (true) {
        bool hasMultiplePools = params.path.hasMultiplePools();

        // the outputs of prior swaps become the inputs to subsequent ones
        params.amountIn = exactInputInternal(
            params.amountIn,
            hasMultiplePools ? address(this) : params.recipient, // for intermediate swaps, this contract custodies
            0,
            SwapCallbackData({
                path: params.path.getFirstPool(), // only the first pool in the path is necessary
                payer: payer
            })
        );

        // decide whether to continue or terminate
        if (hasMultiplePools) {
            payer = address(this); // at this point, the caller has paid
            params.path = params.path.skipToken();
        } else {
            amountOut = params.amountIn;
            break;
        }
    }

    require(amountOut >= params.amountOutMinimum, 'Too little received');
}
```

다중 단계 스왑에서는 스왑 경로에 따라 여러 단일 단계 스왑으로 나누어 진행해야 하며, 경로가 끝날 때까지 루프를 돕니다.

첫 번째 스왑 단계라면 `payer`는 컨트랙트 호출자이고, 그렇지 않으면 `payer`는 현재 `SwapRouter` 컨트랙트입니다.

루프 내에서 먼저 [hasMultiplePools](#hasMultiplePools)에 따라 `path`에 2개 이상의 풀이 남았는지 확인합니다. 그렇다면 중간 스왑 단계의 수령 주소는 현재 `SwapRouter` 컨트랙트로 설정되고, 그렇지 않으면 함수 입력 파라미터인 `recipient`로 설정됩니다.

각 스왑 단계 후에는 현재 `path`에서 앞의 20+3바이트를 제거합니다(Pop). 즉, 앞의 토큰+수수료 정보를 제거하고 다음 스왑으로 진입하며, 각 단계의 출력을 다음 스왑의 입력으로 사용합니다.

각 스왑 단계는 [exactInputInternal](#exactInputInternal)을 호출하여 실행됩니다.

다중 단계 스왑 후, 최종 `amountOut`이 최소 출력 토큰 요구량 `amountOutMinimum`을 충족하는지 확인합니다.

#### exactOutputSingle

단일 단계 스왑으로, 출력 토큰 양을 지정하여 가능한 적은 입력 토큰을 제공합니다.

파라미터는 다음과 같습니다:

* `tokenIn`: 입력 토큰 주소
* `tokenOut`: 출력 토큰 주소
* `fee`: 수수료 등급
* `recipient`: 출력 토큰 수령자
* `deadline`: 요청 마감 기한
* `amountOut`: 출력 토큰 양
* `amountInMaximum`: 최대 입력 토큰 양
* `sqrtPriceLimitX96`: 최대 또는 최소 토큰 가격

반환값:

* `amountIn`: 실제 입력 토큰 양

```solidity
/// @inheritdoc ISwapRouter
function exactOutputSingle(ExactOutputSingleParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountIn)
{
    // avoid an SLOAD by using the swap return data
    amountIn = exactOutputInternal(
        params.amountOut,
        params.recipient,
        params.sqrtPriceLimitX96,
        SwapCallbackData({path: abi.encodePacked(params.tokenOut, params.fee, params.tokenIn), payer: msg.sender})
    );

    require(amountIn <= params.amountInMaximum, 'Too much requested');
    // has to be reset even though we don't use it in the single hop case
    amountInCached = DEFAULT_AMOUNT_IN_CACHED;
}
```

[exactOutputInternal](#exactOutputInternal)을 호출하여 단일 단계 스왑을 완료하고, 실제 입력 토큰 양 `amountIn`이 최대 입력 토큰 양 `amountInMaximum`보다 작거나 같은지 확인합니다.

#### exactOutput

다중 단계 스왑으로, 출력 토큰 양을 지정하여 가능한 적은 입력 토큰을 제공합니다.

파라미터는 다음과 같습니다:

* `path`: 스왑 경로, 형식은 [Path.sol](#Path) 참조
* `recipient`: 출력 토큰 수령자
* `deadline`: 요청 마감 기한
* `amountOut`: 지정된 출력 토큰 양
* `amountInMaximum`: 최대 입력 토큰 양

```solidity
/// @inheritdoc ISwapRouter
function exactOutput(ExactOutputParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountIn)
{
    // it's okay that the payer is fixed to msg.sender here, as they're only paying for the "final" exact output
    // swap, which happens first, and subsequent swaps are paid for within nested callback frames
    exactOutputInternal(
        params.amountOut,
        params.recipient,
        0,
        SwapCallbackData({path: params.path, payer: msg.sender})
    );

    amountIn = amountInCached;
    require(amountIn <= params.amountInMaximum, 'Too much requested');
    amountInCached = DEFAULT_AMOUNT_IN_CACHED;
}
```

[exactOutputInternal](#exactOutputInternal)을 호출하여 스왑을 완료합니다. 참고로 이 메서드는 콜백 메서드에서 다음 단계 스왑을 계속 진행하므로, [exactInput](#exactInput)처럼 루프 거래가 필요하지 않습니다.

마지막으로 실제 입력 토큰 양 `amountIn`이 최대 입력 토큰 양 `amountInMaximum`보다 작거나 같은지 확인합니다.

#### exactInputInternal

단일 단계 스왑, 내부 메서드이며, 입력 토큰 양을 지정하여 가능한 많은 출력 토큰을 얻습니다.

```solidity
/// @dev Performs a single exact input swap
function exactInputInternal(
    uint256 amountIn,
    address recipient,
    uint160 sqrtPriceLimitX96,
    SwapCallbackData memory data
) private returns (uint256 amountOut) {
    // allow swapping to the router address with address 0
    if (recipient == address(0)) recipient = address(this);
```

`recipient`가 지정되지 않은 경우, 기본값은 현재 `SwapRouter` 컨트랙트 주소입니다. 이는 다중 단계 스왑에서 중간 토큰을 현재 `SwapRouter` 컨트랙트에 저장해야 하기 때문입니다.

```solidity
    (address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool();
```

[decodeFirstPool](#decodeFirstPool)에 따라 `path`의 첫 번째 풀 정보를 디코딩합니다.

```solidity
    bool zeroForOne = tokenIn < tokenOut;
```

Uniswap v3 풀은 `token0` 주소가 `token1`보다 작으므로, 두 토큰 주소를 기반으로 현재 스왑이 `token0`에서 `token1`으로 진행되는지 확인합니다. 참고로 `tokenIn`은 `token0`일 수도 있고 `token1`일 수도 있습니다.

```solidity
    (int256 amount0, int256 amount1) =
        getPool(tokenIn, tokenOut, fee).swap(
            recipient,
            zeroForOne,
            amountIn.toInt256(),
            sqrtPriceLimitX96 == 0
                ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
                : sqrtPriceLimitX96,
            abi.encode(data)
        );
```

[swap](#swap) 메서드를 호출하여 이번 스왑을 완료하는 데 필요한 `amount0`와 `amount1`을 얻습니다. `token0`에서 `token1`으로 스왑하는 경우 `amount1`은 음수이고, 그렇지 않으면 `amount0`가 음수입니다.

`sqrtPriceLimitX96`이 지정되지 않은 경우, 최저 또는 최고 가격으로 기본 설정됩니다. 다중 단계 스왑에서는 각 단계의 가격을 지정할 수 없기 때문입니다.

```solidity
    return uint256(-(zeroForOne ? amount1 : amount0));
```

`amountOut`을 반환합니다.

#### exactOutputInternal

단일 단계 스왑, private 메서드이며, 출력 토큰 양을 지정하여 가능한 적은 입력 토큰을 제공합니다.

```solidity
/// @dev Performs a single exact output swap
function exactOutputInternal(
    uint256 amountOut,
    address recipient,
    uint160 sqrtPriceLimitX96,
    SwapCallbackData memory data
) private returns (uint256 amountIn) {
    // allow swapping to the router address with address 0
    if (recipient == address(0)) recipient = address(this);

    (address tokenOut, address tokenIn, uint24 fee) = data.path.decodeFirstPool();

    bool zeroForOne = tokenIn < tokenOut;
```

이 부분의 코드는 [exactInputInternal](#exactInputInternal)과 유사합니다.

```solidity
    (int256 amount0Delta, int256 amount1Delta) =
        getPool(tokenIn, tokenOut, fee).swap(
            recipient,
            zeroForOne,
            -amountOut.toInt256(),
            sqrtPriceLimitX96 == 0
                ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
                : sqrtPriceLimitX96,
            abi.encode(data)
        );
```

Uniswap-v3-core의 `swap` 메서드를 호출하여 단일 단계 스왑을 완료합니다. 출력 토큰 양이 지정되었으므로, 여기서는 `-amountOut.toInt256()`가 사용됩니다.
반환된 `amount0Delta`와 `amount1Delta`는 필요한 `token0` 양과 실제 출력된 `token1` 양입니다.

```solidity
    uint256 amountOutReceived;
    (amountIn, amountOutReceived) = zeroForOne
        ? (uint256(amount0Delta), uint256(-amount1Delta))
        : (uint256(amount1Delta), uint256(-amount0Delta));
    // it's technically possible to not receive the full output amount,
    // so if no price limit has been specified, require this possibility away
    if (sqrtPriceLimitX96 == 0) require(amountOutReceived == amountOut);
}
```

#### uniswapV3SwapCallback

스왑의 콜백 메서드로, `IUniswapV3SwapCallback.uniswapV3SwapCallback` 인터페이스를 구현합니다.

파라미터는 다음과 같습니다:

* `amount0Delta`: 이 스왑에서 발생한 `amount0` (`token0`에 해당); 컨트랙트 입장에서 0보다 크면 토큰을 지불해야 함을, 0보다 작으면 토큰을 받아야 함을 의미
* `amount1Delta`: 이 스왑에서 발생한 `amount1` (`token1`에 해당); 컨트랙트 입장에서 0보다 크면 토큰을 지불해야 함을, 0보다 작으면 토큰을 받아야 함을 의미
* `_data`: 콜백 파라미터, 여기서는 `SwapCallbackData` 타입

```solidity
/// @inheritdoc IUniswapV3SwapCallback
function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes calldata _data
) external override {
    require(amount0Delta > 0 || amount1Delta > 0); // swaps entirely within 0-liquidity regions are not supported
    SwapCallbackData memory data = abi.decode(_data, (SwapCallbackData));
    (address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool();
    CallbackValidation.verifyCallback(factory, tokenIn, tokenOut, fee);
```

콜백 파라미터 `_data`를 디코딩하고 스왑 경로의 첫 번째 거래 쌍 정보를 얻습니다.

```solidity
    (bool isExactInput, uint256 amountToPay) =
        amount0Delta > 0
            ? (tokenIn < tokenOut, uint256(amount0Delta))
            : (tokenOut < tokenIn, uint256(amount1Delta));
```

다양한 입력에 때라 몇 가지 거래 조합이 있습니다:

|시나리오|설명|amount0Delta > 0|amount1Delta > 0|tokenIn < tokenOut|isExactInput|amountToPay|
|---|---|---|---|---|---|---|
|1|`token0` 양 입력 지정, 가능한 많은 `token1` 출력|true|false|true|true|amount0Delta|
|2|가능한 적은 `token0` 제공, `token1` 양 출력 지정|true|false|true|false|amount0Delta|
|3|`token1` 양 입력 지정, 가능한 많은 `token0` 출력|false|true|false|true|amount1Delta|
|4|가능한 적은 `token1` 제공, `token0` 양 출력 지정|false|true|false|false|amount1Delta|

```solidity
    if (isExactInput) {
        pay(tokenIn, data.payer, msg.sender, amountToPay);
    } else {
        // either initiate the next swap or pay
        if (data.path.hasMultiplePools()) {
            data.path = data.path.skipToken();
            exactOutputInternal(amountToPay, msg.sender, 0, data);
        } else {
            amountInCached = amountToPay;
            tokenIn = tokenOut; // swap in/out because exact output swaps are reversed
            pay(tokenIn, data.payer, msg.sender, amountToPay);
        }
    }
```

* `isExactInput`인 경우, 즉 입력 토큰 지정 시나리오(위 표의 1, 3번)라면, `AddLiquidity` (1번) 또는 `amount1Delta` (3번) (둘 다 양수)을 `SwapRouter` 컨트랙트로 즉시 전송합니다.
* 출력 토큰 지정 시나리오인 경우:
    - 다중 단계 스왑이라면, 첫 23자를 제거(앞의 토큰+수수료 Pop)하고, 필요한 입력을 다음 단계의 출력으로 사용하여 다음 스왑 단계로 진입합니다.
    - 단일 단계 스왑(또는 마지막 단계)이라면, `tokenIn`과 `tokenOut`을 바꾸고 `SwapRouter` 컨트랙트로 전송합니다.
### LiquidityManagement.sol

#### uniswapV3MintCallback

유동성 추가를 위한 콜백 메서드입니다.

파라미터는 다음과 같습니다:

* `amount0Owed`: 전송해야 할 `token0`의 양
* `amount1Owed`: 전송해야 할 `token1`의 양
* `data`: `mint` 메서드에서 전달된 콜백 파라미터

```solidity
/// @inheritdoc IUniswapV3MintCallback
function uniswapV3MintCallback(
    uint256 amount0Owed,
    uint256 amount1Owed,
    bytes calldata data
) external override {
    MintCallbackData memory decoded = abi.decode(data, (MintCallbackData));
    CallbackValidation.verifyCallback(factory, decoded.poolKey);

    if (amount0Owed > 0) pay(decoded.poolKey.token0, decoded.payer, msg.sender, amount0Owed);
    if (amount1Owed > 0) pay(decoded.poolKey.token1, decoded.payer, msg.sender, amount1Owed);
}
```

먼저 콜백 파라미터 `MintCallbackData`를 디코딩하고, 이 메서드가 지정된 거래 쌍 컨트랙트에 의해 호출되었는지 확인합니다. 이 메서드는 `external` 메서드로 외부에서 호출될 수 있으므로 호출자를 확인해야 합니다.

마지막으로 지정된 토큰 수량을 호출자에게 전송합니다.

#### addLiquidity

초기화된 거래 쌍(풀)에 유동성을 추가합니다.

```solidity
/// @notice Add liquidity to an initialized pool
function addLiquidity(AddLiquidityParams memory params)
    internal
    returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1,
        IUniswapV3Pool pool
    )
{
    PoolAddress.PoolKey memory poolKey =
        PoolAddress.PoolKey({token0: params.token0, token1: params.token1, fee: params.fee});

    pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));
```

`factory`, `token0`, `token1`, `fee`를 기반으로 거래 쌍 `pool`을 가져옵니다.

```solidity
    // compute the liquidity amount
    {
        (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();
        uint160 sqrtRatioAX96 = TickMath.getSqrtRatioAtTick(params.tickLower);
        uint160 sqrtRatioBX96 = TickMath.getSqrtRatioAtTick(params.tickUpper);

        liquidity = LiquidityAmounts.getLiquidityForAmounts(
            sqrtPriceX96,
            sqrtRatioAX96,
            sqrtRatioBX96,
            params.amount0Desired,
            params.amount1Desired
        );
    }
```

`slot0`에서 현재 가격 `sqrtPriceX96`을 얻고, `tickLower`와 `tickUpper`를 기반으로 최저 가격 `sqrtRatioAX96`과 최고 가격 `sqrtRatioBX96`을 계산합니다.

[getLiquidityForAmounts](#getLiquidityForAmounts)에 따라 얻을 수 있는 최대 유동성을 계산합니다.

```solidity
    (amount0, amount1) = pool.mint(
        params.recipient,
        params.tickLower,
        params.tickUpper,
        liquidity,
        abi.encode(MintCallbackData({poolKey: poolKey, payer: msg.sender}))
    );
```

Uniswap-v3-core의 `mint` 메서드를 사용하여 유동성을 추가하고 실제 소모된 `amount0`와 `amount1`을 반환합니다.

Uniswap-v3-core의 `mint` 메서드에서 언급했듯이, 호출자는 [uniswapV3MintCallback](#LiquidityManagement.uniswapV3MintCallback) 인터페이스를 구현해야 합니다. 여기서 `MintCallbackData`가 콜백 파라미터로 전달되며, 이는 `uniswapV3MintCallback` 메서드에서 디코딩되어 거래 쌍 및 사용자 정보를 얻는 데 사용됩니다.

```solidity
require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
```

마지막으로 실제 소모된 `amount0`와 `amount1`이 `amount0Min`과 `amount1Min`의 최소 요구 조건을 충족하는지 확인합니다.

### LiquidityAmounts.sol

#### getLiquidityForAmount0

`amount0`와 가격 범위를 기반으로 유동성을 계산합니다.

Uniswap-v3-core의 `getAmount0Delta` 공식에 따르면:

$$
amount0 = x_b - x_a = L \cdot (\frac{1}{\sqrt{P_b}} - \frac{1}{\sqrt{P_a}}) = L \cdot (\frac{\sqrt{P_a} - \sqrt{P_b}}{\sqrt{P_a} \cdot \sqrt{P_b}})
$$

따라서 다음을 얻습니다:

$$
L = amount0 \cdot (\frac{\sqrt{P_a} \cdot \sqrt{P_b}}{\sqrt{P_a} - \sqrt{P_b}})
$$

```solidity
/// @notice Computes the amount of liquidity received for a given amount of token0 and price range
/// @dev Calculates amount0 * (sqrt(upper) * sqrt(lower)) / (sqrt(upper) - sqrt(lower))
/// @param sqrtRatioAX96 A sqrt price representing the first tick boundary
/// @param sqrtRatioBX96 A sqrt price representing the second tick boundary
/// @param amount0 The amount0 being sent in
/// @return liquidity The amount of returned liquidity
function getLiquidityForAmount0(
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint256 amount0
) internal pure returns (uint128 liquidity) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);
    uint256 intermediate = FullMath.mulDiv(sqrtRatioAX96, sqrtRatioBX96, FixedPoint96.Q96);
    return toUint128(FullMath.mulDiv(amount0, intermediate, sqrtRatioBX96 - sqrtRatioAX96));
}
```

#### getLiquidityForAmount1

`amount1`과 가격 범위를 기반으로 유동성을 계산합니다.

Uniswap-v3-core의 `getAmount1Delta` 공식에 따르면:

$$
amount1 = y_b - y_a = L \cdot \Delta{\sqrt{P}} = L \cdot (\sqrt{P_b} - \sqrt{P_a})
$$

따라서 다음을 얻습니다:

$$
L = \frac{amount1}{\sqrt{P_b} - \sqrt{P_a}}
$$

```solidity
/// @notice Computes the amount of liquidity received for a given amount of token1 and price range
/// @dev Calculates amount1 / (sqrt(upper) - sqrt(lower)).
/// @param sqrtRatioAX96 A sqrt price representing the first tick boundary
/// @param sqrtRatioBX96 A sqrt price representing the second tick boundary
/// @param amount1 The amount1 being sent in
/// @return liquidity The amount of returned liquidity
function getLiquidityForAmount1(
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);
    return toUint128(FullMath.mulDiv(amount1, FixedPoint96.Q96, sqrtRatioBX96 - sqrtRatioAX96));
}
```

#### getLiquidityForAmounts

현재 가격을 기반으로 반환 가능한 최대 유동성을 계산합니다.

* $\sqrt{P}$가 증가할 때 $x$가 소모되므로, 현재 가격이 가격 범위의 최저점보다 낮다면 유동성은 전적으로 $x$ 또는 `amount0`를 기반으로 계산되어야 합니다.
* 반대로, 현재 가격이 가격 범위의 최고점보다 높다면 유동성은 $y$ 또는 `amount1`을 기반으로 계산되어야 합니다.

아래 그림과 같습니다:

$$
p,...,\overbrace{p_a,...,p_b}^{amount0}
$$

$$
\overbrace{p_a,...}^{amount1},p,\overbrace{...,p_b}^{amount0}
$$

$$
\overbrace{p_a,...,p_b}^{amount1},...,p
$$

여기서 $p$는 현재 가격, $p_a$는 범위의 최저점, $p_b$는 범위의 최고점을 나타냅니다.

```solidity
/// @notice Computes the maximum amount of liquidity received for a given amount of token0, token1, the current
/// pool prices and the prices at the tick boundaries
/// @param sqrtRatioX96 A sqrt price representing the current pool prices
/// @param sqrtRatioAX96 A sqrt price representing the first tick boundary
/// @param sqrtRatioBX96 A sqrt price representing the second tick boundary
/// @param amount0 The amount of token0 being sent in
/// @param amount1 The amount of token1 being sent in
/// @return liquidity The maximum amount of liquidity received
function getLiquidityForAmounts(
    uint160 sqrtRatioX96,
    uint160 sqrtRatioAX96,
    uint160 sqrtRatioBX96,
    uint256 amount0,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

    if (sqrtRatioX96 <= sqrtRatioAX96) {
        liquidity = getLiquidityForAmount0(sqrtRatioAX96, sqrtRatioBX96, amount0);
    } else if (sqrtRatioX96 < sqrtRatioBX96) {
        uint128 liquidity0 = getLiquidityForAmount0(sqrtRatioX96, sqrtRatioBX96, amount0);
        uint128 liquidity1 = getLiquidityForAmount1(sqrtRatioAX96, sqrtRatioX96, amount1);

        liquidity = liquidity0 < liquidity1 ? liquidity0 : liquidity1;
    } else {
        liquidity = getLiquidityForAmount1(sqrtRatioAX96, sqrtRatioBX96, amount1);
    }
}
```

### Path.sol

Uniswap v3 [SwapRouter](#SwapRoutersol)에서 거래 경로는 `bytes` 타입 문자열로 인코딩되며, 형식은 다음과 같습니다:

$$
\overbrace{token_0}^{20}\overbrace{fee_0}^{3}\overbrace{token_1}^{20}\overbrace{fee_1}^{3}\overbrace{token_2}^{20}...
$$

여기서 $token_n$의 길이는 20바이트, $fee_n$의 길이는 3바이트입니다. 위 경로는 `token0`에서 `token1`으로 수수료 등급 `fee0`인 풀(`token0`, `token1`, `fee0`)을 사용하여 스왑한 후, 이어서 `token2`로 수수료 등급 `fee1`인 풀(`token1`, `token2`, `fee1`)을 사용하여 스왑함을 의미합니다.

거래 경로 `path`의 예:

$$
0x\overbrace{ca90cf0734d6ccf5ef52e9ec0a515921a67d6013}^{token0,20bytes}\overbrace{0001f4}^{fee,3bytes}\overbrace{68b3465833fb72a70ecdf485e0e4c7bd8665fc45}^{token1,20bytes}
$$

#### hasMultiplePools

거래 경로가 다중 풀(2개 이상)을 거치는지 판단합니다.

```solidity
/// @notice Returns true iff the path contains two or more pools
/// @param path The encoded swap path
/// @return True if path contains two or more pools, otherwise false
function hasMultiplePools(bytes memory path) internal pure returns (bool) {
    return path.length >= MULTIPLE_POOLS_MIN_LENGTH;
}
```

위의 경로 인코딩에서 2개의 풀을 통과한다면 최소 3개의 토큰을 포함해야 하므로, 경로 길이는 최소 $20+3+20+3+20=66$ 바이트여야 합니다. 코드에서 `MULTIPLE_POOLS_MIN_LENGTH`는 66과 같습니다.

#### numPools

경로 내 풀의 개수를 계산합니다.

알고리즘은 다음과 같습니다:

$$
num = \frac{length - 20}{20 + 3}
$$

```solidity
/// @notice Returns the number of pools in the path
/// @param path The encoded swap path
/// @return The number of pools in the path
function numPools(bytes memory path) internal pure returns (uint256) {
    // Ignore the first token address. From then on, every fee and token offset indicates a pool.
    return ((path.length - ADDR_SIZE) / NEXT_OFFSET);
}
```

#### decodeFirstPool

첫 번째 경로의 정보를 디코딩합니다. `token0`, `token1`, `fee`를 포함합니다.

이는 부분 문자열 0-19 (`token0`, `address` 타입으로 변환), 부분 문자열 20-22 (`fee`, `uint24` 타입으로 변환), 부분 문자열 23-42 (`token1`, `address` 타입으로 변환)를 반환합니다. `BytesLib.sol`의 메서드 [toAddress](#toAddress) 및 [toUint24](#toUint24)를 참조하세요.

```solidity
/// @notice Decodes the first pool in path
/// @param path The bytes encoded swap path
/// @return tokenA The first token of the given pool
/// @return tokenB The second token of the given pool
/// @return fee The fee level of the pool
function decodeFirstPool(bytes memory path)
    internal
    pure
    returns (
        address tokenA,
        address tokenB,
        uint24 fee
    )
{
    tokenA = path.toAddress(0);
    fee = path.toUint24(ADDR_SIZE);
    tokenB = path.toAddress(NEXT_OFFSET);
}
```

#### getFirstPool

첫 번째 풀의 경로, 즉 처음 43 (20+3+20) 문자로 구성된 부분 문자열을 반환합니다.

```solidity
/// @notice Gets the segment corresponding to the first pool in the path
/// @param path The bytes encoded swap path
/// @return The segment containing all data necessary to target the first pool in the path
function getFirstPool(bytes memory path) internal pure returns (bytes memory) {
    return path.slice(0, POP_OFFSET);
}
```

#### skipToken

현재 경로에서 첫 번째 `token+fee`를 건너뜁니다. 즉, 처음 20+3 문자를 건너뜁니다.

```solidity
/// @notice Skips a token + fee element from the buffer and returns the remainder
/// @param path The swap path
/// @return The remaining token + fee elements in the path
function skipToken(bytes memory path) internal pure returns (bytes memory) {
    return path.slice(NEXT_OFFSET, path.length - NEXT_OFFSET);
}
```

### BytesLib.sol

#### toAddress

문자열의 지정된 인덱스에서 주소(20 문자)를 읽습니다:

```solidity
function toAddress(bytes memory _bytes, uint256 _start) internal pure returns (address) {
    require(_start + 20 >= _start, 'toAddress_overflow');
    require(_bytes.length >= _start + 20, 'toAddress_outOfBounds');
    address tempAddress;

    assembly {
        tempAddress := div(mload(add(add(_bytes, 0x20), _start)), 0x1000000000000000000000000)
    }

    return tempAddress;
}
```

변수 `_bytes`의 타입이 `bytes`이므로, [ABI 정의](https://docs.soliditylang.org/en/develop/abi-spec.html)에 따라 `bytes`의 처음 32 바이트는 문자열 길이를 저장합니다. 따라서 처음 32 바이트를 건너뛰어야 합니다. 즉 `add(_bytes, 0x20)`입니다; `add(add(_bytes, 0x20), _start)`는 문자열의 `_start` 인덱스로 위치를 지정함을 의미합니다; `mload`는 이 인덱스에서 32 바이트를 읽습니다. `address` 타입은 20 바이트뿐이므로 `div 0x1000000000000000000000000`, 즉 오른쪽으로 12 바이트 시프트해야 합니다.

`_strat = 0`이라고 가정할 때, `_bytes`의 분포는 아래와 같습니다:

$$
0x\overbrace{0000000...2b}^{length,32bytes}\underbrace{\overbrace{ca90cf0734d6ccf5ef52e9ec0a515921a67d6013}^{address, 20 bytes}\overbrace{0001f468b3465833fb72a70e}^{div,12 bytes}}_{mload, 32bytes}cdf485e0e4c7bd8665fc45
$$

#### toUint24

문자열의 지정된 인덱스에서 `uint24` (24비트, 즉 3 문자)를 읽습니다:

```solidity
function toUint24(bytes memory _bytes, uint256 _start) internal pure returns (uint24) {
    require(_start + 3 >= _start, 'toUint24_overflow');
    require(_bytes.length >= _start + 3, 'toUint24_outOfBounds');
    uint24 tempUint;

    assembly {
        tempUint := mload(add(add(_bytes, 0x3), _start))
    }

    return tempUint;
}
```

`_bytes`의 처음 32 문자는 문자열 길이를 나타내므로, `mload`는 32 바이트를 읽고 `_start`부터 시작하는 3 바이트가 읽은 32 바이트의 최하위 위치에 있도록 합니다. 이 값을 `uint24` 타입 변수에 할당하면 최하위 3 바이트만 유지됩니다.

`_start = 0`이라고 가정할 때, `_bytes`의 분포는 아래와 같습니다:

$$
0x\overbrace{000000}^{0x3+\_start}\underbrace{0...2b\overbrace{ca90cf0734d6ccf5ef52e9ec0a515921a67d6013}^{address1,20bytes}\overbrace{0001f4}^{fee,3bytes}}_{mload,32bytes}\overbrace{68b3465833fb72a70ecdf485e0e4c7bd8665fc45}^{address2,20bytes}
$$

### OracleLibrary.sol

백서의 공식 5.3-5.5에 따르면, $t_1$에서 $t_2$까지의 기하 평균 가격은 다음과 같이 계산됩니다:

$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{\sum_{i=t_1}^{t_2} \log_{1.0001}(P_i)}{t_2 - t_1} \quad \text{(5.3)}
$$

$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{a_{t_2} - a_{t_1}}{t_2 - t_1} \quad \text{(5.4)}
$$

$$
P_{t_1,t_2} = 1.0001^{\frac{a_{t_2} - a_{t_1}}{t_2 - t_1}} \quad \text{(5.5)}
$$

이 컨트랙트는 오라클 관련 메서드를 제공하며, 다음을 포함합니다:

* [consult](#consult): 일정 시간 전부터 현재까지의 기하 평균 가격을 조회 (`tick` 형태)
* [getQuoteAtTick](#getQuoteAtTick): `tick`을 기반으로 토큰 가격 계산

#### consult

일정 시간 전부터 현재까지의 기하 평균 가격을 조회합니다 (`tick` 형태).

파라미터는 다음과 같습니다:

* `pool`: 거래 쌍의 풀 주소
* `period`: 초 단위의 기간

반환값:

* `timeWeightedAverageTick`: 시간 가중 평균 가격

```solidity
/// @notice Fetches time-weighted average tick using Uniswap V3 oracle
/// @param pool Address of Uniswap V3 pool that we want to observe
/// @param period Number of seconds in the past to start calculating time-weighted average
/// @return timeWeightedAverageTick The time-weighted average tick from (block.timestamp - period) to block.timestamp
function consult(address pool, uint32 period) internal view returns (int24 timeWeightedAverageTick) {
    require(period != 0, 'BP');

    uint32[] memory secondAgos = new uint32[](2);
    secondAgos[0] = period;
    secondAgos[1] = 0;
```

두 관측 지점을 구성합니다. 첫 번째는 `period` 시간 전이고, 두 번째는 현재입니다.

```solidity
    (int56[] memory tickCumulatives, ) = IUniswapV3Pool(pool).observe(secondAgos);
    int56 tickCumulativesDelta = tickCumulatives[1] - tickCumulatives[0];
```

`IUniswapV3Pool.observe` 메서드에 따라 누적 `tick`, 즉 공식 5.4의 $a_{t_2}$와 $a_{t_1}$을 얻습니다.

$$
tickCumulativesDelta = a_{t_2} - a_{t_1}
$$

```solidity
    timeWeightedAverageTick = int24(tickCumulativesDelta / period);
```

$$
    timeWeightedAverageTick = \frac{tickCumulativesDelta}{t_2 - t_1}
$$

```solidity
    // Always round to negative infinity
    if (tickCumulativesDelta < 0 && (tickCumulativesDelta % period != 0)) timeWeightedAverageTick--;
```

`tickCumulativesDelta`가 음수이고 `period`로 나누어떨어지지 않는 경우, 평균 가격에서 1을 뺍니다.

#### getQuoteAtTick

```solidity
/// @notice Given a tick and a token amount, calculates the amount of token received in exchange
/// @param tick Tick value used to calculate the quote
/// @param baseAmount Amount of token to be converted
/// @param baseToken Address of an ERC20 token contract used as the baseAmount denomination
/// @param quoteToken Address of an ERC20 token contract used as the quoteAmount denomination
/// @return quoteAmount Amount of quoteToken received for baseAmount of baseToken
function getQuoteAtTick(
    int24 tick,
    uint128 baseAmount,
    address baseToken,
    address quoteToken
) internal pure returns (uint256 quoteAmount) {
    uint160 sqrtRatioX96 = TickMath.getSqrtRatioAtTick(tick);

    // Calculate quoteAmount with better precision if it doesn't overflow when multiplied by itself
    if (sqrtRatioX96 <= type(uint128).max) {
        uint256 ratioX192 = uint256(sqrtRatioX96) * sqrtRatioX96;
        quoteAmount = baseToken < quoteToken
            ? FullMath.mulDiv(ratioX192, baseAmount, 1 << 192)
            : FullMath.mulDiv(1 << 192, baseAmount, ratioX192);
    } else {
        uint256 ratioX128 = FullMath.mulDiv(sqrtRatioX96, sqrtRatioX96, 1 << 64);
        quoteAmount = baseToken < quoteToken
            ? FullMath.mulDiv(ratioX128, baseAmount, 1 << 128)
            : FullMath.mulDiv(1 << 128, baseAmount, ratioX128);
    }
}
```

Uniswap-v3-core `getSqrtRatioAtTick` 메서드를 기반으로 `tick`에 해당하는 $\sqrt{P}$, 즉 $\sqrt{\frac{token1}{token0}}$을 계산합니다.

`baseToken < quoteToken`이면, `baseToken`은 `token0`이고 `quoteToken`은 `token1`입니다:

$$
quoteAmount = baseAmount \cdot (\sqrt{P})^2
$$

그렇지 않으면 `baseToken`은 `token1`이고 `quoteToken`은 `token0`입니다:

$$
quoteAmount = \frac{baseAmount}{(\sqrt{P})^2}
$$
