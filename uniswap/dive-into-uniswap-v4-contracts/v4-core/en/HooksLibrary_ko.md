# Hooks 라이브러리 (Hooks Library)

## Hooks 주소 설계 (Hooks Address Design)

### Hooks 권한 (Hooks Permissions)

Uniswap v4는 Hooks 주소의 하위 14비트를 사용하여 다음 권한을 나타냅니다:

- `BEFORE_INITIALIZE_FLAG = 1 << 13`: 풀 초기화 전에 실행
- `AFTER_INITIALIZE_FLAG = 1 << 12`: 풀 초기화 후에 실행

- `BEFORE_ADD_LIQUIDITY_FLAG = 1 << 11`: 유동성 추가 전에 실행
- `AFTER_ADD_LIQUIDITY_FLAG = 1 << 10`: 유동성 추가 후에 실행

- `BEFORE_REMOVE_LIQUIDITY_FLAG = 1 << 9`: 유동성 제거 전에 실행
- `AFTER_REMOVE_LIQUIDITY_FLAG = 1 << 8`: 유동성 제거 후에 실행

- `BEFORE_SWAP_FLAG = 1 << 7`: 스왑 전에 실행
- `AFTER_SWAP_FLAG = 1 << 6`: 스왑 후에 실행

- `BEFORE_DONATE_FLAG = 1 << 5`: 기부 전에 실행
- `AFTER_DONATE_FLAG = 1 << 4`: 기부 후에 실행

- `BEFORE_SWAP_RETURNS_DELTA_FLAG = 1 << 3`: 스왑 시 델타 반환 전에 실행
  - `BEFORE_SWAP_FLAG`를 활성화해야 함
- `AFTER_SWAP_RETURNS_DELTA_FLAG = 1 << 2`: 스왑 시 델타 반환 후에 실행
  - `AFTER_SWAP_FLAG`를 활성화해야 함
- `AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 1`: 유동성 추가 시 델타 반환 후에 실행
  - `AFTER_ADD_LIQUIDITY_FLAG`를 활성화해야 함
- `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 0`: 유동성 제거 시 델타 반환 후에 실행
  - `AFTER_REMOVE_LIQUIDITY_FLAG`를 활성화해야 함

### Hooks 주소 (Hooks Address)

Hooks의 권한은 주소에 포함되도록 설계되었습니다. Hooks 주소의 특정 비트를 확인함으로써 특정 권한이 있는지 확인할 수 있습니다.

예를 들어 Hooks 주소가 `0x0000000000000000000000000000000000002400`인 경우, 하위 14비트는 `10 0100 0000 0000`이므로 해당 Hooks는 `before initialize` 및 `after add liquidity` 권한을 가집니다.

### Hooks 주소 생성 방법 (How to Generate Hooks Address)

Hooks 주소가 미리 설정된 권한을 충족하도록 하려면, 권한 요구 사항이 충족될 때까지 `CREATE2` `salt` 값을 지속적으로 수정하여 Hooks 주소 생성을 시도할 수 있습니다.

다음 메서드는 Hooks 주소를 계산하는 예제입니다:

```solidity
/// @notice Precompute a contract address deployed via CREATE2
/// @param deployer The address that will deploy the hook. In `forge test`, this will be the test contract `address(this)` or the pranking address
///                 In `forge script`, this should be `0x4e59b44847b379578588920cA78FbF26c0B4956C` (CREATE2 Deployer Proxy)
/// @param salt The salt used to deploy the hook
/// @param creationCode The creation code of a hook contract
function computeAddress(address deployer, uint256 salt, bytes memory creationCode)
    internal
    pure
    returns (address hookAddress)
{
    return address(
        uint160(uint256(keccak256(abi.encodePacked(bytes1(0xFF), deployer, salt, keccak256(creationCode)))))
    );
}
```

## 함수 정의 (Function Definitions)

### isValidHookAddress

Hooks 주소의 유효성을 확인합니다.

