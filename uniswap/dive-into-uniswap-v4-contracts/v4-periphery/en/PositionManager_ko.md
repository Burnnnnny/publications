# PositionManager

PositionManager는 포지션 생성, 유동성 수정, 포지션 삭제 및 기타 작업을 포함한 포지션 관리에 사용됩니다.

PositionManager 컨트랙트는 주로 다음 인터페이스를 포함합니다:

- [initializePool](#initializepool): 풀 초기화
- [modifyLiquidities](#modifyliquidities): 유동성 수정
- [modifyLiquiditiesWithoutUnlock](#modifyliquiditieswithoutunlock): 유동성 수정 (잠금 해제 없이)

[Uniswap v4 워크플로우](../../assets/uniswap-v4-workflow.png) 다이어그램에 따르면, `modifyLiquidities`와 `modifyLiquiditiesWithoutUnlock` 모두 [BaseActionsRouter._executeActionsWithoutUnlock](./BaseActionsRouter_ko.md#_executeactionswithoutunlock)을 통해 [_handleAction](#_handleaction) 메서드를 실행하며, 이는 사용자가 지정한 다양한 작업을 실행합니다.

PositionManager는 주로 다음 두 가지 유형의 작업을 포함합니다:

* 유동성 수정 작업
  - [INCREASE_LIQUIDITY](#increase_liquidity): 유동성 증가
  - [INCREASE_LIQUIDITY_FROM_DELTAS](#increase_liquidity_from_deltas): 플래시 어카운팅 잔액을 사용하여 유동성 증가
  - [DECREASE_LIQUIDITY](#decrease_liquidity): 유동성 감소
  - [MINT_POSITION](#mint_position): 포지션 생성
  - [MINT_POSITION_FROM_DELTAS](#mint_position_from_deltas): 플래시 어카운팅 잔액을 사용하여 포지션 생성
  - [BURN_POSITION](#burn_position): 포지션 소각

* 잔액 정산 작업
  - [SETTLE_PAIR](#settle_pair): 토큰 쌍의 부채 정산, 호출자가 `PoolManager` 컨트랙트에 토큰 지불
  - [TAKE_PAIR](#take_pair): 토큰 쌍의 잔액 인출
  - [SETTLE](#settle): 단일 토큰 부채 정산, `PoolManager` 컨트랙트에 토큰 지불
  - [TAKE](#take): 단일 토큰 잔액 인출
  - [CLOSE_CURRENCY](#close_currency): 단일 토큰 잔액 정산 또는 인출
  - [CLEAR_OR_TAKE](#clear_or_take): 단일 토큰 잔액 포기 또는 인출
  - [SWEEP](#sweep): PositionManager에서 단일 토큰 잔액 인출

## 함수 정의 (Method Definitions)

### initializePool
풀을 초기화합니다.

```solidity
/// @inheritdoc IPoolInitializer_v4
function initializePool(PoolKey calldata key, uint160 sqrtPriceX96) external payable returns (int24) {
    try poolManager.initialize(key, sqrtPriceX96) returns (int24 tick) {
        return tick;
    } catch {
        return type(int24).max;
    }
}
```

[PoolManager.initialize](../../v4-core/en/PoolManager_ko.md#initialize) 메서드를 호출하여 풀을 초기화합니다.

### modifyLiquidities

PositionManager 컨트랙트는 커맨드 라인 디자인 철학을 채택합니다. 각 작업에 대한 인터페이스를 별도로 제공하는 대신, 사용자가 [modifyLiquidities](#modifyLiquidities)를 통해 다양한 작업 명령을 조합하고 순차적으로 실행할 수 있도록 합니다.

이 메서드는 PositionManager의 표준 진입점입니다.

```solidity
/// @inheritdoc IPositionManager
function modifyLiquidities(bytes calldata unlockData, uint256 deadline)
    external
    payable
    isNotLocked
    checkDeadline(deadline)
{
    _executeActions(unlockData);
}
```

`PositionManager`는 `BaseActionsRouter`를 상속하며, `modifyLiquidities`는 `deadline`만 확인하고 [_executeActions](./BaseActionsRouter_ko.md#_executeactions) 메서드를 호출합니다.

### modifyLiquiditiesWithoutUnlock

`modifyLiquiditiesWithoutUnlock` 메서드는 `modifyLiquidities` 메서드와 유사하지만 잠금 해제 작업을 수행하지 않습니다.

외부 컨트랙트는 [PoolManager.unlock](../../v4-core/en/PoolManager_ko.md#unlock) 메서드가 호출되어 잠금이 해제되었는지 확인해야 합니다.

```solidity
/// @inheritdoc IPositionManager
function modifyLiquiditiesWithoutUnlock(bytes calldata actions, bytes[] calldata params)
    external
    payable
    isNotLocked
{
    _executeActionsWithoutUnlock(actions, params);
}
```

### _handleAction

[BaseActionsRouter](./BaseActionsRouter_ko.md)를 상속하는 모든 컨트랙트는 `_handleAction` 메서드를 구현하여 사용자가 전달한 Action 및 매개변수를 기반으로 특정 작업을 실행해야 합니다.

```solidity
function _handleAction(uint256 action, bytes calldata params) internal virtual override {
    if (action < Actions.SETTLE) {
        if (action == Actions.INCREASE_LIQUIDITY) {
            (uint256 tokenId, uint256 liquidity, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData) =
                params.decodeModifyLiquidityParams();
            _increase(tokenId, liquidity, amount0Max, amount1Max, hookData);
            return;
        } else if (action == Actions.INCREASE_LIQUIDITY_FROM_DELTAS) {
            (uint256 tokenId, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData) =
                params.decodeIncreaseLiquidityFromDeltasParams();
            _increaseFromDeltas(tokenId, amount0Max, amount1Max, hookData);
            return;
        } else if (action == Actions.DECREASE_LIQUIDITY) {
            (uint256 tokenId, uint256 liquidity, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData) =
                params.decodeModifyLiquidityParams();
            _decrease(tokenId, liquidity, amount0Min, amount1Min, hookData);
            return;
        } else if (action == Actions.MINT_POSITION) {
            (
                PoolKey calldata poolKey,
                int24 tickLower,
                int24 tickUpper,
                uint256 liquidity,
                uint128 amount0Max,
                uint128 amount1Max,
                address owner,
                bytes calldata hookData
            ) = params.decodeMintParams();
            _mint(poolKey, tickLower, tickUpper, liquidity, amount0Max, amount1Max, _mapRecipient(owner), hookData);
            return;
        } else if (action == Actions.MINT_POSITION_FROM_DELTAS) {
            (
                PoolKey calldata poolKey,
                int24 tickLower,
                int24 tickUpper,
                uint128 amount0Max,
                uint128 amount1Max,
                address owner,
                bytes calldata hookData
            ) = params.decodeMintFromDeltasParams();
            _mintFromDeltas(poolKey, tickLower, tickUpper, amount0Max, amount1Max, _mapRecipient(owner), hookData);
            return;
        } else if (action == Actions.BURN_POSITION) {
            // Will automatically decrease liquidity to 0 if the position is not already empty.
            (uint256 tokenId, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData) =
                params.decodeBurnParams();
            _burn(tokenId, amount0Min, amount1Min, hookData);
            return;
        }
    } else {
        if (action == Actions.SETTLE_PAIR) {
            (Currency currency0, Currency currency1) = params.decodeCurrencyPair();
            _settlePair(currency0, currency1);
            return;
        } else if (action == Actions.TAKE_PAIR) {
            (Currency currency0, Currency currency1, address recipient) = params.decodeCurrencyPairAndAddress();
            _takePair(currency0, currency1, _mapRecipient(recipient));
            return;
        } else if (action == Actions.SETTLE) {
            (Currency currency, uint256 amount, bool payerIsUser) = params.decodeCurrencyUint256AndBool();
            _settle(currency, _mapPayer(payerIsUser), _mapSettleAmount(amount, currency));
            return;
        } else if (action == Actions.TAKE) {
            (Currency currency, address recipient, uint256 amount) = params.decodeCurrencyAddressAndUint256();
            _take(currency, _mapRecipient(recipient), _mapTakeAmount(amount, currency));
            return;
        } else if (action == Actions.CLOSE_CURRENCY) {
            Currency currency = params.decodeCurrency();
            _close(currency);
            return;
        } else if (action == Actions.CLEAR_OR_TAKE) {
            (Currency currency, uint256 amountMax) = params.decodeCurrencyAndUint256();
            _clearOrTake(currency, amountMax);
            return;
        } else if (action == Actions.SWEEP) {
            (Currency currency, address to) = params.decodeCurrencyAndAddress();
            _sweep(currency, _mapRecipient(to));
            return;
        } else if (action == Actions.WRAP) {
            uint256 amount = params.decodeUint256();
            _wrap(_mapWrapUnwrapAmount(CurrencyLibrary.ADDRESS_ZERO, amount, Currency.wrap(address(WETH9))));
            return;
        } else if (action == Actions.UNWRAP) {
            uint256 amount = params.decodeUint256();
            _unwrap(_mapWrapUnwrapAmount(Currency.wrap(address(WETH9)), amount, CurrencyLibrary.ADDRESS_ZERO));
            return;
        }
    }
    revert UnsupportedAction(action);
}
```

[ActionsLibrary](./ActionsLibrary_ko.md)는 모든 `actions`를 숫자 순서대로 정의합니다:

* `Actions.SETTLE`보다 작은 작업은 유동성 수정 작업입니다.
* `Actions.SETTLE` 이상의 작업은 잔액 정산(회계) 작업입니다.

각 Action에 대한 로직은 아래에 소개되어 있습니다:

#### INCREASE_LIQUIDITY

유동성 증가.

```solidity
(uint256 tokenId, uint256 liquidity, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData) =
    params.decodeModifyLiquidityParams();
_increase(tokenId, liquidity, amount0Max, amount1Max, hookData);
return;
```

매개변수를 디코딩하고 [_increase](#_increase) 메서드를 호출합니다.

#### INCREASE_LIQUIDITY_FROM_DELTAS

플래시 어카운팅 잔액을 사용하여 유동성 증가.

```solidity
(uint256 tokenId, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData) =
    params.decodeIncreaseLiquidityFromDeltasParams();
_increaseFromDeltas(tokenId, amount0Max, amount1Max, hookData);
return;
```

매개변수를 디코딩하고 [_increaseFromDeltas](#_increasefromdeltas) 메서드를 호출합니다.

#### DECREASE_LIQUIDITY

유동성 감소.

```solidity
(uint256 tokenId, uint256 liquidity, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData) =
    params.decodeModifyLiquidityParams();
_decrease(tokenId, liquidity, amount0Min, amount1Min, hookData);
return;
```

매개변수를 디코딩하고 [_decrease](#_decrease) 메서드를 호출합니다.

#### MINT_POSITION

포지션 생성.

```solidity
(
    PoolKey calldata poolKey,
    int24 tickLower,
    int24 tickUpper,
    uint256 liquidity,
    uint128 amount0Max,
    uint128 amount1Max,
    address owner,
    bytes calldata hookData
) = params.decodeMintParams();
_mint(poolKey, tickLower, tickUpper, liquidity, amount0Max, amount1Max, _mapRecipient(owner), hookData);
return;
```

매개변수를 디코딩하고 [_mint](#_mint) 메서드를 호출합니다.

#### MINT_POSITION_FROM_DELTAS

플래시 어카운팅 잔액을 사용하여 포지션 생성.

```solidity
(
    PoolKey calldata poolKey,
    int24 tickLower,
    int24 tickUpper,
    uint128 amount0Max,
    uint128 amount1Max,
    address owner,
    bytes calldata hookData
) = params.decodeMintFromDeltasParams();
_mintFromDeltas(poolKey, tickLower, tickUpper, amount0Max, amount1Max, _mapRecipient(owner), hookData);
return;
```

매개변수를 디코딩하고 [_mintFromDeltas](#_mintfromdeltas) 메서드를 호출합니다.

여기서 [_mapRecipient](./BaseActionsRouter_ko.md#_maprecipient) 메서드는 `owner` 주소를 계산하는 데 사용됩니다.

#### BURN_POSITION

포지션 소각.

```solidity
// Will automatically decrease liquidity to 0 if the position is not already empty.
(uint256 tokenId, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData) =
    params.decodeBurnParams();
_burn(tokenId, amount0Min, amount1Min, hookData);
return;
```

매개변수를 디코딩하고 [_burn](#_burn) 메서드를 호출합니다.

#### SETTLE_PAIR

토큰 쌍의 부채 정산, 호출자가 `PoolManager` 컨트랙트에 토큰 지불.

```solidity
(Currency currency0, Currency currency1) = params.decodeCurrencyPair();
_settlePair(currency0, currency1);
return;
```

매개변수를 디코딩하고 [_settlePair](#_settlepair) 메서드를 호출합니다.

#### TAKE_PAIR

토큰 쌍의 잔액 인출, `PoolManager`가 `recipient`에게 토큰 지불.

```solidity
(Currency currency0, Currency currency1, address recipient) = params.decodeCurrencyPairAndAddress();
_takePair(currency0, currency1, _mapRecipient(recipient));
return;
```

매개변수를 디코딩하고 [_takePair](#_takepair) 메서드를 호출합니다.

[_mapRecipient](./BaseActionsRouter_ko.md#_maprecipient) 메서드를 사용하여 `recipient` 주소를 계산합니다.

#### SETTLE

단일 토큰 부채 정산, 호출자가 `PoolManager` 컨트랙트에 토큰 지불.

```solidity
(Currency currency, uint256 amount, bool payerIsUser) = params.decodeCurrencyUint256AndBool();
_settle(currency, _mapPayer(payerIsUser), _mapSettleAmount(amount, currency));
return;
```

매개변수를 디코딩하고, [_mapPayer](./BaseActionsRouter_ko.md#_mappayer) 메서드를 사용하여 지불자를 결정하고, [_mapSettleAmount](./DeltaResolver_ko.md#_mapsettleamount) 메서드를 사용하여 정산할 토큰 양을 계산한 후 최종적으로 [_settle](./DeltaResolver_ko.md#_settle) 메서드를 호출합니다.

#### TAKE

단일 토큰 잔액 인출, `PoolManager`가 `recipient`에게 토큰 지불.

```solidity
(Currency currency, address recipient, uint256 amount) = params.decodeCurrencyAddressAndUint256();
_take(currency, _mapRecipient(recipient), _mapTakeAmount(amount, currency));
return;
```

매개변수를 디코딩하고 [_take](./DeltaResolver_ko.md#_take) 메서드를 호출합니다.

[_mapRecipient](./BaseActionsRouter_ko.md#_maprecipient) 메서드를 사용하여 `recipient` 주소를 계산합니다.
[_mapTakeAmount](./DeltaResolver_ko.md#_maptakeamount) 메서드를 사용하여 인출할 토큰 양을 계산합니다.

#### CLOSE_CURRENCY

단일 토큰 잔액 정산 또는 인출.

정산할지 인출할지 확실하지 않은 경우 `CLOSE_CURRENCY`를 사용하여 지정된 토큰의 포지션을 닫을 수 있습니다.

```solidity
Currency currency = params.decodeCurrency();
_close(currency);
return;
```

매개변수를 디코딩하고 [_close](#_close) 메서드를 호출합니다.

#### CLEAR_OR_TAKE

단일 토큰 잔액 포기 또는 인출.

잔액이 `amountMax` 이하이면 토큰을 포기하고, 그렇지 않으면 토큰을 인출합니다.

```solidity
(Currency currency, uint256 amountMax) = params.decodeCurrencyAndUint256();
_clearOrTake(currency, amountMax);
return;
```

매개변수를 디코딩하고 [_clearOrTake](#_clearortake) 메서드를 호출합니다.

#### SWEEP

`PositionManager`에서 단일 토큰 잔액 인출, `PositionManager`가 `recipient`에게 토큰 지불.

```solidity
(Currency currency, address to) = params.decodeCurrencyAndAddress();
_sweep(currency, _mapRecipient(to));
return;
```

매개변수를 디코딩하고 [_sweep](#_sweep) 메서드를 호출합니다.

[_mapRecipient](./BaseActionsRouter_ko.md#_maprecipient) 메서드를 사용하여 `to` 주소를 계산합니다.

#### WRAP

토큰 래핑, `PositionManager` 컨트랙트의 `ETH`를 `WETH`로 변환.

```solidity
uint256 amount = params.decodeUint256();
_wrap(_mapWrapUnwrapAmount(CurrencyLibrary.ADDRESS_ZERO, amount, Currency.wrap(address(WETH9))));
return;
```

매개변수를 디코딩하고 [_wrap](#_wrap) 메서드를 호출합니다.

[_mapWrapUnwrapAmount](./DeltaResolver_ko.md#_mapwrapunwrapamount) 메서드를 사용하여 래핑할 토큰 양을 계산합니다.

#### UNWRAP

토큰 언래핑, `PositionManager` 컨트랙트의 `WETH`를 `ETH`로 변환.

```solidity
uint256 amount = params.decodeUint256();
_unwrap(_mapWrapUnwrapAmount(Currency.wrap(address(WETH9)), amount, CurrencyLibrary.ADDRESS_ZERO));
return;
```

매개변수를 디코딩하고 [_unwrap](#_unwrap) 메서드를 호출합니다.

[_mapWrapUnwrapAmount](./DeltaResolver_ko.md#_mapwrapunwrapamount) 메서드를 사용하여 언래핑할 토큰 양을 계산합니다.

### _increase

포지션의 유동성 증가.

입력 매개변수:

- `tokenId`: 포지션 ID, 즉 PositionManager가 할당한 ERC721 토큰 ID
- `liquidity`: 증가시킬 유동성 양
- `amount0Max`: 제공할 `token0`의 최대 양
- `amount1Max`: 제공할 `token1`의 최대 양
- `hookData`: Hook 데이터, `beforeModifyLiquidity` 및 `afterModifyLiquidity` Hooks 함수에 전달하는 데 사용됨

```solidity
/// @dev Calling increase with 0 liquidity will credit the caller with any underlying fees of the position
function _increase(
    uint256 tokenId,
    uint256 liquidity,
    uint128 amount0Max,
    uint128 amount1Max,
    bytes calldata hookData
) internal onlyIfApproved(msgSender(), tokenId) {
    (PoolKey memory poolKey, PositionInfo info) = getPoolAndPositionInfo(tokenId);

    // Note: The tokenId is used as the salt for this position, so every minted position has unique storage in the pool manager.
    (BalanceDelta liquidityDelta, BalanceDelta feesAccrued) =
        _modifyLiquidity(info, poolKey, liquidity.toInt256(), bytes32(tokenId), hookData);
    // Slippage checks should be done on the principal liquidityDelta which is the liquidityDelta - feesAccrued
    (liquidityDelta - feesAccrued).validateMaxIn(amount0Max, amount1Max);
}
```

`getPoolAndPositionInfo`를 통해 풀 정보와 포지션 정보를 가져옵니다.

[_modifyLiquidity](#_modifyliquidity) 메서드를 호출하여 유동성 수정을 완료합니다. `liquidityDelta`와 `feesAccrued`를 반환합니다. 그 중에서:

* `liquidityDelta`는 호출자가 정산해야 할 유동성 변경 값이며, 모두 양이 아님(음수 또는 0);
* `feesAccrued`는 포지션이 현재 시점까지 수취할 수 있는 수수료이며, 모두 음이 아님(양수 또는 0).

`liquidityDelta`는 이미 `feeAccrued`를 포함하고 있고 `amount0Max`와 `amount1Max`는 수수료를 포함하지 않기 때문에(유동성 자체만 계산), 최대값 확인을 위해 입력 매개변수로 `liquidityDelta - feesAccrued`를 사용해야 합니다.

> 참고: `liquidityDelta`는 호출자가 지불해야 할 토큰 `amount0`과 `amount1`의 최종 금액이며 음수로 표시됩니다.

### _increaseFromDeltas

이 메서드는 [_increase](#_increase) 메서드와 유사합니다. 차이점은 `_increaseFromDeltas`는 특정 유동성 양을 지정할 필요가 없지만 `PoolManager`에서 풀의 두 토큰에 대한 `PositionManager` 컨트랙트의 현재 플래시 어카운팅 잔액을 조회하여 증가시킬 유동성 양을 계산한다는 것입니다.

마지막으로 [_modifyLiquidity](#_modifyliquidity) 메서드를 호출하여 유동성 수정을 완료합니다.

```solidity
/// @dev The liquidity delta is derived from open deltas in the pool manager.
function _increaseFromDeltas(uint256 tokenId, uint128 amount0Max, uint128 amount1Max, bytes calldata hookData)
    internal
    onlyIfApproved(msgSender(), tokenId)
{
    (PoolKey memory poolKey, PositionInfo info) = getPoolAndPositionInfo(tokenId);

    uint256 liquidity;
    {
        (uint160 sqrtPriceX96,,,) = poolManager.getSlot0(poolKey.toId());

        // Use the credit on the pool manager as the amounts for the mint.
        liquidity = LiquidityAmounts.getLiquidityForAmounts(
            sqrtPriceX96,
            TickMath.getSqrtPriceAtTick(info.tickLower()),
            TickMath.getSqrtPriceAtTick(info.tickUpper()),
            _getFullCredit(poolKey.currency0),
            _getFullCredit(poolKey.currency1)
        );
    }

    // Note: The tokenId is used as the salt for this position, so every minted position has unique storage in the pool manager.
    (BalanceDelta liquidityDelta, BalanceDelta feesAccrued) =
        _modifyLiquidity(info, poolKey, liquidity.toInt256(), bytes32(tokenId), hookData);
    // Slippage checks should be done on the principal liquidityDelta which is the liquidityDelta - feesAccrued
    (liquidityDelta - feesAccrued).validateMaxIn(amount0Max, amount1Max);
}
```

[_getFullCredit](./DeltaResolver_ko.md#_getfullcredit) 메서드는 `PoolManager`에서 `PositionManager`의 플래시 어카운팅 잔액이 항상 음이 아닌지 확인합니다. 즉, 인출할 수 있는 토큰이 있음을 의미합니다. 그렇지 않으면 트랜잭션이 롤백됩니다.

### _decrease

포지션의 유동성 감소.

입력 매개변수:

- `tokenId`: 포지션 ID, 즉 PositionManager가 할당한 ERC721 토큰 ID
- `liquidity`: 감소시킬 유동성 양
- `amount0Min`: 수령할 `token0`의 최소 양
- `amount1Min`: 수령할 `token1`의 최소 양
- `hookData`: Hook 데이터, `beforeModifyLiquidity` 및 `afterModifyLiquidity` Hooks 함수에 전달하는 데 사용됨

```solidity
/// @dev Calling decrease with 0 liquidity will credit the caller with any underlying fees of the position
function _decrease(
    uint256 tokenId,
    uint256 liquidity,
    uint128 amount0Min,
    uint128 amount1Min,
    bytes calldata hookData
) internal onlyIfApproved(msgSender(), tokenId) {
    (PoolKey memory poolKey, PositionInfo info) = getPoolAndPositionInfo(tokenId);

    // Note: the tokenId is used as the salt.
    (BalanceDelta liquidityDelta, BalanceDelta feesAccrued) =
        _modifyLiquidity(info, poolKey, -(liquidity.toInt256()), bytes32(tokenId), hookData);
    // Slippage checks should be done on the principal liquidityDelta which is the liquidityDelta - feesAccrued
    (liquidityDelta - feesAccrued).validateMinOut(amount0Min, amount1Min);
}
```

`getPoolAndPositionInfo`를 통해 풀 정보와 포지션 정보를 가져옵니다.

[_modifyLiquidity](#_modifyliquidity) 메서드를 호출하여 유동성 수정을 완료합니다. `liquidityDelta`와 `feesAccrued`를 반환합니다. 그 중에서:

* `liquidityDelta`는 호출자가 인출해야 할 유동성 변경 값이며, 모두 음이 아님(양수 또는 0);
* `feesAccrued`는 포지션이 현재 시점까지 수취할 수 있는 수수료이며, 모두 음이 아님(양수 또는 0).

`liquidityDelta`는 이미 `feeAccrued`를 포함하고 있고 `amount0Min`과 `amount1Min`는 수수료를 포함하지 않기 때문에(유동성 자체만 계산), 최소값 확인을 위해 입력 매개변수로 `liquidityDelta - feesAccrued`를 사용해야 합니다.

> 참고: `liquidityDelta`는 호출자가 인출할 수 있는 토큰의 최종 금액이며 양수 또는 0으로 표시됩니다.

Uniswap v4는 수수료를 인출하는 직접적인 메서드를 제공하지 않으므로, `liquidity`, `amount0Min`, `amount1Min`을 0으로 설정하여 `DECRESAE_LIQUIDITY` 작업을 사용하여 수수료를 인출할 수 있습니다.

### _mint

포지션 생성.

입력 매개변수:

- `poolKey`: 풀의 키
- `tickLower`: 포지션의 하한 틱
- `tickUpper`: 포지션의 상한 틱
- `liquidity`: 생성할 유동성 양
- `amount0Max`: 제공할 `token0`의 최대 양
- `amount1Max`: 제공할 `token1`의 최대 양
- `owner`: 포지션의 소유자
- `hookData`: Hook 데이터, `beforeModifyLiquidity` 및 `afterModifyLiquidity` Hooks 함수에 전달하는 데 사용됨

```solidity
function _mint(
    PoolKey calldata poolKey,
    int24 tickLower,
    int24 tickUpper,
    uint256 liquidity,
    uint128 amount0Max,
    uint128 amount1Max,
    address owner,
    bytes calldata hookData
) internal {
    // mint receipt token
    uint256 tokenId;
    // tokenId is assigned to current nextTokenId before incrementing it
    unchecked {
        tokenId = nextTokenId++;
    }
    _mint(owner, tokenId);

    // Initialize the position info
    PositionInfo info = PositionInfoLibrary.initialize(poolKey, tickLower, tickUpper);
    positionInfo[tokenId] = info;

    // Store the poolKey if it is not already stored.
    // On UniswapV4, the minimum tick spacing is 1, which means that if the tick spacing is 0, the pool key has not been set.
    bytes25 poolId = info.poolId();
    if (poolKeys[poolId].tickSpacing == 0) {
        poolKeys[poolId] = poolKey;
    }

    // fee delta can be ignored as this is a new position
    (BalanceDelta liquidityDelta,) =
        _modifyLiquidity(info, poolKey, liquidity.toInt256(), bytes32(tokenId), hookData);
    liquidityDelta.validateMaxIn(amount0Max, amount1Max);
}
```

ERC721 토큰 ID `tokenId`를 할당하고 `_mint` 메서드를 호출하여 `owner`를 위한 포지션을 나타내는 ERC721 토큰을 발행합니다.

포지션 정보를 초기화합니다. `positionInfo`는 `tokenId`와 포지션 정보의 매핑을 나타냅니다.
`poolKey`가 저장되지 않은 경우 `poolKey`를 저장합니다.
따라서 나중에 `tokenId`를 기반으로 풀 정보 `poolKey`와 포지션 정보 `positionInfo`를 반환할 수 있습니다.

[_modifyLiquidity](#_modifyliquidity) 메서드를 호출하여 유동성 수정을 완료합니다. `liquidityDelta`와 `feesAccrued`를 반환합니다. 이때 `freeAccrued`는 0이므로 무시할 수 있습니다. `liquidityDelta`가 `amount0Max`와 `amount1Max`를 초과하는지 확인합니다.

### _mintFromDeltas

플래시 어카운팅 잔액을 사용하여 포지션 생성.

입력 매개변수:

- `poolKey`: 풀의 키
- `tickLower`: 포지션의 하한 틱
- `tickUpper`: 포지션의 상한 틱
- `amount0Max`: 제공할 `token0`의 최대 양
- `amount1Max`: 제공할 `token1`의 최대 양
- `owner`: 포지션의 소유자
- `hookData`: Hook 데이터, `beforeModifyLiquidity` 및 `afterModifyLiquidity` Hooks 함수에 전달하는 데 사용됨

`_mint` 메서드와 유사하지만, 차이점은 `_mintFromDeltas`는 특정 유동성 양을 지정할 필요가 없지만 `PoolManager`에서 풀의 두 토큰에 대한 `PositionManager` 컨트랙트의 현재 플래시 어카운팅 잔액을 조회하여 증가시킬 유동성 양을 계산한다는 것입니다.

마지막으로 [_mint](#_mint) 메서드를 호출하여 포지션을 생성합니다.

```solidity
function _mintFromDeltas(
    PoolKey calldata poolKey,
    int24 tickLower,
    int24 tickUpper,
    uint128 amount0Max,
    uint128 amount1Max,
    address owner,
    bytes calldata hookData
) internal {
    (uint160 sqrtPriceX96,,,) = poolManager.getSlot0(poolKey.toId());

    // Use the credit on the pool manager as the amounts for the mint.
    uint256 liquidity = LiquidityAmounts.getLiquidityForAmounts(
        sqrtPriceX96,
        TickMath.getSqrtPriceAtTick(tickLower),
        TickMath.getSqrtPriceAtTick(tickUpper),
        _getFullCredit(poolKey.currency0),
        _getFullCredit(poolKey.currency1)
    );

    _mint(poolKey, tickLower, tickUpper, liquidity, amount0Max, amount1Max, owner, hookData);
}
```

### _burn

포지션 소각.

입력 매개변수:

- `tokenId`: 포지션 ID, 즉 PositionManager가 할당한 ERC721 토큰 ID
- `amount0Min`: 수령할 `token0`의 최소 양
- `amount1Min`: 수령할 `token1`의 최소 양
- `hookData`: Hook 데이터, `beforeModifyLiquidity` 및 `afterModifyLiquidity` Hooks 함수에 전달하는 데 사용됨

```solidity
/// @dev this is overloaded with ERC721Permit_v4._burn
function _burn(uint256 tokenId, uint128 amount0Min, uint128 amount1Min, bytes calldata hookData)
    internal
    onlyIfApproved(msgSender(), tokenId)
{
    (PoolKey memory poolKey, PositionInfo info) = getPoolAndPositionInfo(tokenId);

    uint256 liquidity = uint256(_getLiquidity(tokenId, poolKey, info.tickLower(), info.tickUpper()));

    address owner = ownerOf(tokenId);

    // Clear the position info.
    positionInfo[tokenId] = PositionInfoLibrary.EMPTY_POSITION_INFO;
    // Burn the token.
    _burn(tokenId);

    // Can only call modify if there is non zero liquidity.
    BalanceDelta feesAccrued;
    if (liquidity > 0) {
        BalanceDelta liquidityDelta;
        // do not use _modifyLiquidity as we do not need to notify on modification for burns.
        IPoolManager.ModifyLiquidityParams memory params = IPoolManager.ModifyLiquidityParams({
            tickLower: info.tickLower(),
            tickUpper: info.tickUpper(),
            liquidityDelta: -(liquidity.toInt256()),
            salt: bytes32(tokenId)
        });
        (liquidityDelta, feesAccrued) = poolManager.modifyLiquidity(poolKey, params, hookData);
        // Slippage checks should be done on the principal liquidityDelta which is the liquidityDelta - feesAccrued
        (liquidityDelta - feesAccrued).validateMinOut(amount0Min, amount1Min);
    }

    // deletes then notifies the subscriber
    if (info.hasSubscriber()) _removeSubscriberAndNotifyBurn(tokenId, owner, info, liquidity, feesAccrued);
}
```

`getPoolAndPositionInfo`를 통해 풀 정보와 포지션 정보를 가져옵니다.

`PoolManager`를 통해 포지션의 현재 유동성을 조회합니다.

포지션 정보를 지우고 ERC721 토큰을 소각합니다.

유동성이 0보다 크면 [poolManager.modifyLiquidity](../../v4-core/en/PoolManager_ko.md#modifyliquidity) 메서드를 호출하여 포지션의 모든 유동성을 제거합니다. `liquidityDelta`와 `feesAccrued`를 반환합니다.

`liquidityDelta`는 이미 `feeAccrued`를 포함하고 있고 `amount0Min`과 `amount1Min`는 수수료를 포함하지 않기 때문에(유동성 자체만 계산), 최소값 확인을 위해 입력 매개변수로 `liquidityDelta - feesAccrued`를 사용해야 합니다.

포지션에 구독자가 있는 경우 `_removeSubscriberAndNotifyBurn` 메서드를 호출하여 구독자에게 알립니다.

포지션이 소각된 후 호출자는 토큰을 받지 않는다는 점에 유의하세요. 이 토큰들은 `PoolManager`의 플래시 어카운팅 잔액에 기록됩니다; 호출자는 [TAKE_PAIR](#take_pair)와 같은 작업을 결합하여 토큰을 자신의 계정으로 인출해야 합니다.

### _settlePair

토큰 쌍의 부채 정산. 호출자는 `PoolManager`에 토큰을 지불해야 합니다.

입력 매개변수:

- `currency0`: 토큰 0
- `currency1`: 토큰 1

```solidity
function _settlePair(Currency currency0, Currency currency1) internal {
    // the locker is the payer when settling
    address caller = msgSender();
    _settle(currency0, caller, _getFullDebt(currency0));
    _settle(currency1, caller, _getFullDebt(currency1));
}
```

[_settle](./DeltaResolver_ko.md#_settle) 메서드를 호출하여 토큰 0과 토큰 1의 잔액을 정산합니다. 이때 `PoolManager`의 플래시 어카운팅 잔액은 비양수여야 하며, 이는 호출자가 토큰을 지불해야 함을 의미합니다.

[_getFullDebt](./DeltaResolver_ko.md#_getfulldebt) 메서드를 사용하여 `PoolManager`에서 `PositionManager`의 부채를 가져옵니다.

### _takePair

토큰 쌍의 잔액 인출, `PoolManager`가 `recipient`에게 토큰 지불.

```solidity
function _takePair(Currency currency0, Currency currency1, address recipient) internal {
    _take(currency0, recipient, _getFullCredit(currency0));
    _take(currency1, recipient, _getFullCredit(currency1));
}
```

[_take](./DeltaResolver_ko.md#_take) 메서드를 호출하여 토큰 0과 토큰 1의 잔액을 인출합니다. [_getFullCredit](./DeltaResolver_ko.md#_getfullcredit) 메서드에 따르면 `PoolManager`의 플래시 어카운팅 잔액은 비음수여야 하며, 이는 호출자가 토큰을 인출할 수 있음을 의미합니다.

### _close

단일 토큰 잔액 정산 또는 인출. `currencyDelta`의 부호에 따라 부채를 정산할지 잔액을 인출할지 결정합니다.

```solidity
function _close(Currency currency) internal {
    // this address has applied all deltas on behalf of the user/owner
    // it is safe to close this entire delta because of slippage checks throughout the batched calls.
    int256 currencyDelta = poolManager.currencyDelta(address(this), currency);

    // the locker is the payer or receiver
    address caller = msgSender();
    if (currencyDelta < 0) {
        // Casting is safe due to limits on the total supply of a pool
        _settle(currency, caller, uint256(-currencyDelta));
    } else {
        _take(currency, caller, uint256(currencyDelta));
    }
}
```

[poolManager.currencyDelta](../../v4-core/en/PoolManager_ko.md#currencydelta) 메서드를 호출하여 `PoolManager`에서 `PositionManager`의 플래시 어카운팅 잔액을 가져옵니다.

`currencyDelta`가 0보다 작으면 `PositionManager`가 토큰을 빚지고 있음을 의미하므로 [_settle](./DeltaResolver_ko.md#_settle) 메서드를 호출하여 `PoolManager`에 토큰을 지불하고 `payer`는 호출자가 됩니다.

그렇지 않으면 [_take](./DeltaResolver_ko.md#_take) 메서드를 호출하여 `PoolManager`에서 토큰을 인출하고 토큰 수령인은 호출자가 됩니다.

### _clearOrTake

단일 토큰 잔액 포기 또는 인출.

입력 매개변수:

- `currency`: 토큰
- `amountMax`: 최대 금액. 이 값 이하이면 토큰을 포기하고, 그렇지 않으면 토큰을 인출합니다.

```solidity
/// @dev integrators may elect to forfeit positive deltas with clear
/// if the forfeit amount exceeds the user-specified max, the amount is taken instead
/// if there is no credit, no call is made.
function _clearOrTake(Currency currency, uint256 amountMax) internal {
    uint256 delta = _getFullCredit(currency);
    if (delta == 0) return;

    // forfeit the delta if its less than or equal to the user-specified limit
    if (delta <= amountMax) {
        poolManager.clear(currency, delta);
    } else {
        _take(currency, msgSender(), delta);
    }
}
```

[_getFullCredit](./DeltaResolver_ko.md#_getfullcredit) 메서드를 사용하여 `PoolManager`에서 `PositionManager`의 플래시 어카운팅 잔액을 가져옵니다. 이 값은 비음수여야 합니다.

`delta`가 0이면 바로 반환합니다.

`delta`가 `amountMax` 이하이면 [poolManager.clear](../../v4-core/en/PoolManager_ko.md#clear) 메서드를 호출하여 토큰을 포기합니다.

그렇지 않으면 [_take](./DeltaResolver_ko.md#_take) 메서드를 호출하여 `PoolManager`에서 호출자에게 토큰을 인출합니다.

### _sweep

`PositionManager`에서 단일 토큰 잔액을 인출합니다(참고: `PoolManager`가 아닙니다).

```solidity
/// @notice Sweeps the entire contract balance of specified currency to the recipient
function _sweep(Currency currency, address to) internal {
    uint256 balance = currency.balanceOfSelf();
    if (balance > 0) currency.transfer(to, balance);
}
```

`currency.balanceOfSelf` 메서드를 호출하여 `PositionManager` 컨트랙트가 보유한 해당 토큰의 잔액을 조회합니다.

잔액이 0보다 크면 `currency.transfer` 메서드를 호출하여 `to` 주소로 전송합니다.

### _modifyLiquidity

```solidity
/// @dev if there is a subscriber attached to the position, this function will notify the subscriber
function _modifyLiquidity(
    PositionInfo info,
    PoolKey memory poolKey,
    int256 liquidityChange,
    bytes32 salt,
    bytes calldata hookData
) internal returns (BalanceDelta liquidityDelta, BalanceDelta feesAccrued) {
    (liquidityDelta, feesAccrued) = poolManager.modifyLiquidity(
        poolKey,
        IPoolManager.ModifyLiquidityParams({
            tickLower: info.tickLower(),
            tickUpper: info.tickUpper(),
            liquidityDelta: liquidityChange,
            salt: salt
        }),
        hookData
    );

    if (info.hasSubscriber()) {
        _notifyModifyLiquidity(uint256(salt), liquidityChange, feesAccrued);
    }
}
```

[poolManager.modifyLiquidity](../../v4-core/en/PoolManager_ko.md#modifyliquidity) 메서드를 호출하여 유동성 변경을 수행합니다. 반환값은 `liquidityDelta`와 `feesAccrued`입니다.

포지션에 subscriber가 있으면 `_notifyModifyLiquidity` 메서드를 호출하여 subscriber에게 변경 사실을 알립니다.

### _pay

`payer`가 `poolManager`에 토큰을 지불합니다. `DeltaResolver`의 [_pay](./DeltaResolver_ko.md#_pay) 메서드를 구현합니다.

```solidity
// implementation of abstract function DeltaResolver._pay
function _pay(Currency currency, address payer, uint256 amount) internal override {
    if (payer == address(this)) {
        currency.transfer(address(poolManager), amount);
    } else {
        // Casting from uint256 to uint160 is safe due to limits on the total supply of a pool
        permit2.transferFrom(payer, address(poolManager), uint160(amount), Currency.unwrap(currency));
    }
}
```

`payer`가 `PositionManager` 컨트랙트 자신이면 `currency.transfer`를 직접 호출해 토큰을 `poolManager`로 전송합니다.

그렇지 않으면 `permit2.transferFrom`을 호출하여 `payer` 계정에서 `poolManager`로 토큰을 전송합니다.
> 참고: `payer`는 사전에 `permit` 메서드를 호출하여 `PositionManager`가 자신의 토큰을 이동할 수 있도록 승인해야 합니다.

### _wrap

토큰을 래핑하여 `PositionManager` 컨트랙트의 네이티브 `ETH`를 `WETH`로 변환합니다.

```solidity
/// @dev The amount should already be <= the current balance in this contract.
function _wrap(uint256 amount) internal {
    if (amount > 0) WETH9.deposit{value: amount}();
}
```

`WETH9.deposit` 메서드를 호출하여 `amount`만큼의 `ETH`를 `WETH`로 변환합니다.

### _unwrap

토큰 래핑을 해제하여 `PositionManager` 컨트랙트의 `WETH`를 네이티브 `ETH`로 변환합니다.

```solidity
/// @dev The amount should already be <= the current balance in this contract.
function _unwrap(uint256 amount) internal {
    if (amount > 0) WETH9.withdraw(amount);
}
```

`WETH9.withdraw` 메서드를 호출하여 `amount`만큼의 `WETH`를 `ETH`로 변환합니다.
