# CurrencyDelta 라이브러리 (CurrencyDelta Library)

CurrencyDelta 라이브러리는 특정 주소의 토큰 잔액 변경을 기록하여 토큰 잔액을 업데이트함으로써 플래시 어카운팅(flash accounting)을 구현합니다. 모든 작업이 `임시 저장(transient storage)`에서 수행되므로 가스를 크게 절약할 수 있습니다.

[EIP-1153](https://eips.ethereum.org/EIPS/eip-1153)은 컨트랙트에서 임시 저장을 사용하는 방법을 소개합니다. `tload`와 `tstore` 작업을 도입함으로써 컨트랙트에서 임시 저장소에 대한 읽기 및 쓰기 작업을 구현할 수 있습니다.

### _computeSlot

`target`과 `currency`를 기반으로 잔액 변경 값을 저장할 스토리지 슬롯 주소를 계산합니다.

```solidity
/// @notice calculates which storage slot a delta should be stored in for a given account and currency
function _computeSlot(address target, Currency currency) internal pure returns (bytes32 hashSlot) {
    assembly ("memory-safe") {
        mstore(0, and(target, 0xffffffffffffffffffffffffffffffffffffffff))
        mstore(32, and(currency, 0xffffffffffffffffffffffffffffffffffffffff))
        hashSlot := keccak256(0, 64)
    }
}
```

`target`과 `currency` 모두 `address` 타입이므로, 데이터의 하위 20바이트만 유지하면 됩니다.

EVM 메모리 레이아웃에서 처음 64바이트는 해시 계산을 위한 임시 영역으로 사용되므로, `mstore` 명령어를 사용하여 `target`과 `currency`를 주소 0과 32(각각 32바이트)에 쓸 수 있습니다.

마지막으로 `keccak256(0, 64)`를 사용하여 처음 64바이트의 데이터를 해시하여 슬롯 주소를 얻습니다.

### getDelta

특정 주소와 토큰의 회계 잔액을 가져옵니다.

```solidity
function getDelta(Currency currency, address target) internal view returns (int256 delta) {
    bytes32 hashSlot = _computeSlot(target, currency);
    assembly ("memory-safe") {
        delta := tload(hashSlot)
    }
}
```

`_computeSlot`을 통해 슬롯 주소를 계산한 다음, `tload`를 통해 슬롯의 값을 읽습니다.

### applyDelta

새로운 잔액 변경 값을 적용하여 지정된 주소와 토큰의 잔액을 업데이트합니다.

입력 매개변수:

- `currency` 토큰 타입
- `target` 주소
- `delta` 잔액 변경 값

반환값:

- `previous` 업데이트 전 잔액
- `next` 업데이트 후 잔액

```solidity
/// @notice applies a new currency delta for a given account and currency
/// @return previous The prior value
/// @return next The modified result
function applyDelta(Currency currency, address target, int128 delta)
    internal
    returns (int256 previous, int256 next)
{
    bytes32 hashSlot = _computeSlot(target, currency);

    assembly ("memory-safe") {
        previous := tload(hashSlot)
    }
    next = previous + delta;
    assembly ("memory-safe") {
        tstore(hashSlot, next)
    }
}
```

`_computeSlot`을 통해 슬롯 주소를 계산한 다음, `tload`를 통해 슬롯의 값을 읽어 업데이트 전 잔액을 얻습니다.

그 다음 새로운 잔액을 계산하고 `tstore`를 통해 저장합니다.
