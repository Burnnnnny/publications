# PoolManager

PoolManager는 Uniswap v4의 핵심 컨트랙트로, 싱글톤 컨트랙트 모델을 채택하여 모든 Uniswap v4 풀을 관리하고 생성, 파괴, 유동성 수정, 거래 및 기타 작업을 포함한 풀의 모든 외부 인터페이스를 제공하는 역할을 합니다.

PoolManager의 주요 인터페이스는 다음과 같습니다:

- [unlock](#unlock): 컨트랙트 잠금 해제
- [initialize](#initialize): 풀 초기화
- [modifyLiquidity](#modifyliquidity): 유동성 수정
- [swap](#swap): 토큰 스왑, `token0`를 `token1`으로 또는 그 반대로 교환
- [donate](#donate): 토큰 기부
- [sync](#sync): 토큰 잔액 동기화
- [take](#take): 토큰 인출
- [settle](#settle): 토큰 정산
- [clear](#clear): `PoolManager`에서 인출할 토큰을 포기하고 잔액을 0으로 초기화
- [mint](#mint): ERC6909 토큰을 통해 토큰 인출
- [burn](#burn): ERC6909 토큰을 소각하여 `PoolManager`에 토큰 입금

## 전역 변수 (Global Variables)

싱글톤 컨트랙트로서, 먼저 PoolManager가 모든 풀의 상태를 저장하는 방법에 초점을 맞춥니다. PoolManager는 모든 풀의 상태를 저장하기 위해 전역 변수 `_pools`를 정의합니다.

```solidity
mapping(PoolId id => Pool.State) internal _pools;
```

여기서 `PoolId`는 사용자 정의 타입, 즉 `bytes32`이며 풀의 고유 ID를 식별하는 데 사용됩니다. `PoolId`의 정의는 [PoolId.sol](./PoolIdLibrary_ko.md)에서 찾을 수 있습니다.

## 구조체 정의 (Struct Definitions)

### ModifyLiquidityParams

```solidity
struct ModifyLiquidityParams {
    // the lower and upper tick of the position
    int24 tickLower;
    int24 tickUpper;
    // how to modify the liquidity
    int256 liquidityDelta;
    // a value to set if you want unique liquidity positions at the same range
    bytes32 salt;
}
```

`ModifyLiquidityParams`는 유동성을 수정하는 데 사용되며, 다음을 포함합니다:

- `tickLower`: 포지션의 하한
- `tickUpper`: 포지션의 상한
- `liquidityDelta`: 유동성 변경, 유동성 추가 시 양수, 제거 시 음수
- `salt`: 동일한 범위 내에서 다른 포지션을 구별하는 데 사용됨 (예: ERC721의 토큰 ID를 사용하여 다른 사용자의 포지션을 구별)

### SwapParams

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

`SwapParams`는 거래 매개변수를 정의하는 데 사용되며, 다음을 포함합니다:

- `zeroForOne`: `token0`를 `token1`으로 교환할지 또는 그 반대로 할지 여부
- `amountSpecified`: 거래의 입력 또는 출력 금액, `exactIn`(즉, 입력 토큰 금액)의 경우 음수; `exactOut`(즉, 출력 토큰 금액)의 경우 양수
  * `zeroForOne`이 `true`인 경우:
    * `exactIn`은 정확한 양의 `token0`를 제공함을 의미 (가능한 한 많은 `token1`을 얻기 위함)
    * `exactOut`은 정확한 양의 `token1`을 얻음을 의미 (가능한 한 적은 `token0`를 제공하면서)
  * `zeroForOne`이 `false`인 경우:
    * `exactIn`은 정확한 양의 `token1`을 제공함을 의미 (가능한 한 많은 `token0`를 얻기 위함)
    * `exactOut`은 정확한 양의 `token0`를 얻음을 의미 (가능한 한 적은 `token1`을 제공하면서)
- `sqrtPriceLimitX96`: 거래의 가격 제한, 이 가격에 도달하면 거래가 중지됨
  * `zeroForOne`이 `true`인 경우, 즉 `token0`에서 `token1`으로의 거래인 경우, 거래 후 `token0` ( $x$ )은 증가하고 `token1` ( $y$ )은 감소함, 즉 $\sqrt{P} = \sqrt{\frac{y}{x}}$가 감소하므로 목표 가격 `sqrtPriceLimitX96`은 현재 가격보다 낮아야 함
  * 반대로, 목표 가격 `sqrtPriceLimitX96`은 현재 가격보다 높아야 함

## 함수 정의 (Function Definitions)

### onlyWhenUnlocked

`onlyWhenUnlocked` modifier는 현재 컨트랙트가 `unlocked` 상태인지 확인하는 데 사용되며, 그렇지 않으면 되돌립니다(revert).

Uniswap v4에서 `modifyLiquidity`, `swap`, `mint`, `burn` 등과 같은 [flash accounting](./CurrencyDeltaLibrary_ko.md) 잔액 변경을 포함하는 모든 작업은 `onlyWhenUnlocked` modifier를 사용하여 회계 작업을 허용하기 전에 컨트랙트가 `unlocked` 상태인지 확인합니다.

```solidity
/// @notice This will revert if the contract is locked
modifier onlyWhenUnlocked() {
    if (!Lock.isUnlocked()) ManagerLocked.selector.revertWith();
    _;
}
```

### unlock

`unlock` 함수는 컨트랙트의 잠금을 해제하는 데 사용됩니다. 컨트랙트가 `unlocked` 상태일 때만 플래시 어카운팅 작업을 수행할 수 있습니다.

```solidity
/// @inheritdoc IPoolManager
function unlock(bytes calldata data) external override returns (bytes memory result) {
    if (Lock.isUnlocked()) AlreadyUnlocked.selector.revertWith();

    Lock.unlock();

    // the caller does everything in this callback, including paying what they owe via calls to settle
    result = IUnlockCallback(msg.sender).unlockCallback(data);

    if (NonzeroDeltaCount.read() != 0) CurrencyNotSettled.selector.revertWith();
    Lock.lock();
}
```

호출자(예: 주변 컨트랙트)는 `IUnlockCallback`의 `unlockCallback` 인터페이스를 구현하고 `unlockCallback` 내에서 토큰 전송 작업을 완료하여 호출 종료 시 호출자와 `PoolManager` 컨트랙트 간의 회계 잔액이 0이 되도록 해야 합니다.

`unlock` 함수는 마지막으로 `NonzeroDeltaCount.read()`를 호출하여 0이 아닌 회계 잔액이 있는지 확인합니다. 만약 있다면 전체 컨트랙트가 올바르게 조정되었는지 확인하기 위해 트랜잭션을 되돌립니다.

`unlock` 호출 종료 시, 컨트랙트는 모든 계정의 플래시 어카운팅 잔액이 0인지, 즉 모든 계정이 상환 또는 인출 작업을 완료했는지 확인합니다.

### initialize

풀을 초기화합니다. 이 메서드는 플래시 어카운팅 작업을 포함하지 않으므로 `onlyWhenUnlocked` modifier가 필요하지 않습니다.

매개변수는 다음과 같습니다:

- [PoolKey](./PoolIdLibrary_ko.md#poolkey) memory `key`: 풀의 키, 풀을 고유하게 식별하는 데 사용됨
- uint160 `sqrtPriceX96`: 풀의 초기 가격, 96비트 고정 소수점 숫자 $\sqrt{P}$로 표현됨

반환값:

- int24 `tick`: 풀의 초기 가격에 해당하는 틱

```solidity
/// @inheritdoc IPoolManager
function initialize(PoolKey memory key, uint160 sqrtPriceX96) external noDelegateCall returns (int24 tick) {
    // see TickBitmap.sol for overflow conditions that can arise from tick spacing being too large
    if (key.tickSpacing > MAX_TICK_SPACING) TickSpacingTooLarge.selector.revertWith(key.tickSpacing);
    if (key.tickSpacing < MIN_TICK_SPACING) TickSpacingTooSmall.selector.revertWith(key.tickSpacing);
    if (key.currency0 >= key.currency1) {
        CurrenciesOutOfOrderOrEqual.selector.revertWith(
            Currency.unwrap(key.currency0), Currency.unwrap(key.currency1)
        );
    }
    if (!key.hooks.isValidHookAddress(key.fee)) Hooks.HookAddressNotValid.selector.revertWith(address(key.hooks));

    uint24 lpFee = key.fee.getInitialLPFee();

    key.hooks.beforeInitialize(key, sqrtPriceX96);

    PoolId id = key.toId();

    tick = _pools[id].initialize(sqrtPriceX96, lpFee);

    // event is emitted before the afterInitialize call to ensure events are always emitted in order
    // emit all details of a pool key. poolkeys are not saved in storage and must always be provided by the caller
    // the key's fee may be a static fee or a sentinel to denote a dynamic fee.
    emit Initialize(id, key.currency0, key.currency1, key.fee, key.tickSpacing, key.hooks, sqrtPriceX96, tick);

    key.hooks.afterInitialize(key, sqrtPriceX96, tick);
}
```

* `tickSpacing`의 유효성을 확인하며, $1 \leq tickSpacing \leq 32767$을 만족해야 합니다.

* `currency0`와 `currency1`의 순서를 확인하며, `currency0 < currency1`을 요구합니다. 그중 네이티브 ETH는 `address(0)`로 표현됩니다.

* [isValidHookAddress](./HooksLibrary_ko.md#isvalidhookaddress)를 통해 Hooks 주소의 유효성을 확인하며, `key.hooks`가 유효한 컨트랙트 주소일 것을 요구합니다.

* 초기 `lpFee`를 가져옵니다. 동적 수수료가 있는 풀의 경우 기본 초기 `lpFee`는 0입니다.

* Hooks 라이브러리의 [beforeInitialize](./HooksLibrary_ko.md#beforeinitialize) 함수를 호출합니다.

    > 참고: 이것은 Hooks 컨트랙트의 `beforeInitialize` 메서드에 대한 직접 호출이 아니라, Hooks 라이브러리의 `beforeInitialize` 함수에 대한 호출입니다.

* [initialize](./PoolLibrary_ko.md#initialize) 함수를 통해 풀을 초기화합니다.

* Hooks 라이브러리의 [afterInitialize](./HooksLibrary_ko.md#afterinitialize) 함수를 호출합니다.

### modifyLiquidity

유동성을 수정합니다. 이 메서드는 플래시 어카운팅 작업을 호출하므로 `onlyWhenUnlocked` modifier가 필요합니다.

입력 매개변수는 다음과 같습니다:

- [PoolKey](./PoolIdLibrary_ko.md#poolkey) memory `key`: 풀의 키
- [ModifyLiquidityParams](#modifyliquidityparams) memory `params`: 유동성 수정 매개변수
- bytes calldata `hookData`: Hooks 함수를 위한 데이터

반환값:

- [BalanceDelta](./BalanceDelta_ko.md) `callerDelta`: 호출자의 회계 잔액 변경
- [BalanceDelta](./BalanceDelta_ko.md) `feesAccrued`: 호출자의 수수료 수입

```solidity
/// @inheritdoc IPoolManager
function modifyLiquidity(
    PoolKey memory key,
    IPoolManager.ModifyLiquidityParams memory params,
    bytes calldata hookData
) external onlyWhenUnlocked noDelegateCall returns (BalanceDelta callerDelta, BalanceDelta feesAccrued) {
    PoolId id = key.toId();
    {
        Pool.State storage pool = _getPool(id);
        pool.checkPoolInitialized();

        key.hooks.beforeModifyLiquidity(key, params, hookData);

        BalanceDelta principalDelta;
        (principalDelta, feesAccrued) = pool.modifyLiquidity(
            Pool.ModifyLiquidityParams({
                owner: msg.sender,
                tickLower: params.tickLower,
                tickUpper: params.tickUpper,
                liquidityDelta: params.liquidityDelta.toInt128(),
                tickSpacing: key.tickSpacing,
                salt: params.salt
            })
        );

        // fee delta and principal delta are both accrued to the caller
        callerDelta = principalDelta + feesAccrued;
    }

    // event is emitted before the afterModifyLiquidity call to ensure events are always emitted in order
    emit ModifyLiquidity(id, msg.sender, params.tickLower, params.tickUpper, params.liquidityDelta, params.salt);

    BalanceDelta hookDelta;
    (callerDelta, hookDelta) = key.hooks.afterModifyLiquidity(key, params, callerDelta, feesAccrued, hookData);

    // if the hook doesn't have the flag to be able to return deltas, hookDelta will always be 0
    if (hookDelta != BalanceDeltaLibrary.ZERO_DELTA) _accountPoolBalanceDelta(key, hookDelta, address(key.hooks));

    _accountPoolBalanceDelta(key, callerDelta, msg.sender);
}
```

PoolId를 기반으로 현재 풀 상태를 가져오고 풀이 초기화되었는지 확인합니다.

Hooks 라이브러리의 [beforeModifyLiquidity](./HooksLibrary_ko.md#beforemodifyliquidity) 함수를 호출합니다.

풀의 [pool.modifyLiquidity](./PoolLibrary_ko.md#modifyliquidity) 함수를 호출하여 유동성을 수정합니다.
반환된 `principalDelta`는 호출자의 유동성 변경(수수료 제외)이며, 호출자가 예치해야 할 토큰 양의 경우 음수, 인출할 수 있는 토큰 양의 경우 양수입니다; `feesAccrued`는 호출자가 수집할 수수료입니다.

`callerDelta = principalDelta + feesAccrued;`는 수수료를 포함한 호출자의 회계 잔액 변경을 나타냅니다.

Hooks 라이브러리의 [afterModifyLiquidity](./HooksLibrary_ko.md#aftermodifyliquidity) 함수를 호출하여 Hooks 컨트랙트가 이전 `callerDelta`를 새로운 `callerDelta`와 `hookDelta`로 분할하도록 허용합니다. 따라서 Hooks 컨트랙트는 사용자가 지불하거나 받아야 할 토큰 양을 재할당할 수 있습니다.

* `hookDelta`가 0이 아닌 경우, [_accountPoolBalanceDelta](#_accountpoolbalancedelta) 함수를 호출하여 Hooks 컨트랙트의 잔액 변경을 기록하고 잔액을 Hooks 컨트랙트에 할당합니다;
* [_accountPoolBalanceDelta](#_accountpoolbalancedelta) 함수를 호출하여 호출자의 잔액 변경을 기록하고 잔액을 호출자에게 할당합니다.

참고: `hookDelta`와 `callerDelta`는 각각 Hooks 컨트랙트와 호출자가 정산/수취해야 할 잔액 변경입니다.
* `delta`가 양수이면 Hooks 컨트랙트나 호출자가 토큰을 인출할 수 있습니다;
* `delta`가 음수이면 Hooks 컨트랙트나 호출자가 토큰을 예치해야 합니다;
* 상위 128비트는 `token0`의 잔액 변경을 나타내고, 하위 128비트는 `token1`의 잔액 변경을 나타냅니다.

### swap

토큰을 스왑합니다. 이 메서드는 플래시 어카운팅 작업을 호출하므로 `onlyWhenUnlocked` modifier가 필요합니다.

입력 매개변수는 다음과 같습니다:

- [PoolKey](./PoolIdLibrary_ko.md#poolkey) memory `key`: 풀의 키
- [SwapParams](#swapparams) memory `params`: 거래 매개변수
- bytes calldata `hookData`: Hooks 함수를 위한 데이터

반환값:

- [BalanceDelta](./BalanceDelta_ko.md) swapDelta: 거래의 잔액 변경, 호출자와 `PoolManager` 간의 잔액 관계를 나타냄

```solidity
/// @inheritdoc IPoolManager
function swap(PoolKey memory key, IPoolManager.SwapParams memory params, bytes calldata hookData)
    external
    onlyWhenUnlocked
    noDelegateCall
    returns (BalanceDelta swapDelta)
{
    if (params.amountSpecified == 0) SwapAmountCannotBeZero.selector.revertWith();
    PoolId id = key.toId();
    Pool.State storage pool = _getPool(id);
    pool.checkPoolInitialized();

    BeforeSwapDelta beforeSwapDelta;
    {
        int256 amountToSwap;
        uint24 lpFeeOverride;
        (amountToSwap, beforeSwapDelta, lpFeeOverride) = key.hooks.beforeSwap(key, params, hookData);

        // execute swap, account protocol fees, and emit swap event
        // _swap is needed to avoid stack too deep error
        swapDelta = _swap(
            pool,
            id,
            Pool.SwapParams({
                tickSpacing: key.tickSpacing,
                zeroForOne: params.zeroForOne,
                amountSpecified: amountToSwap,
                sqrtPriceLimitX96: params.sqrtPriceLimitX96,
                lpFeeOverride: lpFeeOverride
            }),
            params.zeroForOne ? key.currency0 : key.currency1 // input token
        );
    }

    BalanceDelta hookDelta;
    (swapDelta, hookDelta) = key.hooks.afterSwap(key, params, swapDelta, hookData, beforeSwapDelta);

    // if the hook doesn't have the flag to be able to return deltas, hookDelta will always be 0
    if (hookDelta != BalanceDeltaLibrary.ZERO_DELTA) _accountPoolBalanceDelta(key, hookDelta, address(key.hooks));

    _accountPoolBalanceDelta(key, swapDelta, msg.sender);
}
```

먼저 입력 매개변수를 확인합니다. `swap` 메서드는 `amountSpecified`를 제공해야 하며, 즉 거래 금액은 0일 수 없습니다.

PoolId를 기반으로 현재 풀 상태를 가져오고 풀이 초기화되었는지 확인합니다.

Hooks 라이브러리의 [beforeSwap](./HooksLibrary_ko.md#beforeswap) 함수를 호출하여 거래 금액, 거래 전 잔액 변경, LP 수수료를 가져옵니다.

`_swap` 함수를 호출하여 거래를 실행하고 거래의 잔액 변경 `swapDelta`를 계산합니다. 이는 호출자가 정산해야 할 토큰 양입니다.

  * `_swap` 함수 내에서 [pool.swap](./PoolLibrary_ko.md#swap) 함수를 호출하여 거래를 실행하고 거래의 잔액 변경 `swapDelta`를 계산합니다.

Hooks 라이브러리의 [afterSwap](./HooksLibrary_ko.md#afterswap) 함수를 호출하여 Hooks 컨트랙트가 `swapDelta`를 새로운 `swapDelta`와 `hookDelta`로 분할하도록 허용합니다.

* `hookDelta`가 0이 아닌 경우, [_accountPoolBalanceDelta](#_accountpoolbalancedelta) 함수를 호출하여 Hooks 컨트랙트의 잔액 변경을 기록합니다.
* [_accountPoolBalanceDelta](#_accountpoolbalancedelta) 함수를 호출하여 호출자의 잔액 변경 `swapDelta`를 기록합니다.

이 두 잔액은 각각 Hooks 컨트랙트와 호출자에 의해 정산되어야 합니다.
* 양수이면 Hooks 컨트랙트나 호출자가 토큰을 인출할 수 있습니다;
* 음수이면 Hooks 컨트랙트나 호출자가 토큰을 예치해야 합니다;
* 상위 128비트는 `token0`의 잔액 변경을 나타내고, 하위 128비트는 `token1`의 잔액 변경을 나타냅니다.

### _accountPoolBalanceDelta

풀의 두 토큰을 기반으로 특정 주소의 플래시 어카운팅 잔액 변경을 기록합니다.

```solidity
/// @notice Accounts the deltas of 2 currencies to a target address
function _accountPoolBalanceDelta(PoolKey memory key, BalanceDelta delta, address target) internal {
    _accountDelta(key.currency0, delta.amount0(), target);
    _accountDelta(key.currency1, delta.amount1(), target);
}
```

### _accountDelta

특정 주소 `target`과 토큰 `currency`의 플래시 어카운팅 잔액 변경을 기록합니다.

```solidity
/// @notice Adds a balance delta in a currency for a target address
function _accountDelta(Currency currency, int128 delta, address target) internal {
    if (delta == 0) return;

    (int256 previous, int256 next) = currency.applyDelta(target, delta);

    if (next == 0) {
        NonzeroDeltaCount.decrement();
    } else if (previous == 0) {
        NonzeroDeltaCount.increment();
    }
}
```

[currency.applyDelta](./CurrencyDeltaLibrary_ko.md#applydelta) 함수를 통해 주소 `target`에 대한 토큰 `currency`의 잔액 변경을 기록합니다. 따라서 플래시 어카운팅은 주소와 토큰을 기반으로 합니다.

수정 전후의 잔액을 반환합니다. `delta != 0`이므로 `previous`와 `next` 중 최대 하나는 0입니다.

수정 후 잔액 `next`가 0이면 `NonzeroDeltaCount.decrement()`를 호출하여 0이 아닌 잔액의 수를 줄이고, 그렇지 않으면 늘립니다.

> 참고: 여기서 `NonzeroDeltaCount`는 임의의 `target` 주소와 `currency`를 기반으로 하므로 전역적입니다.

이전에 소개된 [unlock](#unlock) 함수를 검토하면, `NonzeroDeltaCount.read()`는 호출 종료 시 0이 아닌 잔액의 수를 확인하는 데 사용됩니다.

### donate

풀에 토큰을 기부합니다. 이 메서드는 플래시 어카운팅 작업을 호출하므로 `onlyWhenUnlocked` modifier가 필요합니다.

```solidity
/// @inheritdoc IPoolManager
function donate(PoolKey memory key, uint256 amount0, uint256 amount1, bytes calldata hookData)
    external
    onlyWhenUnlocked
    noDelegateCall
    returns (BalanceDelta delta)
{
    PoolId poolId = key.toId();
    Pool.State storage pool = _getPool(poolId);
    pool.checkPoolInitialized();

    key.hooks.beforeDonate(key, amount0, amount1, hookData);

    delta = pool.donate(amount0, amount1);

    _accountPoolBalanceDelta(key, delta, msg.sender);

    // event is emitted before the afterDonate call to ensure events are always emitted in order
    emit Donate(poolId, msg.sender, amount0, amount1);

    key.hooks.afterDonate(key, amount0, amount1, hookData);
}
```

PoolId를 기반으로 현재 풀 상태를 가져오고 풀이 초기화되었는지 확인합니다.

Hooks 라이브러리의 [beforeDonate](./HooksLibrary_ko.md#beforedonate) 함수를 호출합니다.

풀의 [pool.donate](./PoolLibrary_ko.md#donate) 함수를 호출하여 토큰을 기부하고 잔액 변경 `delta`를 반환합니다.

참고: 여기서 `amount0`와 `amount1`은 모두 `uint256` 타입입니다. `pool.donate` 함수 내에서 `amount0`와 `amount1`은 음수로 변환되어 `delta`의 `amount0`와 `amount1`이 음수가 되도록 하여, 호출자가 예치해야 할 토큰 양을 나타냅니다.

[_accountPoolBalanceDelta](#_accountpoolbalancedelta) 함수를 호출하여 호출자의 잔액 변경을 기록합니다.

Hooks 라이브러리의 [afterDonate](./HooksLibrary_ko.md#afterdonate) 함수를 호출합니다.

### sync

`PoolManager` 내의 지정된 ERC20 토큰의 현재 잔액을 `임시 저장(transient storage)`으로 동기화합니다. 이 메서드는 토큰이 전송된 금액을 계산하기 위해 토큰을 `PoolManager` 컨트랙트로 보내기 전에 호출해야 합니다.

> 참고: 네이티브 ETH의 잔액을 정산하려면 이 메서드를 먼저 호출할 수도 있으며, 이 경우 `currency`는 `address(0)`입니다.

```solidity
/// @inheritdoc IPoolManager
function sync(Currency currency) external {
    // address(0) is used for the native currency
    if (currency.isAddressZero()) {
        // The reserves balance is not used for native settling, so we only need to reset the currency.
        CurrencyReserves.resetCurrency();
    } else {
        uint256 balance = currency.balanceOfSelf();
        CurrencyReserves.syncCurrencyAndReserves(currency, balance);
    }
}
```

`currency`가 네이티브 ETH인 경우 `currency` 주소를 재설정합니다.

그렇지 않으면 컨트랙트(`PoolManager`) 내의 `currency` 토큰의 현재 잔액을 가져오고 토큰 주소와 잔액을 `임시 저장`에 동기화합니다.

### take

`PoolManager`에서 지정된 양의 토큰을 인출하여 지정된 주소로 보냅니다.

```solidity
/// @inheritdoc IPoolManager
function take(Currency currency, address to, uint256 amount) external onlyWhenUnlocked {
    unchecked {
        // negation must be safe as amount is not negative
        _accountDelta(currency, -(amount.toInt128()), msg.sender);
        currency.transfer(to, amount);
    }
}
```

이 작업은 먼저 [_accountDelta](#_accountdelta) 함수를 호출하여 플래시 어카운팅 잔액에서 인출된 토큰 금액을 차감한 다음, `currency.transfer`를 호출하여 토큰을 지정된 주소로 전송합니다.

> 참고: 이 메서드는 플래시 론 작업을 완료하는 데 사용될 수 있습니다.

### settle

부채를 정산합니다.

`PoolManager`는 호출자 `msg.sender`와 지정된 주소 `target`에 대해 `PoolManager`에게 빚진 토큰 금액을 정산하기 위해 `settle` 및 `settleFor` 메서드를 모두 제공합니다. 두 메서드 모두 [_settle](#_settle) 메서드를 호출합니다.

```solidity
/// @inheritdoc IPoolManager
function settle() external payable onlyWhenUnlocked returns (uint256) {
    return _settle(msg.sender);
}

/// @inheritdoc IPoolManager
function settleFor(address recipient) external payable onlyWhenUnlocked returns (uint256) {
    return _settle(recipient);
}
```

일반적인 정산 프로세스는 다음과 같습니다:

1. [sync](#sync) 메서드를 호출하여 토큰 주소와 잔액을 동기화합니다.
2. 토큰을 전송합니다.
3. [settle](#settle) 또는 `settleFor` 메서드를 호출하여 토큰 잔액을 정산합니다.

따라서 이 메서드를 호출하기 전에 외부 컨트랙트는 [sync](#sync) 메서드를 호출하여 토큰 주소와 잔액을 동기화해야 합니다.

### _settle

지정된 주소에 대한 부채를 정산합니다.

```solidity
// if settling native, integrators should still call `sync` first to avoid DoS attack vectors
function _settle(address recipient) internal returns (uint256 paid) {
    Currency currency = CurrencyReserves.getSyncedCurrency();

    // if not previously synced, or the syncedCurrency slot has been reset, expects native currency to be settled
    if (currency.isAddressZero()) {
        paid = msg.value;
    } else {
        if (msg.value > 0) NonzeroNativeValue.selector.revertWith();
        // Reserves are guaranteed to be set because currency and reserves are always set together
        uint256 reservesBefore = CurrencyReserves.getSyncedReserves();
        uint256 reservesNow = currency.balanceOfSelf();
        paid = reservesNow - reservesBefore;
        CurrencyReserves.resetCurrency();
    }

    _accountDelta(currency, paid.toInt128(), recipient);
}
```

`CurrencyReserves.getSyncedCurrency()`를 통해 정산할 토큰 주소 `currency`를 가져옵니다. 이 주소는 [sync](#sync) 메서드를 통해 동기화되었습니다.

`currency`가 네이티브 ETH인 경우 `msg.value`에서 직접 정산할 금액을 가져옵니다.

그렇지 않으면 `PoolManager` 내의 `currency` 토큰 잔액을 가져와 정산 전 잔액(`sync`)과 비교하고 정산 금액(즉, 전송된 토큰 금액)을 계산합니다. 여기서 정산 금액 `paid`는 양수 또는 0이어야 합니다.

[_accountDelta](#_accountdelta) 함수를 호출하여 `recipient` 주소의 잔액 변경을 기록합니다.

### clear

`clear` 메서드는 호출자가 `PoolManager`에서 인출할 토큰을 포기하고 잔액을 0으로 초기화하는 데 사용됩니다.

이 메서드는 일반적으로 호출자가 소량의 더스트 토큰을 포기하는 데 사용됩니다. 이러한 토큰을 가져가는 데 드는 거래 수수료(예: `transfer token`에 소비되는 가스)가 토큰 자체의 가치보다 높을 수 있기 때문입니다.

입력 매개변수:

- Currency `currency`: 초기화할 토큰 주소.
- uint256 `amount`: 초기화할 토큰 금액.

```solidity
/// @inheritdoc IPoolManager
function clear(Currency currency, uint256 amount) external onlyWhenUnlocked {
    int256 current = currency.getDelta(msg.sender);
    // Because input is `uint256`, only positive amounts can be cleared.
    int128 amountDelta = amount.toInt128();
    if (amountDelta != current) MustClearExactPositiveDelta.selector.revertWith();
    // negation must be safe as amountDelta is positive
    unchecked {
        _accountDelta(currency, -(amountDelta), msg.sender);
    }
}
```

`currency.getDelta(msg.sender)`를 통해 호출자 `msg.sender`의 회계 잔액을 가져옵니다.

초기화할 토큰 금액 `amount`가 현재 잔액과 일치하지 않으면 되돌립니다(revert).

`amount`가 음수가 아니므로 호출자는 토큰을 인출할 권리만 포기할 수 있습니다.

`_accountDelta` 함수를 호출하여 호출자의 회계 잔액을 초기화합니다.

### mint

[ERC6909](https://eips.ethereum.org/EIPS/eip-6909) `mint token`을 통해 호출자 `msg.sender`에게 토큰을 발행하여 `transfer token` 작업을 피합니다. `PoolManager` 컨트랙트와 자주 상호 작용해야 하는 호출자의 경우, 이 메서드는 각 트랜잭션에 대한 최종 토큰 전송 작업을 피함으로써 가스 소비를 크게 줄일 수 있습니다.

입력 매개변수:

- `address` to: 토큰을 받을 주소.
- `uint256` id: 토큰의 ID, 실제로는 토큰 주소 `address`의 uint160 값.
- `uint256` amount: 토큰 금액, 음수가 아니어야 함.

```solidity
    /// @inheritdoc IPoolManager
    function mint(address to, uint256 id, uint256 amount) external onlyWhenUnlocked {
        unchecked {
            Currency currency = CurrencyLibrary.fromId(id);
            // negation must be safe as amount is not negative
            _accountDelta(currency, -(amount.toInt128()), msg.sender);
            _mint(to, currency.toId(), amount);
        }
    }
```

`CurrencyLibrary.fromId`를 통해 토큰 주소 `address`를 가져옵니다.

`amount`가 음수가 아니므로 회계 잔액에서 인출된 토큰 금액, 즉 `-amount`를 차감한 다음 `_mint` 함수를 호출하여 지정된 주소로 ERC6909 토큰을 발행합니다(잔액 변경만 기록하고 `transfer token` 작업은 없음).

### burn

[mint](#mint)와 유사하게 `burn`은 [ERC6909](https://eips.ethereum.org/EIPS/eip-6909) `burn token`을 사용하여 이미 인출된 토큰을 파괴하는 동시에 회계 잔액에서 해당 토큰 금액을 증가시켜 `transfer token` 작업을 피합니다.

입력 매개변수:

- `address` from: `ERC6909` 토큰을 보유한 `owner` 주소.
- `uint256` id: 토큰의 ID, 실제로는 토큰 주소 `address`의 uint160 값.
- `uint256` amount: 토큰 금액, 음수가 아니어야 함.

```solidity
/// @inheritdoc IPoolManager
function burn(address from, uint256 id, uint256 amount) external onlyWhenUnlocked {
    Currency currency = CurrencyLibrary.fromId(id);
    _accountDelta(currency, amount.toInt128(), msg.sender);
    _burnFrom(from, currency.toId(), amount);
}
```

`CurrencyLibrary.fromId`를 통해 토큰 주소 `address`를 가져옵니다.

`amount`가 음수가 아니므로 회계 잔액에서 호출자가 인출해야 할 토큰 금액을 증가시킨 다음, `_burnFrom` 함수를 호출하여 `owner` 주소에서 해당 양의 ERC6909 토큰을 파괴합니다(잔액 변경만 기록하고 `transfer token` 작업은 없음).

### updateDynamicLPFee

동적 LP 수수료를 업데이트합니다. 동적 수수료가 있는 풀과 Hooks 컨트랙트만 이 메서드를 호출할 수 있습니다.

```solidity
/// @inheritdoc IPoolManager
function updateDynamicLPFee(PoolKey memory key, uint24 newDynamicLPFee) external {
    if (!key.fee.isDynamicFee() || msg.sender != address(key.hooks)) {
        UnauthorizedDynamicLPFeeUpdate.selector.revertWith();
    }
    newDynamicLPFee.validate();
    PoolId id = key.toId();
    _pools[id].setLPFee(newDynamicLPFee);
}
```

수수료가 동적 수수료인지, 호출자가 Hooks 컨트랙트인지 확인합니다.

새로운 동적 LP 수수료가 유효한지 확인합니다.

풀의 [setLPFee](./PoolLibrary_ko.md#setlpfee) 함수를 호출하여 동적 LP 수수료를 업데이트합니다.
