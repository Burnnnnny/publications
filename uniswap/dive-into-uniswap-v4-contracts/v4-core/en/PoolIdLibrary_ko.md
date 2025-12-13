# PoolId 라이브러리 (PoolId Library)

`PoolId` 타입은 PoolId.sol 컨트랙트에 정의되어 있으며, 본질적으로 `bytes32`입니다:

```solidity
import {PoolKey} from "./PoolKey.sol";

type PoolId is bytes32;

/// @notice Library for computing the ID of a pool
library PoolIdLibrary {
    /// @notice Returns value equal to keccak256(abi.encode(poolKey))
    function toId(PoolKey memory poolKey) internal pure returns (PoolId poolId) {
        assembly ("memory-safe") {
            // 0xa0 represents the total size of the poolKey struct (5 slots of 32 bytes)
            poolId := keccak256(poolKey, 0xa0)
        }
    }
}
```

이 컨트랙트는 `PoolKey` 구조체를 `PoolId`로 변환하는 `toId`라는 하나의 메서드만 가지고 있습니다. `PoolKey` 구조체의 모든 필드를 `keccak256`으로 해시하며, 그 결과가 `PoolId`가 됩니다.

### PoolKey

`PoolKey` 구조체는 다음과 같이 정의됩니다:

```solidity
/// @notice Returns the key for identifying a pool
struct PoolKey {
    /// @notice The lower currency of the pool, sorted numerically
    Currency currency0;
    /// @notice The higher currency of the pool, sorted numerically
    Currency currency1;
    /// @notice The pool LP fee, capped at 1_000_000. If the highest bit is 1, the pool has a dynamic fee and must be exactly equal to 0x800000
    uint24 fee;
    /// @notice Ticks that involve positions must be a multiple of tick spacing
    int24 tickSpacing;
    /// @notice The hooks of the pool
    IHooks hooks;
}
```

Uniswap v4가 다음 필드들을 통해 풀을 고유하게 식별함을 알 수 있습니다:

- `currency0`: 풀의 첫 번째 토큰
- `currency1`: 풀의 두 번째 토큰
- `fee`: 풀의 수수료
- `tickSpacing`: 풀의 틱 간격
- `hooks`: 풀의 훅

이 구조체를 `keccak256`을 사용하여 해시하면, 그 결과가 풀의 고유 ID가 됩니다.