```solidity
/// @notice Ensures that the hook address includes at least one hook flag or dynamic fees, or is the 0 address
/// @param self The hook to verify
/// @param fee The fee of the pool the hook is used with
/// @return bool True if the hook address is valid
function isValidHookAddress(IHooks self, uint24 fee) internal pure returns (bool) {
    // The hook can only have a flag to return a hook delta on an action if it also has the corresponding action flag
    if (!self.hasPermission(BEFORE_SWAP_FLAG) && self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) return false;
    if (!self.hasPermission(AFTER_SWAP_FLAG) && self.hasPermission(AFTER_SWAP_RETURNS_DELTA_FLAG)) return false;
    if (!self.hasPermission(AFTER_ADD_LIQUIDITY_FLAG) && self.hasPermission(AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG))
    {
        return false;
    }
    if (
        !self.hasPermission(AFTER_REMOVE_LIQUIDITY_FLAG)
            && self.hasPermission(AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG)
    ) return false;

    // If there is no hook contract set, then fee cannot be dynamic
    // If a hook contract is set, it must have at least 1 flag set, or have a dynamic fee
    return address(self) == address(0)
        ? !fee.isDynamicFee()
        : (uint160(address(self)) & ALL_HOOK_MASK > 0 || fee.isDynamicFee());
}
```

Hooks 주소의 권한이 올바른지 확인합니다:
* `BEFORE_SWAP_RETURNS_DELTA_FLAG`가 활성화된 경우 `BEFORE_SWAP_FLAG`가 활성화되어야 합니다.
* `AFTER_SWAP_RETURNS_DELTA_FLAG`가 활성화된 경우 `AFTER_SWAP_FLAG`가 활성화되어야 합니다.
* `AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG`가 활성화된 경우 `AFTER_ADD_LIQUIDITY_FLAG`가 활성화되어야 합니다.
* `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG`가 활성화된 경우 `AFTER_REMOVE_LIQUIDITY_FLAG`가 활성화되어야 합니다.

Hooks 주소의 로직을 확인합니다:
* Hooks가 `address(0)`인 경우, 즉 Hooks 주소가 설정되지 않은 경우 풀 수수료는 동적일 수 없습니다. 즉, `fee`는 `0x800000`과 같을 수 없습니다.
* Hooks 주소가 `address(0)`이 아닌 경우 다음 두 가지 조건 중 하나를 충족해야 합니다:
  * Hooks 주소의 하위 14비트는 적어도 하나의 권한 플래그를 가져야 합니다;
  * 권한 플래그가 없는 경우, 주소는 동적 수수료를 구현하는 컨트랙트로만 사용되므로 `fee`는 동적 수수료(즉, `0x800000`)여야 합니다.

### callHook

Hooks 컨트랙트 메서드를 실행하고 오류를 일관되게 처리합니다.

여기서 `self`는 Hooks 주소이고, `data`는 `abi.encodeCall(IHooks.beforeInitialize, (msg.sender, key, sqrtPriceX96))`와 같은 호출 데이터입니다.

```solidity
/// @notice performs a hook call using the given calldata on the given hook that doesn't return a delta
/// @return result The complete data returned by the hook
function callHook(IHooks self, bytes memory data) internal returns (bytes memory result) {
    bool success;
    assembly ("memory-safe") {
        success := call(gas(), self, 0, add(data, 0x20), mload(data), 0, 0)
    }
    // Revert with FailedHookCall, containing any error message to bubble up
    if (!success) CustomRevert.bubbleUpAndRevertWith(address(self), bytes4(data), HookCallFailed.selector);

    // The call was successful, fetch the returned data
    assembly ("memory-safe") {
        // allocate result byte array from the free memory pointer
        result := mload(0x40)
        // store new free memory pointer at the end of the array padded to 32 bytes
        mstore(0x40, add(result, and(add(returndatasize(), 0x3f), not(0x1f))))
        // store length in memory
        mstore(result, returndatasize())
        // copy return data to result
        returndatacopy(add(result, 0x20), 0, returndatasize())
    }

    // Length must be at least 32 to contain the selector. Check expected selector and returned selector match.
    if (result.length < 32 || result.parseSelector() != data.parseSelector()) {
        InvalidHookResponse.selector.revertWith();
    }
}
```

### callHookAndReturnDelta

