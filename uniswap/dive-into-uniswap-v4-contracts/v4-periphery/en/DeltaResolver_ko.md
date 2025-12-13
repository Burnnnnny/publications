# DeltaResolver

DeltaResolver는 `PoolManager` 컨트랙트와 동기화(sync), 토큰 전송, 자금 정산을 수행하는 데 사용되는 추상 컨트랙트입니다.

## 함수 정의 (Method Definitions)

### _pay

DeltaResolver는 `_pay`라는 추상 메서드를 정의하며, 이는 지정된 양의 토큰을 `poolManager`에 지불하는 것을 구현합니다. DeltaResolver를 상속하는 모든 컨트랙트는 `_pay` 메서드를 구현해야 합니다.

예를 들어, [PositionManager](./PositionManager_ko.md) 컨트랙트는 DeltaResolver 컨트랙트를 상속하고 [_pay](./PositionManager_ko.md#_pay) 메서드를 구현합니다.

입력 매개변수:

- `Currency token`: 지불할 토큰
- `address payer`: 지불자의 주소
- `uint256 amount`: 지불할 금액

```solidity
/// @notice Abstract function for contracts to implement paying tokens to the poolManager
/// @dev The recipient of the payment should be the poolManager
/// @param token The token to settle. This is known not to be the native currency
/// @param payer The address who should pay tokens
/// @param amount The number of tokens to send
function _pay(Currency token, address payer, uint256 amount) internal virtual;
```

### _getFullDebt

`PoolManager`에서 현재 컨트랙트의 전체 부채(음수 델타)를 가져옵니다. 반환 값은 부호를 반전시킨 후의 `uint256` 타입입니다.

입력 매개변수:

- `Currency currency`: 토큰 주소

```solidity
/// @notice Obtain the full amount owed by this contract (negative delta)
/// @param currency Currency to get the delta for
/// @return amount The amount owed by this contract as a uint256
function _getFullDebt(Currency currency) internal view returns (uint256 amount) {
    int256 _amount = poolManager.currencyDelta(address(this), currency);
    // If the amount is positive, it should be taken not settled.
    if (_amount > 0) revert DeltaNotNegative(currency);
    // Casting is safe due to limits on the total supply of a pool
    amount = uint256(-_amount);
}
```

### _getFullCredit

`PoolManager`에서 현재 컨트랙트의 전체 크레딧(양수 델타)을 가져옵니다. 반환 값은 `uint256` 타입입니다.

입력 매개변수:

- `Currency currency`: 토큰 주소

```solidity
/// @notice Obtain the full credit owed to this contract (positive delta)
/// @param currency Currency to get the delta for
/// @return amount The amount owed to this contract as a uint256
function _getFullCredit(Currency currency) internal view returns (uint256 amount) {
    int256 _amount = poolManager.currencyDelta(address(this), currency);
    // If the amount is negative, it should be settled not taken.
    if (_amount < 0) revert DeltaNotPositive(currency);
    amount = uint256(_amount);
}
```

### _take

단일 토큰의 잔액을 인출합니다. `PoolManager` 컨트랙트는 토큰을 `recipient`에게 전송합니다.

입력 매개변수:

- `currency`: 토큰
- `recipient`: 수령인
- `amount`: 인출할 금액

```solidity
/// @notice Take an amount of currency out of the PoolManager
/// @param currency Currency to take
/// @param recipient Address to receive the currency
/// @param amount Amount to take
/// @dev Returns early if the amount is 0
function _take(Currency currency, address recipient, uint256 amount) internal {
    if (amount == 0) return;
    poolManager.take(currency, recipient, amount);
}
```

`amount`가 0이면 바로 반환합니다.

[PoolManager.take](../../v4-core/en/PoolManager_ko.md#take) 메서드를 호출하여 `poolManager`에서 지정된 양의 토큰을 인출합니다.

### _settle

단일 토큰의 부채를 정산합니다. 토큰은 `PoolManager`에 지불되어야 합니다.

[PoolManager.settle](../../v4-core/en/PoolManager_ko.md#settle) 메서드에서 우리는 부채 정산 과정을 소개했습니다:

1. [PoolManager.sync](../../v4-core/en/PoolManager_ko.md#sync) 메서드를 호출하여 토큰 잔액을 동기화합니다.
2. `PoolManager`로 토큰을 전송합니다.
3. [PoolManager.settle](../../v4-core/en/PoolManager_ko.md#settle) 메서드를 호출하여 회계 잔액을 정산합니다.

입력 매개변수:

- `currency`: 토큰
- `payer`: 지불자
- `amount`: 지불할 금액

```solidity
/// @notice Pay and settle a currency to the PoolManager
/// @dev The implementing contract must ensure that the `payer` is a secure address
/// @param currency Currency to settle
/// @param payer Address of the payer
/// @param amount Amount to send
/// @dev Returns early if the amount is 0
function _settle(Currency currency, address payer, uint256 amount) internal {
    if (amount == 0) return;

    poolManager.sync(currency);
    if (currency.isAddressZero()) {
        poolManager.settle{value: amount}();
    } else {
        _pay(currency, payer, amount);
        poolManager.settle();
    }
}
```

`amount`가 0이면 바로 반환합니다.

[poolManager.sync](../../v4-core/en/PoolManager_ko.md#sync) 메서드를 호출하여 `poolManager`의 토큰 잔액을 동기화합니다.

`currency`가 `ADDRESS_ZERO`, 즉 네이티브 ETH인 경우, `{value: amount}`를 통해 `poolManager`에 ETH를 전송하고 [PoolManager.settle](../../v4-core/en/PoolManager_ko.md#settle) 메서드를 호출하여 회계 잔액을 정산합니다.

그렇지 않고 ERC20 토큰인 경우, [_pay](#_pay) 메서드를 호출하여 `poolManager`에 토큰을 전송한 다음, [PoolManager.settle](../../v4-core/en/PoolManager_ko.md#settle) 메서드를 호출하여 회계 잔액을 정산합니다.

### _mapSettleAmount

`_mapSettleAmount` 메서드는 `amount`와 `currency`를 기반으로 정산할 토큰 금액을 계산합니다:
* `amount`가 `1 << 255`, 즉 `ActionConstants.CONTRACT_BALANCE`인 경우, 정산을 위해 현재 컨트랙트의 토큰 잔액을 사용하는 것을 의미합니다.
* 그렇지 않고 `amount`가 `0`, 즉 `ActionConstants.OPEN_DELTA`인 경우, 모든 부채를 정산하는 것을 의미합니다.
* 그렇지 않으면 지정된 `amount`만큼의 토큰을 정산하는 것을 의미합니다.

```solidity
/// @notice Calculates the amount for a settle action
function _mapSettleAmount(uint256 amount, Currency currency) internal view returns (uint256) {
    if (amount == ActionConstants.CONTRACT_BALANCE) {
        return currency.balanceOfSelf();
    } else if (amount == ActionConstants.OPEN_DELTA) {
        return _getFullDebt(currency);
    } else {
        return amount;
    }
}
```

### _mapTakeAmount

`_mapTakeAmount` 메서드는 `amount`와 `currency`를 기반으로 인출할 토큰 금액을 계산합니다:
* `amount`가 `0`인 경우, `PoolManager`에서 현재 컨트랙트의 인출 가능한 모든 잔액을 인출하는 것을 의미합니다.
* 그렇지 않으면 지정된 `amount`만큼의 토큰을 인출하는 것을 의미합니다.

```solidity
/// @notice Calculates the amount for a take action
function _mapTakeAmount(uint256 amount, Currency currency) internal view returns (uint256) {
    if (amount == ActionConstants.OPEN_DELTA) {
        return _getFullCredit(currency);
    } else {
        return amount;
    }
}
```

### _mapWrapUnwrapAmount

래핑(wrap)/언래핑(unwrap)할 토큰 양을 계산합니다.

입력 매개변수:

- `inputCurrency`: 입력 토큰, 네이티브 토큰 또는 래핑된 토큰일 수 있음
- `amount`: 래핑/언래핑할 금액, `CONTRACT_BALANCE`, `OPEN_DELTA` 또는 특정 금액일 수 있음
- `outputCurrency`: 래핑/언래핑 후의 토큰, 사용자가 `PoolManager`에서 빚질 수 있는 토큰

```solidity
/// @notice Calculates the sanitized amount before wrapping/unwrapping.
/// @param inputCurrency The currency, either native or wrapped native, that this contract holds
/// @param amount The amount to wrap or unwrap. Can be CONTRACT_BALANCE, OPEN_DELTA or a specific amount
/// @param outputCurrency The currency after the wrap/unwrap that the user may owe a balance in on the poolManager
function _mapWrapUnwrapAmount(Currency inputCurrency, uint256 amount, Currency outputCurrency)
    internal
    view
    returns (uint256)
{
    // if wrapping, the balance in this contract is in ETH
    // if unwrapping, the balance in this contract is in WETH
    uint256 balance = inputCurrency.balanceOf(address(this));
    if (amount == ActionConstants.CONTRACT_BALANCE) {
        // return early to avoid unnecessary balance check
        return balance;
    }
    if (amount == ActionConstants.OPEN_DELTA) {
        // if wrapping, the open currency on the PoolManager is WETH.
        // if unwrapping, the open currency on the PoolManager is ETH.
        // note that we use the DEBT amount. Positive deltas can be taken and then wrapped.
        amount = _getFullDebt(outputCurrency);
    }
    if (amount > balance) revert InsufficientBalance();
    return amount;
}
```

현재 컨트랙트의 `inputCurrency` 토큰 잔액을 가져옵니다.

`amount`가 `1 << 255`인 경우, 현재 컨트랙트의 토큰 잔액을 래핑/언래핑 금액으로 사용함을 의미합니다.

`amount`가 `0`인 경우, `PoolManager`에서 `outputCurrency`에 대한 현재 컨트랙트의 부채를 래핑/언래핑 금액으로 사용함을 의미합니다.

부채 금액이 `inputCurrency` 토큰 잔액보다 작은지 확인합니다.
