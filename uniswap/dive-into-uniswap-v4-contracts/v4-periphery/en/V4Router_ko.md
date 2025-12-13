# V4Router

[PositionManager](./PositionManager_ko.md)가 포지션 관리에 중점을 둔 것과 달리, V4Router는 주로 거래 실행에 사용되며 [PoolManager](../../v4-core/en/PoolManager_ko.md) 컨트랙트를 호출하여 특정 거래 작업을 완료합니다.

V4Router 컨트랙트의 선언을 살펴보겠습니다:

```solidity
/// @title UniswapV4Router
/// @notice Abstract contract that contains all internal logic needed for routing through Uniswap v4 pools
/// @dev the entry point to executing actions in this contract is calling `BaseActionsRouter._executeActions`
/// An inheriting contract should call _executeActions at the point that they wish actions to be executed
abstract contract V4Router is IV4Router, BaseActionsRouter, DeltaResolver {
```

[PositionManager](./PositionManager_ko.md)와 마찬가지로, V4Router 컨트랙트도 `BaseActionsRouter` 및 `DeltaResolver` 컨트랙트를 상속하며, `BaseActionsRouter._executeActions` 메서드를 호출하여 작업을 일괄 실행합니다.

V4Router 자체는 추상 컨트랙트이며 직접 배포할 수 없습니다. 다른 컨트랙트가 V4Router 컨트랙트를 상속하고 `DeltaResolver` 컨트랙트의 `pay` 메서드를 구현해야 합니다:

```solidity
/// @notice Abstract function for contracts to implement paying tokens to the poolManager
/// @dev The recipient of the payment should be the poolManager
/// @param token The token to settle. This is known not to be the native currency
/// @param payer The address who should pay tokens
/// @param amount The number of tokens to send
function _pay(Currency token, address payer, uint256 amount) internal virtual;
```

`pay` 메서드는 지정된 양의 토큰을 `poolManager`에 지불합니다.