[callHook](#callhook)을 통해 Hooks 컨트랙트 메서드를 실행하고, `parseReturn == true`인 경우 델타를 반환합니다.

```solidity
/// @notice performs a hook call using the given calldata on the given hook
/// @return int256 The delta returned by the hook
function callHookWithReturnDelta(IHooks self, bytes memory data, bool parseReturn) internal returns (int256) {
    bytes memory result = callHook(self, data);

    // If this hook wasn't meant to return something, default to 0 delta
    if (!parseReturn) return 0;

    // A length of 64 bytes is required to return a bytes4, and a 32 byte delta
    if (result.length != 64) InvalidHookResponse.selector.revertWith();
    return result.parseReturnDelta();
}
```

### beforeInitialize

Hooks가 `BEFORE_INITIALIZE_FLAG` 권한을 가진 경우 [callHook](#callhook)을 통해 `beforeInitialize` 메서드를 실행합니다.

```solidity
/// @notice calls beforeInitialize hook if permissioned and validates return value
function beforeInitialize(IHooks self, PoolKey memory key, uint160 sqrtPriceX96) internal noSelfCall(self) {
    if (self.hasPermission(BEFORE_INITIALIZE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeInitialize, (msg.sender, key, sqrtPriceX96)));
    }
}
```

### afterInitialize

Hooks가 `AFTER_INITIALIZE_FLAG` 권한을 가진 경우 [callHook](#callhook)을 통해 `afterInitialize` 메서드를 실행합니다.

```solidity
/// @notice calls afterInitialize hook if permissioned and validates return value
function afterInitialize(IHooks self, PoolKey memory key, uint160 sqrtPriceX96, int24 tick)
    internal
    noSelfCall(self)
{
    if (self.hasPermission(AFTER_INITIALIZE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.afterInitialize, (msg.sender, key, sqrtPriceX96, tick)));
    }
}
```

### beforeModifyLiquidity

* `liquidityDelta`가 0보다 크고 Hooks가 `BEFORE_ADD_LIQUIDITY_FLAG` 권한을 가진 경우 [callHook](#callhook)을 통해 `beforeAddLiquidity` 메서드를 실행합니다.
* 그렇지 않고 `liquidityDelta`가 0 이하이고 Hooks가 `BEFORE_REMOVE_LIQUIDITY_FLAG` 권한을 가진 경우 [callHook](#callhook)을 통해 `beforeRemoveLiquidity` 메서드를 실행합니다.

```solidity
/// @notice calls beforeModifyLiquidity hook if permissioned and validates return value
function beforeModifyLiquidity(
    IHooks self,
    PoolKey memory key,
    IPoolManager.ModifyLiquidityParams memory params,
    bytes calldata hookData
) internal noSelfCall(self) {
    if (params.liquidityDelta > 0 && self.hasPermission(BEFORE_ADD_LIQUIDITY_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeAddLiquidity, (msg.sender, key, params, hookData)));
    } else if (params.liquidityDelta <= 0 && self.hasPermission(BEFORE_REMOVE_LIQUIDITY_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeRemoveLiquidity, (msg.sender, key, params, hookData)));
    }
}
```

### afterModifyLiquidity

`callerDelta`를 `delta`로 초기화하고 다음 작업을 수행합니다:

* `liquidityDelta`가 0보다 큰 경우
  * Hooks가 `AFTER_ADD_LIQUIDITY_FLAG` 권한을 가진 경우 [callHookWithReturnDelta](#callhookandreturndelta)를 통해 `afterAddLiquidity` 메서드를 실행합니다;
    * Hooks가 `AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG` 권한을 가진 경우 `hookDelta`를 파싱합니다.
    * `callerDelta`에서 `hookDelta`를 뺍니다.
* `liquidityDelta`가 0 이하인 경우
  * Hooks가 `AFTER_REMOVE_LIQUIDITY_FLAG` 권한을 가진 경우 [callHookWithReturnDelta](#callhookandreturndelta)를 통해 `afterRemoveLiquidity` 메서드를 실행합니다;
    * Hooks가 `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG` 권한을 가진 경우 `hookDelta`를 파싱합니다.
    * `callerDelta`에서 `hookDelta`를 뺍니다.

이 과정에서 Hooks 컨트랙트는 `hookDelta`를 반환하여 `callerDelta`를 수정함으로써 유동성 변경에 영향을 줄 수 있습니다.

```solidity
/// @notice calls afterModifyLiquidity hook if permissioned and validates return value
function afterModifyLiquidity(
    IHooks self,
    PoolKey memory key,
    IPoolManager.ModifyLiquidityParams memory params,
    BalanceDelta delta,
    BalanceDelta feesAccrued,
    bytes calldata hookData
) internal returns (BalanceDelta callerDelta, BalanceDelta hookDelta) {
    if (msg.sender == address(self)) return (delta, BalanceDeltaLibrary.ZERO_DELTA);

    callerDelta = delta;
    if (params.liquidityDelta > 0) {
        if (self.hasPermission(AFTER_ADD_LIQUIDITY_FLAG)) {
            hookDelta = BalanceDelta.wrap(
                self.callHookWithReturnDelta(
                    abi.encodeCall(
                        IHooks.afterAddLiquidity, (msg.sender, key, params, delta, feesAccrued, hookData)
                    ),
                    self.hasPermission(AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG)
                )
            );
            callerDelta = callerDelta - hookDelta;
        }
    } else {
        if (self.hasPermission(AFTER_REMOVE_LIQUIDITY_FLAG)) {
            hookDelta = BalanceDelta.wrap(
                self.callHookWithReturnDelta(
                    abi.encodeCall(
                        IHooks.afterRemoveLiquidity, (msg.sender, key, params, delta, feesAccrued, hookData)
                    ),
                    self.hasPermission(AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG)
                )
            );
            callerDelta = callerDelta - hookDelta;
        }
    }
}
```

### beforeSwap

* Hooks가 `BEFORE_SWAP_FLAG` 권한을 가진 경우 [callHook](#callhook)을 통해 `beforeSwap` 메서드를 실행합니다;
  * 풀이 동적 수수료를 지원하는 경우 Hooks는 `lpFeeOverride`를 반환하여 현재 LP 수수료를 덮어쓸 수 있습니다;
  * Hooks가 `BEFORE_SWAP_RETURNS_DELTA_FLAG` 권한을 가진 경우 `hookReturn`을 파싱하고 `hookReturn`(상위 128비트)을 기반으로 `amountToSwap`을 수정합니다.
    > 참고: `hookReturn`의 상위 128비트는 `hookDeltaSpecified`를 나타내고, 하위 128비트는 `hookDeltaUnspecified`를 나타냅니다.

    `params.amountSpecified` 및 `params.zeroForOne`에 따라 `hookDeltaSpecified` 및 `hookDeltaUnspecified`는 `amount0` 또는 `amount1`을 나타낼 수 있으며 조합은 다음과 같습니다:

    | `params.amountSpecified < 0` | `params.zeroForOne` | `hookDeltaSpecified` | `hookDeltaUnspecified` |
    | --- | --- | --- | --- |
    | `true` | `true` | `amount0` | `amount1` |
    | `true` | `false` | `amount1` | `amount0` |
    | `false` | `true` | `amount1` | `amount0` |
    | `false` | `false` | `amount0` | `amount1` |

  * `beforeSwap`의 반환값은 `amountToSwap` 값에 영향을 주지만(수정하지만) 스왑 유형(정확한 입력/출력)에는 영향을 주지 않습니다.

```solidity
/// @notice calls beforeSwap hook if permissioned and validates return value
function beforeSwap(IHooks self, PoolKey memory key, IPoolManager.SwapParams memory params, bytes calldata hookData)
    internal
    returns (int256 amountToSwap, BeforeSwapDelta hookReturn, uint24 lpFeeOverride)
{
    amountToSwap = params.amountSpecified;
    if (msg.sender == address(self)) return (amountToSwap, BeforeSwapDeltaLibrary.ZERO_DELTA, lpFeeOverride);

    if (self.hasPermission(BEFORE_SWAP_FLAG)) {
        bytes memory result = callHook(self, abi.encodeCall(IHooks.beforeSwap, (msg.sender, key, params, hookData)));

        // A length of 96 bytes is required to return a bytes4, a 32 byte delta, and an LP fee
        if (result.length != 96) InvalidHookResponse.selector.revertWith();

        // dynamic fee pools that want to override the cache fee, return a valid fee with the override flag. If override flag
        // is set but an invalid fee is returned, the transaction will revert. Otherwise the current LP fee will be used
        if (key.fee.isDynamicFee()) lpFeeOverride = result.parseFee();

        // skip this logic for the case where the hook return is 0
        if (self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) {
            hookReturn = BeforeSwapDelta.wrap(result.parseReturnDelta());

            // any return in unspecified is passed to the afterSwap hook for handling
            int128 hookDeltaSpecified = hookReturn.getSpecifiedDelta();

            // Update the swap amount according to the hook's return, and check that the swap type doesn't change (exact input/output)
            if (hookDeltaSpecified != 0) {
                bool exactInput = amountToSwap < 0;
                amountToSwap += hookDeltaSpecified;
                if (exactInput ? amountToSwap > 0 : amountToSwap < 0) {
                    HookDeltaExceedsSwapAmount.selector.revertWith();
                }
            }
        }
    }
}
```

### afterSwap

* Hooks가 `AFTER_SWAP_FLAG` 권한을 가진 경우 [callHookWithReturnDelta](#callhookandreturndelta)를 통해 `afterSwap` 메서드를 실행합니다;
  * Hooks가 `AFTER_SWAP_RETURNS_DELTA_FLAG` 권한을 가진 경우 `hookDelta`를 파싱합니다.
  * 참고: `afterSwap`은 `hookDeltaUnspecified`만 반환하지만, `beforeSwap`은 `hookDeltaSpecified`와 `hookDeltaUnspecified`를 모두 반환합니다.
    * `beforeSwap`은 입력 값 `amountToSwap`에 영향을 미치는 반면, `afterSwap`은 출력 값 `hookDeltaUnspecified`에 영향을 미칩니다.
* `swapDelta`에서 `hookDelta`를 뺍니다.

`params.amountSpecified` 및 `params.zeroForOne`에 따라 `hookDeltaSpecified` 및 `hookDeltaUnspecified`의 순서를 결정하며, 조합은 다음과 같습니다:

| `params.amountSpecified < 0` | `params.zeroForOne` | `hookDeltaSpecified` | `hookDeltaUnspecified` |
| --- | --- | --- | --- |
| `true` | `true` | `amount0` | `amount1` |
| `true` | `false` | `amount1` | `amount0` |
| `false` | `true` | `amount1` | `amount0` |
| `false` | `false` | `amount0` | `amount1` |

따라서 `params.amountSpecified < 0 == params.zeroForOne`일 때 `hookDeltaSpecified`는 항상 `amount0`을 나타내고 `hookDeltaUnspecified`는 항상 `amount1`을 나타냅니다.

```solidity
/// @notice calls afterSwap hook if permissioned and validates return value
function afterSwap(
    IHooks self,
    PoolKey memory key,
    IPoolManager.SwapParams memory params,
    BalanceDelta swapDelta,
    bytes calldata hookData,
    BeforeSwapDelta beforeSwapHookReturn
) internal returns (BalanceDelta, BalanceDelta) {
    if (msg.sender == address(self)) return (swapDelta, BalanceDeltaLibrary.ZERO_DELTA);

    int128 hookDeltaSpecified = beforeSwapHookReturn.getSpecifiedDelta();
    int128 hookDeltaUnspecified = beforeSwapHookReturn.getUnspecifiedDelta();

    if (self.hasPermission(AFTER_SWAP_FLAG)) {
        hookDeltaUnspecified += self.callHookWithReturnDelta(
            abi.encodeCall(IHooks.afterSwap, (msg.sender, key, params, swapDelta, hookData)),
            self.hasPermission(AFTER_SWAP_RETURNS_DELTA_FLAG)
        ).toInt128();
    }

    BalanceDelta hookDelta;
    if (hookDeltaUnspecified != 0 || hookDeltaSpecified != 0) {
        hookDelta = (params.amountSpecified < 0 == params.zeroForOne)
            ? toBalanceDelta(hookDeltaSpecified, hookDeltaUnspecified)
            : toBalanceDelta(hookDeltaUnspecified, hookDeltaSpecified);

        // the caller has to pay for (or receive) the hook's delta
        swapDelta = swapDelta - hookDelta;
    }
    return (swapDelta, hookDelta);
}
```

### beforeDonate

Hooks가 `BEFORE_DONATE_FLAG` 권한을 가진 경우 [callHook](#callhook)을 통해 `beforeDonate` 메서드를 실행합니다.

```solidity
/// @notice calls beforeDonate hook if permissioned and validates return value
function beforeDonate(IHooks self, PoolKey memory key, uint256 amount0, uint256 amount1, bytes calldata hookData)
    internal
    noSelfCall(self)
{
    if (self.hasPermission(BEFORE_DONATE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.beforeDonate, (msg.sender, key, amount0, amount1, hookData)));
    }
}
```

### afterDonate

Hooks가 `AFTER_DONATE_FLAG` 권한을 가진 경우 [callHook](#callhook)을 통해 `afterDonate` 메서드를 실행합니다.

```solidity
/// @notice calls afterDonate hook if permissioned and validates return value
function afterDonate(IHooks self, PoolKey memory key, uint256 amount0, uint256 amount1, bytes calldata hookData)
    internal
    noSelfCall(self)
{
    if (self.hasPermission(AFTER_DONATE_FLAG)) {
        self.callHook(abi.encodeCall(IHooks.afterDonate, (msg.sender, key, amount0, amount1, hookData)));
    }
}
```

### hasPermission

Hooks 주소가 특정 권한을 가지고 있는지, 즉 지정된 비트가 1인지 확인합니다.

```solidity
function hasPermission(IHooks self, uint160 flag) internal pure returns (bool) {
    return uint160(address(self)) & flag != 0;
}
```
