# BalanceDelta

## 타입 설명 (Type Description)

BalanceDelta 타입은 `int256` 타입을 사용하여 `token0`과 `token1`의 잔액 변경을 동시에 나타냅니다.

```solidity
/// @dev Two `int128` values packed into a single `int256` where the upper 128 bits represent the amount0
/// and the lower 128 bits represent the amount1.
type BalanceDelta is int256;
```

상위 128비트는 `token0`의 잔액 변경 값 `amount0`을 나타내고, 하위 128비트는 `token1`의 잔액 변경 값 `amount1`을 나타냅니다.

## 연산자 오버로딩 (Operator Overloading)

다음 코드는 `BalanceDelta` 타입에 대한 연산자 오버로딩을 선언합니다:

```solidity
using {add as +, sub as -, eq as ==, neq as !=} for BalanceDelta global;
```

`BalanceDelta` 타입 변수에 `+`, `-`, `==`, `!=` 연산자를 사용할 때, 해당하는 [add](#add), [sub](#sub), [eq](#eq), [neq](#neq) 메서드가 호출됩니다.

## 메서드 정의 (Method Definitions)

### toBalanceDelta

두 개의 `int128` 타입 `amount0`와 `amount1`을 `BalanceDelta` 타입으로 결합합니다.

```solidity
function toBalanceDelta(int128 _amount0, int128 _amount1) pure returns (BalanceDelta balanceDelta) {
    assembly ("memory-safe") {
        balanceDelta := or(shl(128, _amount0), and(sub(shl(128, 1), 1), _amount1))
    }
}
```

`shl(128, _amount0)`은 `_amount0`을 왼쪽으로 128비트 시프트하여 `_amount0`을 상위 128비트에 배치하고, 하위 128비트는 `0`으로 설정합니다.

`sub(shl(128, 1), 1)`은 `1`을 왼쪽으로 128비트 시프트한 다음 `1`을 빼서 하위 128비트를 `1`로 설정합니다.
`and(sub(shl(128, 1), 1), _amount1)`은 `_amount1`과 하위 128비트 `1` 간의 비트 AND 연산을 수행하여 `_amount1`의 하위 128비트만 유지하고 상위 128비트는 `0`으로 설정합니다.

마지막으로 `or` 연산은 `_amount0`와 `_amount1`을 `BalanceDelta` 타입으로 결합합니다.

### add

두 `BalanceDelta` 타입 변수를 더하여 `BalanceDelta` 타입 결과를 얻습니다.

```solidity
function add(BalanceDelta a, BalanceDelta b) pure returns (BalanceDelta) {
    int256 res0;
    int256 res1;
    assembly ("memory-safe") {
        let a0 := sar(128, a)
        let a1 := signextend(15, a)
        let b0 := sar(128, b)
        let b1 := signextend(15, b)
        res0 := add(a0, b0)
        res1 := add(a1, b1)
    }
    return toBalanceDelta(res0.toInt128(), res1.toInt128());
}
```

먼저 `sar(128, a)`는 `a`에 대해 128비트 산술 오른쪽 시프트를 수행하여 `a`의 상위 128비트 `a0`를 추출합니다.

그 다음 `signextend(15, a)`는 `a`의 하위 128비트를 256비트로 부호 확장하여 부호 비트를 보존하면서 `a`의 하위 128비트 `a1`을 추출합니다.

마찬가지로 `b`의 상위 128비트 `b0`와 하위 128비트 `b1`을 추출합니다.

다음으로 `a0`와 `b0`를 더하여 `res0`를 얻고, `a1`과 `b1`을 더하여 `res1`을 얻습니다.

마지막으로 [toBalanceDelta](#toBalanceDelta) 메서드를 사용하여 `res0`와 `res1`을 `BalanceDelta` 타입으로 결합합니다.

### sub

두 `BalanceDelta` 타입 변수를 빼서 `BalanceDelta` 타입 결과를 얻습니다.

```solidity
function sub(BalanceDelta a, BalanceDelta b) pure returns (BalanceDelta) {
    int256 res0;
    int256 res1;
    assembly ("memory-safe") {
        let a0 := sar(128, a)
        let a1 := signextend(15, a)
        let b0 := sar(128, b)
        let b1 := signextend(15, b)
        res0 := sub(a0, b0)
        res1 := sub(a1, b1)
    }
    return toBalanceDelta(res0.toInt128(), res1.toInt128());
}
```

[add](#add) 메서드와 유사하지만, `a0`와 `b0`를 빼서 `res0`를 얻고, `a1`과 `b1`을 빼서 `res1`을 얻는 점이 다릅니다.

### eq

두 `BalanceDelta` 타입 변수가 같은지 확인합니다.

```solidity
function eq(BalanceDelta a, BalanceDelta b) pure returns (bool) {
    return BalanceDelta.unwrap(a) == BalanceDelta.unwrap(b);
}
```

`unwrap` 메서드를 사용하여 `a`와 `b`를 `int256` 타입으로 푼 다음 동등성을 비교합니다.

### neq

두 `BalanceDelta` 타입 변수가 같지 않은지 확인합니다.

```solidity
function neq(BalanceDelta a, BalanceDelta b) pure returns (bool) {
    return BalanceDelta.unwrap(a) != BalanceDelta.unwrap(b);
}
```

`unwrap` 메서드를 사용하여 `a`와 `b`를 `int256` 타입으로 푼 다음 부등성을 비교합니다.

## BalanceDeltaLibrary

BalanceDeltaLibrary는 `BalanceDelta` 타입 변수의 `amount0`와 `amount1` 값을 가져오는 `amount0` 및 `amount1` 메서드를 제공합니다.

```solidity
/// @notice Library for getting the amount0 and amount1 deltas from the BalanceDelta type
library BalanceDeltaLibrary {
    /// @notice A BalanceDelta of 0
    BalanceDelta public constant ZERO_DELTA = BalanceDelta.wrap(0);

    function amount0(BalanceDelta balanceDelta) internal pure returns (int128 _amount0) {
        assembly ("memory-safe") {
            _amount0 := sar(128, balanceDelta)
        }
    }

    function amount1(BalanceDelta balanceDelta) internal pure returns (int128 _amount1) {
        assembly ("memory-safe") {
            _amount1 := signextend(15, balanceDelta)
        }
    }
}
```

### amount0

`BalanceDelta` 타입 변수의 `amount0` 값을 가져옵니다.

`sar(128, balanceDelta)`를 사용하여 `balanceDelta`에 대해 128비트 산술 오른쪽 시프트를 수행하여 상위 128비트를 추출합니다.

### amount1

`BalanceDelta` 타입 변수의 `amount1` 값을 가져옵니다.

`signextend(15, balanceDelta)`를 사용하여 `balanceDelta`의 하위 128비트를 256비트로 부호 확장하여 부호 비트를 보존하면서 하위 128비트를 추출합니다.
