# Position 라이브러리 (Position Library)

## 상태 정의 (State Definition)

`State` 구조체는 사용자가 보유한 포지션 정보를 정의합니다. 여기에는 다음이 포함됩니다:

- `liquidity`: 사용자가 보유한 유동성 양
- `feeGrowthInside0LastX128`: 사용자가 유동성을 보유할 때의 수수료 증가율
- `feeGrowthInside1LastX128`: 사용자가 유동성을 보유할 때의 수수료 증가율

```solidity
// info stored for each user's position
struct State {
    // the amount of liquidity owned by this position
    uint128 liquidity;
    // fee growth per unit of liquidity as of the last update to liquidity or fees owed
    uint256 feeGrowthInside0LastX128;
    uint256 feeGrowthInside1LastX128;
}
```

## get

사용자의 포지션 정보를 가져옵니다. `positionKey`를 계산한 다음 `position`을 반환합니다. 포지션은 `owner`, `tickLower`, `tickUpper`, `salt`에 의해 고유하게 결정됩니다.

```solidity
/// @notice Returns the State struct of a position, given an owner and position boundaries
/// @param self The mapping containing all user positions
/// @param owner The address of the position owner
/// @param tickLower The lower tick boundary of the position
/// @param tickUpper The upper tick boundary of the position
/// @param salt A unique value to differentiate between multiple positions in the same range
/// @return position The position info struct of the given owners' position
function get(mapping(bytes32 => State) storage self, address owner, int24 tickLower, int24 tickUpper, bytes32 salt)
    internal
    view
    returns (State storage position)
{
    bytes32 positionKey = calculatePositionKey(owner, tickLower, tickUpper, salt);
    position = self[positionKey];
}
```

## calculatePositionKey

포지션 키를 계산합니다.
`owner`, `tickLower`, `tickUpper`, `salt`를 입력하고 `positionKey`를 반환합니다. `positionKey`는 `keccak256(abi.encodePacked(owner, tickLower, tickUpper, salt))`로 계산됩니다.

```solidity
/// @notice A helper function to calculate the position key
/// @param owner The address of the position owner
/// @param tickLower the lower tick boundary of the position
/// @param tickUpper the upper tick boundary of the position
/// @param salt A unique value to differentiate between multiple positions in the same range, by the same owner. Passed in by the caller.
function calculatePositionKey(address owner, int24 tickLower, int24 tickUpper, bytes32 salt)
    internal
    pure
    returns (bytes32 positionKey)
{
    // positionKey = keccak256(abi.encodePacked(owner, tickLower, tickUpper, salt))
    assembly ("memory-safe") {
        let fmp := mload(0x40)
        mstore(add(fmp, 0x26), salt) // [0x26, 0x46)
        mstore(add(fmp, 0x06), tickUpper) // [0x23, 0x26)
        mstore(add(fmp, 0x03), tickLower) // [0x20, 0x23)
        mstore(fmp, owner) // [0x0c, 0x20)
        positionKey := keccak256(add(fmp, 0x0c), 0x3a) // len is 58 bytes

        // now clean the memory we used
        mstore(add(fmp, 0x40), 0) // fmp+0x40 held salt
        mstore(add(fmp, 0x20), 0) // fmp+0x20 held tickLower, tickUpper, salt
        mstore(fmp, 0) // fmp held owner
    }
}
```

`mstore(p, v)`는 메모리 주소 `p`에 `v`를 저장하는 데 사용되며 길이는 32바이트입니다. `owner`는 20바이트, `tickLower`와 `tickUpper`는 3바이트, `salt`는 32바이트이므로 서로 다른 메모리 주소에 저장해야 합니다.

`fmp`는 Solidity 메모리 관리의 특수 포인터인 Free Memory Pointer로, 주소 `0x40`에 저장되며 현재 사용 가능한 메모리 주소를 기록하는 데 사용됩니다. 기본적으로 `fmp`는 `0x80`을 가리킵니다.

`fmp`가 `0x00`에서 시작한다고 가정하면 여러 변수의 메모리 레이아웃은 다음과 같습니다:

$$
\overbrace{0x00, ... 0x0b}^{skip, 12 bytes}, \underbrace{\overbrace{0x0c, ..., 0x1f}^{owner, 20 bytes}, \overbrace{0x20, ..., 0x22}^{tickLower , 3 bytes}, \overbrace{0x23, ..., 0x25}^{tickUpper, 3 bytes}, \overbrace{0x26, ..., 0x45}^{salt, 32 bytes}}_{keccak256(abi.encodePacked()), 58 bytes}
$$

`mstore`는 매번 32바이트를 저장하므로 변수 길이가 32바이트보다 작으면 자동으로 0으로 채워집니다. 따라서 `salt`를 먼저 저장하고 `owner`를 마지막에 저장해야 합니다.

## update

포지션의 유동성 및 수수료를 업데이트합니다.

```solidity
/// @notice Credits accumulated fees to a user's position
/// @param self The individual position to update
/// @param liquidityDelta The change in pool liquidity as a result of the position update
/// @param feeGrowthInside0X128 The all-time fee growth in currency0, per unit of liquidity, inside the position's tick boundaries
/// @param feeGrowthInside1X128 The all-time fee growth in currency1, per unit of liquidity, inside the position's tick boundaries
/// @return feesOwed0 The amount of currency0 owed to the position owner
/// @return feesOwed1 The amount of currency1 owed to the position owner
function update(
    State storage self,
    int128 liquidityDelta,
    uint256 feeGrowthInside0X128,
    uint256 feeGrowthInside1X128
) internal returns (uint256 feesOwed0, uint256 feesOwed1) {
    uint128 liquidity = self.liquidity;

    if (liquidityDelta == 0) {
        // disallow pokes for 0 liquidity positions
        if (liquidity == 0) CannotUpdateEmptyPosition.selector.revertWith();
    } else {
        self.liquidity = LiquidityMath.addDelta(liquidity, liquidityDelta);
    }

    // calculate accumulated fees. overflow in the subtraction of fee growth is expected
    unchecked {
        feesOwed0 =
            FullMath.mulDiv(feeGrowthInside0X128 - self.feeGrowthInside0LastX128, liquidity, FixedPoint128.Q128);
        feesOwed1 =
            FullMath.mulDiv(feeGrowthInside1X128 - self.feeGrowthInside1LastX128, liquidity, FixedPoint128.Q128);
    }

    // update the position
    self.feeGrowthInside0LastX128 = feeGrowthInside0X128;
    self.feeGrowthInside1LastX128 = feeGrowthInside1X128;
}
```

* `liquidityDelta`는 유동성 변경 값으로, 유동성을 추가할 때는 양수, 제거할 때는 음수입니다.
* `feeGrowthInside0X128 - self.feeGrowthInside0LastX128`은 유동성 단위당 `token0`의 수수료 증가율을 계산하고, 포지션이 보유한 유동성을 기반으로 얻을 수 있는 수수료를 계산합니다.
* `feeGrowthInside1X128 - self.feeGrowthInside1LastX128`은 유동성 단위당 `token1`의 수수료 증가율을 계산하고, 포지션이 보유한 유동성을 기반으로 얻을 수 있는 수수료를 계산합니다.
