# BaseActionsRouter

## 함수 정의 (Method Definitions)

### _executeActions

작업(Actions)을 일괄 실행합니다.

```solidity
/// @notice internal function that triggers the execution of a set of actions on v4
/// @dev inheriting contracts should call this function to trigger execution
function _executeActions(bytes calldata unlockData) internal {
    poolManager.unlock(unlockData);
}
```

[PoolManager.unlock](../../v4-core/en/PoolManager_ko.md#unlock) 메서드를 호출합니다. 이 메서드는 잠금 해제(unlock) 작업을 수행하고 [_unlockCallback](#_unlockcallback) 메서드를 콜백합니다.

### _unlockCallback

콜백 메서드로, [PoolManager.unlock](../../v4-core/en/PoolManager_ko.md#unlock) 메서드에 의해 호출되어 특정 작업을 실행합니다.

입력 매개변수 `data`는 `PoolManager.unlock` 메서드를 호출할 때 전달됩니다.

```solidity
/// @notice function that is called by the PoolManager through the SafeCallback.unlockCallback
/// @param data Abi encoding of (bytes actions, bytes[] params)
/// where params[i] is the encoded parameters for actions[i]
function _unlockCallback(bytes calldata data) internal override returns (bytes memory) {
    // abi.decode(data, (bytes, bytes[]));
    (bytes calldata actions, bytes[] calldata params) = data.decodeActionsRouterParams();
    _executeActionsWithoutUnlock(actions, params);
    return "";
}
```

`data.decodeActionsRouterParams()`를 통해 `data`를 디코딩하여 `actions`와 `params`를 얻습니다.

`data`의 형식은 `abi.encode(bytes actions, bytes[] params)`로 계산된 `bytes`이며, 여기서 `params[i]`는 `actions[i]`의 매개변수입니다.

* `actions`의 형식은 `abi.encodePacked(uint8 action1, uint8 action2, ...)`로 계산된 `bytes`이며, 각 바이트는 하나의 `action`을 나타냅니다. 모든 `actions`는 [ActionsLibrary](./ActionsLibrary_ko.md)에 정의되어 있습니다.
* `params`는 배열이며, `params[i]`의 형식은 `abi.encode(v1, v2, ...)`로 계산된 `bytes`입니다.

[_executeActionsWithoutUnlock](#_executeactionswithoutunlock) 메서드를 호출하여 특정 작업을 실행합니다.

### _executeActionsWithoutUnlock

모든 작업과 해당 매개변수를 순회하며 각 작업을 순서대로 실행합니다.

```solidity
function _executeActionsWithoutUnlock(bytes calldata actions, bytes[] calldata params) internal {
    uint256 numActions = actions.length;
    if (numActions != params.length) revert InputLengthMismatch();

    for (uint256 actionIndex = 0; actionIndex < numActions; actionIndex++) {
        uint256 action = uint8(actions[actionIndex]);

        _handleAction(action, params[actionIndex]);
    }
}
```

`actions`와 `params`의 길이가 일치하는지 확인한 다음, `_handleAction`을 통해 각 작업을 순서대로 실행합니다. `_handleAction` 메서드는 [PositionManager._handleAction](./PositionManager_ko.md#_handleaction) 및 [V4Router._handleAction](./V4Router_ko.md#_handleaction)과 같이 `BaseActionsRouter`를 상속하는 컨트랙트에 의해 구현됩니다.

### _mapRecipient

`_mapRecipient` 메서드는 `recipient` 주소를 계산하는 데 사용됩니다:

* `recipient`가 `address(1)`, 즉 `ActionConstants.MSG_SENDER`인 경우, `msgSender()` 호출자 주소를 나타냅니다.
* 그렇지 않고 `recipient`가 `address(2)`, 즉 `ActionConstants.ADDRESS_THIS`인 경우, 현재 컨트랙트 주소(예: `PositionManager`)를 나타냅니다.
* 그렇지 않으면 `recipient` 주소 자체를 사용합니다.

```solidity
/// @notice Calculates the address for a action
function _mapRecipient(address recipient) internal view returns (address) {
    if (recipient == ActionConstants.MSG_SENDER) {
        return msgSender();
    } else if (recipient == ActionConstants.ADDRESS_THIS) {
        return address(this);
    } else {
        return recipient;
    }
}
```

### _mapPayer

`_mapPayer` 메서드는 `payerIsUser`를 기반으로 지불자를 결정합니다:

* `payerIsUser`가 `true`인 경우, 지불자는 `msgSender()`, 즉 호출자입니다.
* 그렇지 않으면 지불자는 현재 컨트랙트 주소(예: `PositionManager`)입니다.

```solidity
/// @notice Calculates the payer for an action
function _mapPayer(bool payerIsUser) internal view returns (address) {
    return payerIsUser ? msgSender() : address(this);
}
```