Uniswap v4 [universal-router](https://github.com/Uniswap/universal-router) 컨트랙트 [V4SwapRouter.sol](https://github.com/Uniswap/universal-router/blob/8bd498a3fc9f8bc8577e626c024c4fcf0691f885/contracts/modules/uniswap/v4/V4SwapRouter.sol#L14)은 V4Router 컨트랙트를 상속하고 `pay` 메서드를 구현합니다.

## 구조체 정의 (Struct Definitions)

IV4Router 인터페이스는 `swap` 메서드에 대해 일반적으로 사용되는 몇 가지 구조체를 정의합니다:

### ExactInputSingleParams

지정된 풀에서의 단일 홉 거래를 위한 정확한 입력 스왑 매개변수:

```solidity
/// @notice Parameters for a single-hop exact-input swap
struct ExactInputSingleParams {
    PoolKey poolKey;
    bool zeroForOne;
    uint128 amountIn;
    uint128 amountOutMinimum;
    bytes hookData;
}
```

그 중에서:

- `poolKey`: 풀의 키
- `zeroForOne`: `token0`에서 `token1`으로 스왑할지 여부
- `amountIn`: 입력 토큰 금액
- `amountOutMinimum`: 최소 출력 토큰 금액
- `hookData`: Hook 데이터

### ExactInputParams

다중 홉 거래를 위한 정확한 입력 스왑 매개변수:

```solidity
/// @notice Parameters for a multi-hop exact-input swap
struct ExactInputParams {
    Currency currencyIn;
    PathKey[] path;
    uint128 amountIn;
    uint128 amountOutMinimum;
}
```

그 중에서:

- `currencyIn`: 입력 토큰
- `path`: 중간 토큰에 대한 정보를 포함한 스왑 경로, [PathKey](./PathKeyLibrary_ko.md#pathkey) 구조체 정의 참조
- `amountIn`: 입력 토큰 금액
- `amountOutMinimum`: 최소 출력 토큰 금액

### ExactOutputSingleParams

지정된 풀에서의 단일 홉 거래를 위한 정확한 출력 스왑 매개변수:

```solidity
/// @notice Parameters for a single-hop exact-output swap
struct ExactOutputSingleParams {
    PoolKey poolKey;
    bool zeroForOne;
    uint128 amountOut;
    uint128 amountInMaximum;
    bytes hookData;
}
```

그 중에서:

- `poolKey`: 풀의 키
- `zeroForOne`: `token0`에서 `token1`으로 스왑할지 여부
- `amountOut`: 출력 토큰 금액
- `amountInMaximum`: 최대 입력 토큰 금액
- `hookData`: Hook 데이터

### ExactOutputParams

다중 홉 거래를 위한 정확한 출력 스왑 매개변수:

```solidity
/// @notice Parameters for a multi-hop exact-output swap
struct ExactOutputParams {
    Currency currencyOut;
    PathKey[] path;
    uint128 amountOut;
    uint128 amountInMaximum;
}
```

그 중에서:

- `currencyOut`: 출력 토큰
- `path`: 스왑 경로
- `amountOut`: 출력 토큰 금액
- `amountInMaximum`: 최대 입력 토큰 금액

## 함수 정의 (Method Definitions)

V4Router는 추상 컨트랙트이므로 직접적인 외부 호출 인터페이스를 제공하지 않습니다. 대신 이를 상속하는 컨트랙트가 특정 스왑 진입점을 구현합니다.

### _handleAction

V4Router 컨트랙트는 주로 `BaseActionsRouter._handleAction` 메서드를 구현하여 다양한 유형의 스왑 작업을 처리합니다.

```solidity
function _handleAction(uint256 action, bytes calldata params) internal override {
    // swap actions and payment actions in different blocks for gas efficiency
    if (action < Actions.SETTLE) {
        if (action == Actions.SWAP_EXACT_IN) {
            IV4Router.ExactInputParams calldata swapParams = params.decodeSwapExactInParams();
            _swapExactInput(swapParams);
            return;
        } else if (action == Actions.SWAP_EXACT_IN_SINGLE) {
            IV4Router.ExactInputSingleParams calldata swapParams = params.decodeSwapExactInSingleParams();
            _swapExactInputSingle(swapParams);
            return;
        } else if (action == Actions.SWAP_EXACT_OUT) {
            IV4Router.ExactOutputParams calldata swapParams = params.decodeSwapExactOutParams();
            _swapExactOutput(swapParams);
            return;
        } else if (action == Actions.SWAP_EXACT_OUT_SINGLE) {
            IV4Router.ExactOutputSingleParams calldata swapParams = params.decodeSwapExactOutSingleParams();
            _swapExactOutputSingle(swapParams);
            return;
        }
    } else {
        if (action == Actions.SETTLE_ALL) {
            (Currency currency, uint256 maxAmount) = params.decodeCurrencyAndUint256();
            uint256 amount = _getFullDebt(currency);
            if (amount > maxAmount) revert V4TooMuchRequested(maxAmount, amount);
            _settle(currency, msgSender(), amount);
            return;
        } else if (action == Actions.TAKE_ALL) {
            (Currency currency, uint256 minAmount) = params.decodeCurrencyAndUint256();
            uint256 amount = _getFullCredit(currency);
            if (amount < minAmount) revert V4TooLittleReceived(minAmount, amount);
            _take(currency, msgSender(), amount);
            return;
        } else if (action == Actions.SETTLE) {
            (Currency currency, uint256 amount, bool payerIsUser) = params.decodeCurrencyUint256AndBool();
            _settle(currency, _mapPayer(payerIsUser), _mapSettleAmount(amount, currency));
            return;
        } else if (action == Actions.TAKE) {
            (Currency currency, address recipient, uint256 amount) = params.decodeCurrencyAddressAndUint256();
            _take(currency, _mapRecipient(recipient), _mapTakeAmount(amount, currency));
            return;
        } else if (action == Actions.TAKE_PORTION) {
            (Currency currency, address recipient, uint256 bips) = params.decodeCurrencyAddressAndUint256();
            _take(currency, _mapRecipient(recipient), _getFullCredit(currency).calculatePortion(bips));
            return;
        }
    }
    revert UnsupportedAction(action);
}
```

V4Router 또한 [ActionsLibrary](./ActionsLibrary_ko.md)에 정의된 작업 유형을 사용합니다.

#### SWAP_EXACT_IN

정확한 입력 토큰 금액과 스왑 경로를 지정하여 다중 홉 스왑을 완료하고 출력 토큰을 계산합니다:

```solidity
IV4Router.ExactInputParams calldata swapParams = params.decodeSwapExactInParams();
_swapExactInput(swapParams);
return;
```

[ExactInputParams](#exactinputparams) 유형의 매개변수를 디코딩하고 [_swapExactInput](#_swapexactinput) 메서드를 호출하여 정확한 입력 다중 홉 스왑을 완료합니다.

#### SWAP_EXACT_IN_SINGLE

정확한 입력 토큰 금액과 풀을 지정하여 단일 홉 스왑을 완료하고 출력 토큰을 계산합니다:

```solidity
IV4Router.ExactInputSingleParams calldata swapParams = params.decodeSwapExactInSingleParams();
_swapExactInputSingle(swapParams);
return;
```

[ExactInputSingleParams](#exactinputsingleparams) 유형의 매개변수를 디코딩하고 [_swapExactInputSingle](#_swapexactinputsingle) 메서드를 호출하여 정확한 입력 단일 홉 스왑을 완료합니다.

#### SWAP_EXACT_OUT

정확한 출력 토큰 금액과 스왑 경로를 지정하여 다중 홉 스왑을 완료하고 입력 토큰을 계산합니다:

```solidity
IV4Router.ExactOutputParams calldata swapParams = params.decodeSwapExactOutParams();
_swapExactOutput(swapParams);
return;
```

[ExactOutputParams](#exactoutputparams) 유형의 매개변수를 디코딩하고 [_swapExactOutput](#_swapexactoutput) 메서드를 호출하여 정확한 출력 다중 홉 스왑을 완료합니다.

#### SWAP_EXACT_OUT_SINGLE

정확한 출력 토큰 금액과 풀을 지정하여 단일 홉 스왑을 완료하고 입력 토큰을 계산합니다:

```solidity
IV4Router.ExactOutputSingleParams calldata swapParams = params.decodeSwapExactOutSingleParams();
_swapExactOutputSingle(swapParams);
return;
```

[ExactOutputSingleParams](#exactoutputsingleparams) 유형의 매개변수를 디코딩하고 [_swapExactOutputSingle](#_swapexactoutputsingle) 메서드를 호출하여 정확한 출력 단일 홉 스왑을 완료합니다.

#### SETTLE_ALL

`poolManager`에 있는 지정된 토큰의 모든 부채(음수 델타)를 정산합니다.

```solidity
(Currency currency, uint256 maxAmount) = params.decodeCurrencyAndUint256();
uint256 amount = _getFullDebt(currency);
if (amount > maxAmount) revert V4TooMuchRequested(maxAmount, amount);
_settle(currency, msgSender(), amount);
return;
```

매개변수를 디코딩하고 [_getFullDebt](./DeltaResolver_ko.md#_getfulldebt) 메서드를 호출하여 지정된 토큰의 전체 부채(음수 델타)를 가져옵니다.

지정된 최대 상환 금액 `maxAmount`가 부채보다 작으면 예외를 발생시킵니다.

[_settle](./DeltaResolver_ko.md#_settle) 메서드를 호출하여 모든 부채를 정산합니다. `payer`는 호출자이며, `fullDebt`는 상환 금액입니다.

#### TAKE_ALL

`poolManager`에 있는 지정된 토큰의 모든 크레딧(양수 델타)을 인출합니다.

```solidity
(Currency currency, uint256 minAmount) = params.decodeCurrencyAndUint256();
uint256 amount = _getFullCredit(currency);
if (amount < minAmount) revert V4TooLittleReceived(minAmount, amount);
_take(currency, msgSender(), amount);
return;
```

매개변수를 디코딩하고 [_getFullCredit](./DeltaResolver_ko.md#_getfullcredit) 메서드를 호출하여 지정된 토큰의 전체 크레딧(양수 델타)을 가져옵니다.

지정된 최소 인출 금액 `minAmount`가 크레딧보다 크면 예외를 발생시킵니다.

[_take](./DeltaResolver_ko.md#_take) 메서드를 호출하여 모든 크레딧을 인출합니다. `recipient`는 호출자이며, `fullCredit`은 인출 금액입니다.

#### SETTLE

`poolManager`에 있는 지정된 토큰의 부채(음수 델타) 일부를 정산합니다.

```solidity
(Currency currency, uint256 amount, bool payerIsUser) = params.decodeCurrencyUint256AndBool();
_settle(currency, _mapPayer(payerIsUser), _mapSettleAmount(amount, currency));
return;
```

매개변수를 디코딩하고, [_mapPayer](./BaseActionsRouter_ko.md#_mappayer) 메서드를 호출하여 지불자를 결정하고, [_mapSettleAmount](./DeltaResolver_ko.md#_mapsettleamount) 메서드를 호출하여 정산 금액을 계산합니다.

[_settle](./DeltaResolver_ko.md#_settle) 메서드를 호출하여 지정된 부채를 정산합니다.

#### TAKE

`poolManager`에 있는 지정된 토큰의 크레딧(양수 델타) 일부를 인출합니다.

```solidity
(Currency currency, address recipient, uint256 amount) = params.decodeCurrencyAddressAndUint256();
_take(currency, _mapRecipient(recipient), _mapTakeAmount(amount, currency));
return;
```

매개변수를 디코딩하고, [_mapRecipient](./BaseActionsRouter_ko.md#_maprecipient) 메서드를 사용하여 수령인을 결정하고, [_mapTakeAmount](./DeltaResolver_ko.md#_maptakeamount) 메서드를 사용하여 인출 금액을 계산합니다.

[_take](./DeltaResolver_ko.md#_take) 메서드를 호출하여 지정된 크레딧을 인출합니다.

#### TAKE_PORTION

`poolManager`에 있는 지정된 토큰의 크레딧(양수 델타) 일부를 인출합니다. 인출 비율은 `bips`로 지정됩니다. 상한선은 `10000`으로 `100%`를 의미합니다.

[TAKE](#take)의 로직과 유사하지만, 인출 금액이 비율 `bips`에 의해 계산된다는 점이 다릅니다.

```solidity
(Currency currency, address recipient, uint256 bips) = params.decodeCurrencyAddressAndUint256();
_take(currency, _mapRecipient(recipient), _getFullCredit(currency).calculatePortion(bips));
return;
```

매개변수를 디코딩하고, [_mapRecipient](./BaseActionsRouter_ko.md#_maprecipient) 메서드를 사용하여 수령인을 결정하고, `calculatePortion` 메서드를 호출하여 인출 금액(최대 10000)을 계산합니다.

[_take](./DeltaResolver_ko.md#_take) 메서드를 호출하여 지정된 크레딧을 인출합니다.

### _swapExactInput

정확한 입력 다중 홉 스왑을 완료합니다.

입력 토큰 금액과 스왑 경로를 지정하고 다중 홉 스왑을 순차적으로 완료하여 출력 토큰을 계산합니다.

```solidity
function _swapExactInput(IV4Router.ExactInputParams calldata params) private {
    unchecked {
        // Caching for gas savings
        uint256 pathLength = params.path.length;
        uint128 amountOut;
        Currency currencyIn = params.currencyIn;
        uint128 amountIn = params.amountIn;
        if (amountIn == ActionConstants.OPEN_DELTA) amountIn = _getFullCredit(currencyIn).toUint128();
        PathKey calldata pathKey;

        for (uint256 i = 0; i < pathLength; i++) {
            pathKey = params.path[i];
            (PoolKey memory poolKey, bool zeroForOne) = pathKey.getPoolAndSwapDirection(currencyIn);
            // The output delta will always be positive, except for when interacting with certain hook pools
            amountOut = _swap(poolKey, zeroForOne, -int256(uint256(amountIn)), pathKey.hookData).toUint128();

            amountIn = amountOut;
            currencyIn = pathKey.intermediateCurrency;
        }

        if (amountOut < params.amountOutMinimum) revert V4TooLittleReceived(params.amountOutMinimum, amountOut);
    }
}
```

입력 토큰 금액 `amountIn`이 `ActionConstants.OPEN_DELTA`, 즉 `0`인 경우, `poolManager`에 있는 현재 컨트랙트의 전체 크레딧(플래시 어카운팅 잔액)을 `amountIn`으로 사용합니다. [_getFullCredit](./DeltaResolver_ko.md#_getfullcredit) 메서드를 참조하십시오.

스왑 경로를 순차적으로 순회합니다:

1. 각 스왑 경로에 대해 [getPoolAndSwapDirection](./PathKeyLibrary_ko.md#getpoolandswapdirection) 메서드를 기반으로 이 스왑의 스왑 풀과 방향을 결정합니다.
2. [_swap](#_swap) 메서드를 호출하여 단일 단계 스왑을 완료합니다. `amountSpecified`는 음수이며 정확한 입력을 나타냅니다. 반환된 `amountOut`은 출력 토큰 금액입니다.
3. 이 중간 스왑의 출력 `amountOut`을 다음 스왑의 입력 `amountIn`으로 사용합니다.
4. 이 중간 스왑의 중간 토큰 주소 `intermediateCurrency`를 다음 스왑의 입력 토큰 주소로 사용하여 다음 스왑 풀과 방향을 결정합니다.

모든 스왑이 완료된 후 얻은 `amountOut`은 목표 토큰의 양입니다.

`amountOut`이 `params.amountOutMinimum`보다 작은지 확인합니다. 작다면 예외를 발생시킵니다.

### _swapExactInputSingle

정확한 입력 단일 홉 스왑을 완료합니다.

* `zeroForOne`이 `true`이면 `token0`의 정확한 입력과 `token1`의 출력을 의미합니다.
* 그렇지 않으면 `token1`의 정확한 입력과 `token0`의 출력을 의미합니다.

```solidity
function _swapExactInputSingle(IV4Router.ExactInputSingleParams calldata params) private {
    uint128 amountIn = params.amountIn;
    if (amountIn == ActionConstants.OPEN_DELTA) {
        amountIn =
            _getFullCredit(params.zeroForOne ? params.poolKey.currency0 : params.poolKey.currency1).toUint128();
    }
    uint128 amountOut =
        _swap(params.poolKey, params.zeroForOne, -int256(uint256(amountIn)), params.hookData).toUint128();
    if (amountOut < params.amountOutMinimum) revert V4TooLittleReceived(params.amountOutMinimum, amountOut);
}
```

먼저 입력 토큰 금액 `amountIn`을 계산합니다. `amountIn`이 `ActionConstants.OPEN_DELTA`, 즉 `0`인 경우 `poolManager`에 있는 현재 컨트랙트의 전체 크레딧(플래시 어카운팅 잔액)으로 설정합니다. `params.zeroForOne`을 기반으로 토큰 주소 `currency0` 또는 `currency1`을 조회할지 결정합니다.

[_swap](#_swap) 메서드를 호출하여 단일 단계 스왑을 완료합니다. `amountSpecified`는 음수이며 정확한 입력을 나타냅니다. 반환된 `amountOut`은 출력 토큰 금액입니다.
`amountOut`이 `params.amountOutMinimum`보다 작은지 확인합니다. 작다면 예외를 발생시킵니다.

### _swapExactOutput

정확한 출력 다중 홉 스왑을 완료합니다.

출력 토큰 금액과 스왑 경로를 지정하고 다중 홉 스왑을 순차적으로 완료하여 입력 토큰을 계산합니다.

```solidity
function _swapExactOutput(IV4Router.ExactOutputParams calldata params) private {
    unchecked {
        // Caching for gas savings
        uint256 pathLength = params.path.length;
        uint128 amountIn;
        uint128 amountOut = params.amountOut;
        Currency currencyOut = params.currencyOut;
        PathKey calldata pathKey;

        if (amountOut == ActionConstants.OPEN_DELTA) {
            amountOut = _getFullDebt(currencyOut).toUint128();
        }

        for (uint256 i = pathLength; i > 0; i--) {
            pathKey = params.path[i - 1];
            (PoolKey memory poolKey, bool oneForZero) = pathKey.getPoolAndSwapDirection(currencyOut);
            // The output delta will always be negative, except for when interacting with certain hook pools
            amountIn = (uint256(-int256(_swap(poolKey, !oneForZero, int256(uint256(amountOut)), pathKey.hookData))))
                .toUint128();

            amountOut = amountIn;
            currencyOut = pathKey.intermediateCurrency;
        }
        if (amountIn > params.amountInMaximum) revert V4TooMuchRequested(params.amountInMaximum, amountIn);
    }
}
```

출력 토큰 금액 `amountOut`이 `ActionConstants.OPEN_DELTA`, 즉 `0`인 경우 `poolManager`에 있는 현재 컨트랙트의 전체 부채(음수 델타)로 설정합니다. [_getFullDebt](./DeltaResolver_ko.md#_getfulldebt) 메서드를 참조하십시오.

출력 토큰을 기반으로 입력 토큰을 계산해야 하므로 스왑 경로를 역순으로 순회합니다:

1. 각 스왑 경로에 대해 [getPoolAndSwapDirection](./PathKeyLibrary_ko.md#getpoolandswapdirection) 메서드를 기반으로 이 스왑의 스왑 풀과 방향을 결정합니다.
2. [_swap](#_swap) 메서드를 호출하여 단일 단계 스왑을 완료합니다. `amountSpecified`는 양수이며 정확한 출력을 나타냅니다. 반환된 `amountIn`은 입력 토큰 금액입니다.
   * 반환된 `amountIn`은 음수로 입력 토큰 금액을 나타내며, 다음 작업에서는 출력 토큰(즉, 양수)으로 표시되어야 하므로 부호를 부정해야 합니다.
3. 이 중간 스왑의 입력 `amountIn`을 다음 스왑의 출력 `amountOut`으로 사용합니다.
4. 이 중간 스왑의 중간 토큰 주소 `intermediateCurrency`를 다음 스왑의 출력 토큰 주소로 사용합니다.

모든 스왑이 완료된 후 얻은 `amountIn`은 목표 토큰의 양입니다.

`amountIn`과 `params.amountInMaximum`은 모두 양수이므로, `amountIn`이 `params.amountInMaximum`보다 큰지 확인합니다. 크다면 예외를 발생시킵니다.

### _swapExactOutputSingle

정확한 출력 단일 홉 스왑을 완료합니다.

* `zeroForOne`이 `true`이면 `token1`의 정확한 출력과 `token0`의 입력을 의미합니다.
* 그렇지 않으면 `token0`의 정확한 출력과 `token1`의 입력을 의미합니다.

```solidity
function _swapExactOutputSingle(IV4Router.ExactOutputSingleParams calldata params) private {
    uint128 amountOut = params.amountOut;
    if (amountOut == ActionConstants.OPEN_DELTA) {
        amountOut =
            _getFullDebt(params.zeroForOne ? params.poolKey.currency1 : params.poolKey.currency0).toUint128();
    }
    uint128 amountIn = (
        uint256(-int256(_swap(params.poolKey, params.zeroForOne, int256(uint256(amountOut)), params.hookData)))
    ).toUint128();
    if (amountIn > params.amountInMaximum) revert V4TooMuchRequested(params.amountInMaximum, amountIn);
}
```

먼저 출력 토큰 금액 `amountOut`을 계산합니다. `amountOut`이 `ActionConstants.OPEN_DELTA`, 즉 `0`인 경우 `poolManager`에 있는 현재 컨트랙트의 전체 부채(음수 델타)로 설정합니다. `params.zeroForOne`을 기반으로 토큰 주소 `currency1` 또는 `currency0`을 조회할지 결정합니다.

[_swap](#_swap) 메서드를 호출하여 단일 단계 스왑을 완료합니다. `amountSpecified`는 양수이며 정확한 출력을 나타냅니다. 반환된 `amountIn`은 음수로 입력 토큰 금액을 나타냅니다. 이를 양수로 변환하기 위해 부호를 부정합니다.

`amountIn`이 `params.amountInMaximum`보다 큰지 확인합니다. 크다면 예외를 발생시킵니다.

### _swap

단일 단계 스왑을 완료합니다.

입력 매개변수:

- `poolKey`: 풀의 키
- `zeroForOne`: `token0`에서 `token1`으로 스왑할지 여부
- `amountSpecified`: 지정된 토큰 금액
  - `0`보다 작으면 정확한 입력을 나타냄
  - `0`보다 크면 정확한 출력을 나타냄
- `hookData`: Hooks의 `beforeSwap` 및 `afterSwap` 콜백을 위한 Hook 데이터

```solidity
function _swap(PoolKey memory poolKey, bool zeroForOne, int256 amountSpecified, bytes calldata hookData)
    private
    returns (int128 reciprocalAmount)
{
    // for protection of exactOut swaps, sqrtPriceLimit is not exposed as a feature in this contract
    unchecked {
        BalanceDelta delta = poolManager.swap(
            poolKey,
            IPoolManager.SwapParams(
                zeroForOne, amountSpecified, zeroForOne ? TickMath.MIN_SQRT_PRICE + 1 : TickMath.MAX_SQRT_PRICE - 1
            ),
            hookData
        );

        reciprocalAmount = (zeroForOne == amountSpecified < 0) ? delta.amount1() : delta.amount0();
    }
}
```

그 중에서 `IPoolManager.SwapParams` 구조체는 다음과 같이 정의됩니다:

```solidity
struct SwapParams {
    /// Whether to swap token0 for token1 or vice versa
    bool zeroForOne;
    /// The desired input amount if negative (exactIn), or the desired output amount if positive (exactOut)
    int256 amountSpecified;
    /// The sqrt price at which, if reached, the swap will stop executing
    uint160 sqrtPriceLimitX96;
}
```

`zeroForOne`이 `true`이면 `token0`를 `token1`으로 스왑하는 것을 의미합니다. 스왑이 진행됨에 따라 풀의 `token0` 양은 증가하고 `token1` 양은 감소합니다. 따라서 $ \sqrt{P} $ = $ \sqrt{\frac{y}{x}} $는 감소하며, 가격 제한은 현재 가격보다 작아야 합니다. 여기서 `sqrtPriceLimitX96`은 최소 가격인 `TickMath.MIN_SQRT_PRICE + 1`로 설정되어 스왑에 대한 가격 제한이 없음을 나타냅니다.

마찬가지로 `zeroForOne`이 `false`이면 `sqrtPriceLimitX96`을 `TickMath.MAX_SQRT_PRICE - 1`로 설정합니다.

여기서 가격 제한은 설정되지 않았지만, 외부 호출 메서드에서 `reciprocalAmount`가 최대/최소 값을 초과하는지 확인하여 스왑의 안전을 보장할 수 있습니다.

[poolManager.swap](../../v4-core/en/PoolManager_ko.md#swap) 메서드를 호출하여 특정 스왑 작업을 완료합니다. 반환된 `delta`는 `BalanceDelta` 구조체이며, 상위 128비트는 `amount0`을 나타내고 하위 128비트는 `amount1`을 나타냅니다.

`amountSpecified < 0`이면 `exactInput`, 즉 정확한 입력을 나타냅니다. `zeroForOne`과 결합하면 다음 조합이 가능합니다:

| zeroForOne | amountSpecified < 0 | 설명 |
| --- | --- | --- |
| true | true | `amount0`의 정확한 입력, `amount1`의 출력 계산 |
| true | false | `amount1`의 정확한 출력, `amount0`의 입력 계산 |
| false | true | `amount1`의 정확한 입력, `amount0`의 출력 계산 |
| false | false | `amount0`의 정확한 출력, `amount1`의 입력 계산 |

따라서 `zeroForOne == amountSpecified < 0`이면 항상 `delta.amount1()`을 반환하고, 그렇지 않으면 `delta.amount0()`을 반환합니다.
