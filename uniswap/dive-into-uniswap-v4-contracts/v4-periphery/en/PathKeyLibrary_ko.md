# PathKey 라이브러리 (PathKey Library)

## 구조체 정의 (Struct Definitions)

### PathKey

```solidity
struct PathKey {
    Currency intermediateCurrency;
    uint24 fee;
    int24 tickSpacing;
    IHooks hooks;
    bytes hookData;
}
```

- `intermediateCurrency`: 중간 토큰
- `fee`: 수수료
- `tickSpacing`: 풀 틱 간격
- `hooks`: Hooks 컨트랙트
- `hookData`: Hooks 데이터

## 함수 정의 (Method Definitions)

### getPoolAndSwapDirection

입력 토큰과 중간 토큰 정보를 기반으로 거래 풀과 거래 방향을 계산합니다.

```solidity
/// @notice Get the pool and swap direction for a given PathKey
/// @param params the given PathKey
/// @param currencyIn the input currency
/// @return poolKey the pool key of the swap
/// @return zeroForOne the direction of the swap, true if currency0 is being swapped for currency1
function getPoolAndSwapDirection(PathKey calldata params, Currency currencyIn)
    internal
    pure
    returns (PoolKey memory poolKey, bool zeroForOne)
{
    Currency currencyOut = params.intermediateCurrency;
    (Currency currency0, Currency currency1) =
        currencyIn < currencyOut ? (currencyIn, currencyOut) : (currencyOut, currencyIn);

    zeroForOne = currencyIn == currency0;
    poolKey = PoolKey(currency0, currency1, params.fee, params.tickSpacing, params.hooks);
}
```

[PoolKey](../../v4-core/en/PoolIdLibrary_ko.md#poolkey)에서 `PoolKey` 구조체가 소개되었습니다. Uniswap v4는 `currency0`, `currency1`, `fee`, `tickSpacing`, `hooks`를 통해 풀을 고유하게 식별하며, 여기서 `currency0` 주소는 `currency1` 주소보다 작습니다.

따라서 `getPoolAndSwapDirection`에서 거래 풀을 결정할 때, 먼저 `currency0`와 `currency1`을 정렬하고 `fee`, `tickSpacing`, `hooks`와 결합하여 `poolKey`를 얻습니다.

`currencyIn`이 `currency0`와 같은지 여부에 따라 거래 방향 `zeroForOne`을 결정합니다. `currency0`가 `token0`이기 때문입니다.
