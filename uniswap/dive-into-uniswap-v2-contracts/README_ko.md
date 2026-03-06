[English](./README.md) | [中文](./README_zh.md)

# Uniswap v2 스마트 컨트랙트 심층 분석

###### 태그: `uniswap` `uniswap-v2` `smart contract` `solidity`

[Uniswap v2 백서 심층 분석](../dive-into-uniswap-v2-whitepaper/README_ko.md) 소개에 이어, 오늘은 Uniswap v2 컨트랙트 코드를 살펴보겠습니다.

> 이 글은 컨트랙트 코드를 한 줄씩 다루지는 않으며, 컨트랙트 아키텍처와 주요 메서드에 집중할 것입니다. 자세한 코드 설명은 이더리움 공식 블로그인 [Uniswap v2 contract walk-through](https://ethereum.org/en/developers/tutorials/uniswap-v2-annotated-code/#introduction)를 읽어보시기를 권장합니다.

## 컨트랙트 아키텍처 (Contract Architecture)

Uniswap v2 컨트랙트는 크게 두 가지 범주인 핵심(core) 컨트랙트와 주변(periphery) 컨트랙트로 나뉩니다. 핵심 컨트랙트는 가장 기본적인 거래 기능만 포함하며, 핵심 코드는 약 200줄 정도로 사용자의 자금이 이 컨트랙트에 저장되므로 버그 도입을 피하기 위해 최소화되었습니다. 주변 컨트랙트는 사용자의 시나리오에 맞춘 다양한 캡슐화된 메서드를 제공하며, 예를 들어 네이티브 ETH 거래 지원(자동으로 WETH로 변환), 다중 경로 스왑(A→B→C 거래를 하나의 메서드로 실행) 등을 핵심 컨트랙트를 호출하여 수행합니다. [app.uniswap.org](https://app.uniswap.org/) 인터페이스에서의 작업은 주변 컨트랙트를 활용합니다.

![](./assets/uniswap-v2.png)

몇 가지 주요 컨트랙트의 기능을 소개하겠습니다:

* uniswap-v2-core
  * UniswapV2Factory: Pair 컨트랙트를 생성하는(그리고 프로토콜 수수료 수취 주소를 설정하는) 팩토리 컨트랙트
  * UniswapV2Pair: 거래 쌍(Pair) 컨트랙트로, swap/mint/burn, 가격 오라클 등 거래와 관련된 몇 가지 기본 메서드를 정의하며, 그 자체로 UniswapV2ERC20을 상속받는 ERC20 컨트랙트임
  * UniswapV2ERC20: ERC20 표준 메서드를 구현

* uniswap-v2-periphery
  * UniswapV2Router02: 라우터 컨트랙트의 최신 버전으로, UniswapV2Router01과 비교하여 FeeOnTransfer 토큰 지원이 추가됨; 유동성 추가/제거, 토큰 A를 B로 스왑, ETH를 토큰으로 스왑 등 Uniswap v2에서 가장 일반적으로 사용되는 인터페이스를 구현
  * UniswapV1Router01: 이전 버전의 Router 구현으로, Router02와 유사하지만 FeeOnTransferTokens를 지원하지 않으며 더 이상 사용되지 않음

## uniswap-v2-core

[Github](https://github.com/Uniswap/v2-core)

### UniswapV2Factory

팩토리 컨트랙트에서 가장 중요한 메서드는 `createPair`입니다:

```solidity
function createPair(address tokenA, address tokenB) external returns (address pair) {
    require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES');
    (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
    require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS');
    require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); // single check is sufficient
    bytes memory bytecode = type(UniswapV2Pair).creationCode;
    bytes32 salt = keccak256(abi.encodePacked(token0, token1));
    assembly {
        pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
    }
    IUniswapV2Pair(pair).initialize(token0, token1);
    getPair[token0][token1] = pair;
    getPair[token1][token0] = pair; // populate mapping in the reverse direction
    allPairs.push(pair);
    emit PairCreated(token0, token1, pair, allPairs.length);
}
```

먼저 `token0`와 `token1`을 정렬하여 `token0`의 리터럴 주소가 `token1`보다 작도록 합니다. 그런 다음 `assembly` + `create2`를 사용하여 컨트랙트를 생성합니다. [`assembly`](https://docs.soliditylang.org/en/develop/assembly.html#inline-assembly)는 [Yul](https://docs.soliditylang.org/en/develop/yul.html#yul) 언어를 사용하여 Solidity에서 EVM을 직접 조작할 수 있게 해주는 저수준 작업 방식입니다. 백서에서 논의된 바와 같이, `create2`는 주로 결정론적 거래 쌍 컨트랙트 주소를 생성하는 데 사용되며, 이는 온체인 컨트랙트 쿼리 없이 두 토큰 주소에서 직접 페어 주소를 계산할 수 있음을 의미합니다.

`CREATE2`는 [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014)에서 유래했으며, 이에 따르면 최종 생성된 주소는 페어 컨트랙트 생성 시 제공된 커스텀 `salt` 값의 영향을 받습니다. 두 토큰의 거래 쌍에 대해 `salt` 값은 일관되어야 하므로, 자연스럽게 거래 쌍의 두 토큰 주소를 사용하는 것이 떠오릅니다. 순서가 페어에 영향을 미치지 않도록 하기 위해 컨트랙트는 두 토큰을 정렬하여 오름차순으로 `salt` 값을 생성합니다.

최신 EVM 버전에서는 `new` 메서드에 `salt` 파라미터를 직접 전달하는 것을 지원합니다:

```solidity
pair = new UniswapV2Pair{salt: salt}();
```

Uniswap v2 컨트랙트 개발 당시에는 이 기능이 없었기 때문에 `assembly create2`가 사용되었습니다.

[Yul 사양](https://docs.soliditylang.org/en/develop/yul.html#yul)에 따르면 `create2`는 다음과 같이 정의됩니다:

> create2(v, p, n, s)
> 
> create new contract with code mem[p…(p+n)) at address keccak256(0xff . this . s . keccak256(mem[p…(p+n))) and send v wei and return the new address, where 0xff is a 1 byte value, this is the current contract’s address as a 20 byte value and s is a big-endian 256-bit value; returns 0 on error

소스 코드에서 `create2` 메서드 호출은 다음과 같습니다:

```solidity
pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
```

파라미터는 다음과 같이 해석됩니다:

- v=0: 새로 생성된 페어 컨트랙트로 전송된 ETH 토큰의 양(wei 단위)
- p=add(bytecode, 32): 컨트랙트 바이트코드의 시작 위치
  > 왜 32를 더할까요? `bytecode` 타입이 `bytes`이고, ABI 사양에 따라 `bytes`는 가변 길이 타입이기 때문입니다. 처음 32바이트는 `bytecode`의 길이를 저장하고 그 뒤에 실제 내용이 오므로, 실제 컨트랙트 바이트코드의 시작은 `bytecode+32` 바이트 위치입니다.
- n=mload(bytecode): 컨트랙트 바이트코드의 총 바이트 길이
  > 언급했듯이 `bytecode`의 처음 32바이트는 컨트랙트 바이트코드의 실제 길이(바이트 단위)를 저장하며, `mload`는 전달된 파라미터의 처음 32바이트 값을 가져오므로 `mload(bytecode)`는 `n`과 같습니다.
- s=salt: 커스텀 `salt`로, `token0`와 `token1` 주소를 인코딩하여 결합한 것입니다.

### UniswapV2ERC20

이 컨트랙트는 주로 UniswapV2의 ERC20 표준 구현을 정의하며, 코드는 비교적 간단합니다. 여기서는 `permit` 메서드를 소개하겠습니다:

```solidity
function permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s) external {
    require(deadline >= block.timestamp, 'UniswapV2: EXPIRED');
    bytes32 digest = keccak256(
        abi.encodePacked(
            '\x19\x01',
            DOMAIN_SEPARATOR,
            keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, nonces[owner]++, deadline))
        )
    );
    address recoveredAddress = ecrecover(digest, v, r, s);
    require(recoveredAddress != address(0) && recoveredAddress == owner, 'UniswapV2: INVALID_SIGNATURE');
    _approve(owner, spender, value);
}
```

`permit` 메서드는 백서의 2.5절에서 소개된 "풀 지분을 위한 메타 트랜잭션" 기능을 구현합니다. [EIP-712](https://eips.ethereum.org/EIPS/eip-712)는 오프체인 서명 표준, 즉 사용자가 서명하는 `digest`의 형식을 정의합니다. 서명의 내용은 소유자(owner)가 컨트랙트(`spender`)에게 마감 기한(deadline) 전에 일정 금액(`value`)의 토큰(Pair 유동성 토큰)을 사용할 수 있도록 승인(`approve`)하는 것입니다. 애플리케이션(주변 컨트랙트)은 원본 정보와 생성된 v, r, s 서명을 사용하여 Pair 컨트랙트의 `permit` 메서드를 호출하여 승인을 얻을 수 있습니다. `permit` 메서드는 `ecrecover`를 사용하여 서명 주소를 토큰 소유자로 복원하며, 검증이 통과되면 승인이 부여됩니다.

### UniswapV2Pair

Pair 컨트랙트는 주로 `mint`(유동성 추가), `burn`(유동성 제거), `swap`(교환) 세 가지 메서드를 구현합니다.

#### mint

이 메서드는 유동성 추가 기능을 구현합니다.

```solidity
// this low-level function should be called from a contract which performs important safety checks
function mint(address to) external lock returns (uint liquidity) {
    (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
    uint balance0 = IERC20(token0).balanceOf(address(this));
    uint balance1 = IERC20(token1).balanceOf(address(this));
    uint amount0 = balance0.sub(_reserve0);
    uint amount1 = balance1.sub(_reserve1);

    bool feeOn = _mintFee(_reserve0, _reserve1);
    uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
    if (_totalSupply == 0) {
        liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
        _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
    } else {
        liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
    }
    require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
    _mint(to, liquidity);

    _update(balance0, balance1, _reserve0, _reserve1);
    if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
    emit Mint(msg.sender, amount0, amount1);
}
```

처음에 ```getReserves()```는 두 토큰의 캐시된 잔액을 가져옵니다. 백서에서 언급했듯이 잔액 캐싱은 가격 오라클 조작을 방지하기 위함입니다. 또한 여기서는 현재 잔액에서 캐시된 잔액을 빼서 전송된 토큰 양을 결정하여 프로토콜 수수료를 계산하는 데 사용됩니다.

_mintFee는 프로토콜 수수료를 계산합니다:

```solidity
// if fee is on, mint liquidity equivalent to 1/6th of the growth in sqrt(k)
function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
    address feeTo = IUniswapV2Factory(factory).feeTo();
    feeOn = feeTo != address(0);
    uint _kLast = kLast; // gas savings
    if (feeOn) {
        if (_kLast != 0) {
            uint rootK = Math.sqrt(uint(_reserve0).mul(_reserve1));
            uint rootKLast = Math.sqrt(_kLast);
            if (rootK > rootKLast) {
                uint numerator = totalSupply.mul(rootK.sub(rootKLast));
                uint denominator = rootK.mul(5).add(rootKLast);
                uint liquidity = numerator / denominator;
                if (liquidity > 0) _mint(feeTo, liquidity);
            }
        }
    } else if (_kLast != 0) {
        kLast = 0;
    }
}
```

프로토콜 수수료 계산에 대해서는 백서를 참조하세요.

`mint` 메서드는 거래 쌍에 처음으로 유동성을 공급하는 경우 xy의 제곱근을 기준으로 유동성 토큰을 생성하고 MINIMUM_LIQUIDITY(즉, 1000 wei)를 소각한다고 결정합니다; 그렇지 않은 경우 전송된 토큰 가치 대 현재 유동성 가치의 비율을 기준으로 유동성 토큰을 발행합니다.

#### burn

이 메서드는 유동성 제거 기능을 구현합니다.

```solidity
// this low-level function should be called from a contract which performs important safety checks
function burn(address to) external lock returns (uint amount0, uint amount1) {
    (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
    address _token0 = token0;                                // gas savings
    address _token1 = token1;                                // gas savings
    uint balance0 = IERC20(_token0).balanceOf(address(this));
    uint balance1 = IERC20(_token1).balanceOf(address(this));
    uint liquidity = balanceOf[address(this)];

    bool feeOn = _mintFee(_reserve0, _reserve1);
    uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
    amount0 = liquidity.mul(balance0) / _totalSupply; // using balances ensures pro-rata distribution
    amount1 = liquidity.mul(balance1) / _totalSupply; // using balances ensures pro-rata distribution
    require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED');
    _burn(address(this), liquidity);
    _safeTransfer(_token0, to, amount0);
    _safeTransfer(_token1, to, amount1);
    balance0 = IERC20(_token0).balanceOf(address(this));
    balance1 = IERC20(_token1).balanceOf(address(this));

    _update(balance0, balance1, _reserve0, _reserve1);
    if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
    emit Burn(msg.sender, amount0, amount1, to);
}
```

`mint`와 유사하게, `burn` 메서드도 프로토콜 수수료를 계산합니다.

백서를 참조하면, 트랜잭션 수수료를 절약하기 위해 Uniswap v2는 유동성을 발행/소각할 때만 누적된 프로토콜 수수료를 징수합니다.

유동성을 제거한 후, 전체 토큰에 대한 소각된 유동성 토큰의 비율이 사용자가 받게 될 두 토큰의 해당 양을 결정합니다.

#### swap

이 메서드는 두 토큰을 교환(거래)하는 기능을 구현합니다.

```solidity
// this low-level function should be called from a contract which performs important safety checks
function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
    require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
    (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
    require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');

    uint balance0;
    uint balance1;
    { // scope for _token{0,1}, avoids stack too deep errors
    address _token0 = token0;
    address _token1 = token1;
    require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO');
    if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
    if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
    if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
    balance0 = IERC20(_token0).balanceOf(address(this));
    balance1 = IERC20(_token1).balanceOf(address(this));
    }
    uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
    uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
    require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
    { // scope for reserve{0,1}Adjusted, avoids stack too deep errors
    uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
    uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
    require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');
    }

    _update(balance0, balance1, _reserve0, _reserve1);
    emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
}
```

플래시 론 기능을 수용하고 특정 토큰의 전송 메서드에 의존하지 않기 위해, 전체 `swap` 메서드는 `amountIn`과 같은 파라미터를 사용하지 않고, 현재 잔액과 캐시된 잔액을 비교하여 전송된 토큰의 양을 계산합니다.

`swap` 메서드는 상수 곱 공식 제약 조건(백서 공식 참조)을 준수하는지 확인(수수료 차감 후 잔액 확인)하므로, 사용자가 이전에 거래를 위해 토큰을 컨트랙트로 전송하지 않았다면 컨트랙트는 먼저 사용자가 받고자 하는 토큰을 전송할 수 있습니다. 이는 토큰을 빌리는 것(즉, 플래시 론)과 같으며, 플래시 론을 사용하는 경우 커스텀 `uniswapV2Call` 메서드에서 빌린 토큰을 반환해야 하며, 그렇지 않으면 트랜잭션이 revert됩니다.

`swap` 메서드는 캐시된 잔액을 사용하여 가격 오라클에 필요한 누적 가격을 업데이트하고, 메서드 마지막에 캐시된 잔액을 현재 잔액으로 업데이트합니다.

```solidity
// update reserves and, on the first call per block, price accumulators
function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
    require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
    uint32 blockTimestamp = uint32(block.timestamp % 2**32);
    uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
    if (timeElapsed > 0 and _reserve0 != 0 and _reserve1 != 0) {
        // * never overflows, and + overflow is desired
        price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
        price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
    }
    reserve0 = uint112(balance0);
    reserve1 = uint112(balance1);
    blockTimestampLast = blockTimestamp;
    emit Sync(reserve0, reserve1);
}
```

참고로 블록 타임스탬프와 누적 가격 모두 오버플로우에 안전합니다. (자세한 유도 과정은 백서를 참조하세요)

## uniswap-v2-periphery

UniswapV2Router01은 FeeOnTransferTokens 처리에 버그가 있어 더 이상 사용되지 않습니다. 여기서는 최신 버전인 UniswapV2Router02 컨트랙트만 소개하겠습니다.

[코드 주소](https://github.com/Uniswap/v2-periphery)

### UniswapV2Router02

Router02는 가장 일반적으로 사용되는 거래 인터페이스를 캡슐화합니다; 네이티브 ETH 거래 필요성을 수용하기 위해 대부분의 인터페이스는 ETH 버전으로 제공됩니다. Router01과 비교하여 일부 인터페이스는 FeeOnTransferTokens에 대한 지원을 추가했습니다.

![](./assets/router02.png)

ETH 버전의 로직은 동일하며 ETH와 WETH 간의 변환만 포함하므로 주로 ERC20 버전의 코드를 소개하겠습니다.

구체적인 ERC20 메서드를 살펴보기 전에, Library 컨트랙트의 몇 가지 공통 메서드와 수학적 유도를 먼저 소개하겠습니다.

### Library

[코드 주소](https://github.com/Uniswap/v2-periphery/blob/master/contracts/libraries/UniswapV2Library.sol)

#### pairFor

팩토리 주소와 두 토큰 주소를 입력하여 거래 쌍의 주소를 계산합니다.

```solidity
// calculates the CREATE2 address for a pair without making any external calls
function pairFor(address factory, address tokenA, address tokenB) internal pure returns (address pair) {
    (address token0, address token1) = sortTokens(tokenA, tokenB);
    pair = address(uint(keccak256(abi.encodePacked(
            hex'ff',
            factory,
            keccak256(abi.encodePacked(token0, token1)),
            hex'96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f' // init code hash
        ))));
}
```

앞서 언급했듯이 `CREATE2` 옵코드가 사용되므로 온체인 컨트랙트 쿼리 없이 페어 주소를 직접 계산할 수 있습니다.

> create2(v, p, n, s)
>
> create new contract with code mem[p…(p+n)) at address keccak256(0xff . this . s . keccak256(mem[p…(p+n))) and send v wei and return the new address, where 0xff is a 1 byte value, this is the current contract’s address as a 20 byte value and s is a big-endian 256-bit value; returns 0 on error

새 페어 컨트랙트의 주소는 `keccak256(0xff + this + salt +  keccak256(mem[p…(p+n)))`로 계산됩니다:

* this: 팩토리 컨트랙트 주소
* salt: `keccak256(abi.encodePacked(token0, token1))`
* `keccak256(mem[p…(p+n))`: `0x96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f`

모든 거래 쌍은 UniswapV2Pair 컨트랙트를 사용하여 생성되므로 init code hash는 동일합니다. 해시를 계산하기 위해 UniswapV2Factory에 Solidity 메서드를 작성할 수 있습니다:

```solidity
function initCodeHash() external pure returns (bytes32) {
    bytes memory bytecode = type(UniswapV2Pair).creationCode;
    bytes32 hash;
    assembly {
        hash := keccak256(add(bytecode, 32), mload(bytecode))
    }
    return hash;
}
```

#### quote

`quote` 메서드는 컨트랙트의 준비금(reserves)을 기반으로 일정 양(`amountA`)의 토큰 A를 동등한 양의 토큰 B로 변환합니다. 이는 단순한 단위 변환이므로 수수료는 고려되지 않습니다.

```solidity
// given some amount of an asset and pair reserves, returns an equivalent amount of the other asset
function quote(uint amountA, uint reserveA, uint reserveB) internal pure returns (uint amountB) {
    require(amountA > 0, 'UniswapV2Library: INSUFFICIENT_AMOUNT');
    require(reserveA > 0 && reserveB > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
    amountB = amountA.mul(reserveB) / reserveA;
}
```

#### getAmountOut

이 메서드는 풀의 준비금에 기초하여 주어진 양의 토큰 A(`amountIn`)에 대해 얻을 수 있는 토큰 B의 양(`amountOut`)을 계산합니다.

```solidity
// given an input amount of an asset and pair reserves, returns the maximum output amount of the other asset
function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut) internal pure returns (uint amountOut) {
    require(amountIn > 0, 'UniswapV2Library: INSUFFICIENT_INPUT_AMOUNT');
    require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
    uint amountInWithFee = amountIn.mul(997);
    uint numerator = amountInWithFee.mul(reserveOut);
    uint denominator = reserveIn.mul(1000).add(amountInWithFee);
    amountOut = numerator / denominator;
}
```

이 메서드의 수학 공식 유도를 위해 백서와 핵심 컨트랙트에서 논의된 스왑 후 토큰 잔액 제약 조건을 다시 살펴봅시다:

$$
(x_1 - 0.003 \cdot x_{in}) \cdot (y_1 - 0.003 \cdot y_{in}) \geq x_0 \cdot y_0
$$

여기서 $x_0$, $y_0$은 스왑 전 두 토큰의 잔액이고, $x_1$, $y_1$은 스왑 후 잔액이며, $x_{in}$은 제공된 토큰 A의 양(토큰 A가 제공되므로 $y_{in}=0$), $y_{out}$은 계산될 토큰 B의 양입니다.

수학적 유도 과정은 다음과 같습니다:

> $y_{in} = 0$
>
> $x_1 = x_0 + x_{in}$
>
> $y_1 = y_0 - y_{out}$
>
> $(x_1 - 0.003 \cdot x_{in}) \cdot (y_1 - 0.003 \cdot y_{in}) = x_0 \cdot y_0$
>
> $(x_1 - 0.003 \cdot x_{in}) \cdot y_1 = x_0 \cdot y_0$
>
> $(x_0 + x_{in} - 0.003 \cdot x_{in}) \cdot (y_0 - y_{out}) = x_0 \cdot y_0$
>
> $(x_0 + 0.997 \cdot x_{in}) \cdot (y_0 - y_{out}) = x_0 \cdot y_0$
>
> $y_{out} = y_0 - \frac {x_0 \cdot y_0}{x_0 + 0.997 \cdot x_{in}}$
>
> $y_{out} = \frac {0.997 \cdot x_{in} \cdot y_0}{x_0 + 0.997 \cdot x_{in}}$

Solidity는 부동 소수점 숫자를 지원하지 않으므로 공식을 다음과 같이 변환할 수 있습니다:

$$
y_{out} = \frac {997 \cdot x_{in} \cdot y_0}{1000 \cdot x_0 + 997 \cdot x_{in}}
$$

이 계산 결과는 `getAmountOut` 메서드에 표시된 `amountOut`과 같으며, 여기서:

> $amountIn = x_{in}$
>
> $reserveIn = x_0$
>
> $reserveOut = y_0$
>
> $amountOut = y_{out}$

#### getAmountIn

이 메서드는 지정된 양의 토큰 B(`amountOut`)를 얻기 위해 필요한 토큰 A의 양(`amountIn`)을 계산합니다.

```solidity
// given an output amount of an asset and pair reserves, returns a required input amount of the other asset
function getAmountIn(uint amountOut, uint reserveIn, uint reserveOut) internal pure returns (uint amountIn) {
    require(amountOut > 0, 'UniswapV2Library: INSUFFICIENT_OUTPUT_AMOUNT');
    require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
    uint numerator = reserveIn.mul(amountOut).mul(1000);
    uint denominator = reserveOut.sub(amountOut).mul(997);
    amountIn = (numerator / denominator).add(1);
}
```

`getAmountOut`은 $x_{in}$이 주어졌을 때 $y_{out}$을 계산하고, 이에 대응하여 `getAmountIn`은 $y_{out}$이 주어졌을 때 $x_{in}$을 계산합니다. 유도된 공식은 다음과 같습니다:

$$
(x_0 + 0.997 \cdot x_{in}) \cdot (y_0 - y_{out}) = x_0 \cdot y_0\\
x_{in} = \frac {\frac {x_0 \cdot y_0}{y_0 - y_{out}} - x_0} {0.997}
$$

$$
x_{in} = \frac {x_0 \cdot y_{out}}{0.997 \cdot (y_0 - y_{out})} = \frac {1000 \cdot x_0 \cdot y_{out}}{997 \cdot (y_0 - y_{out})}
$$

$$
amountIn = x_{in}\\
reserveIn = x_0\\
reserveOut = y_0\\
amountOut = y_{out}
$$

컨트랙트 코드에 표시된 계산 결과는 위 공식에 해당합니다. 마지막 `add(1)`은 반올림 오류로 인해 이론적 최솟값보다 작지 않도록 입력 양(`amountIn`)을 보장하여, 사실상 최소 1단위의 입력 토큰을 더 요구하게 하는 것입니다.

#### getAmountsOut

이 메서드는 주어진 양(`amountIn`)의 첫 번째 토큰에 대해 다중 거래 쌍을 사용하여 시퀀스의 마지막 토큰(`amounts`)을 얼마나 얻을 수 있는지 계산합니다. `amounts` 배열의 첫 번째 요소는 `amountIn`을 나타내고, 마지막 요소는 목표 토큰의 수량을 나타냅니다. 이 메서드는 기본적으로 `getAmountOut` 메서드를 반복합니다.

```solidity
// performs chained getAmountOut calculations on any number of pairs
function getAmountsOut(address factory, uint amountIn, address[] memory path) internal view returns (uint[] memory amounts) {
    require(path.length >= 2, 'UniswapV2Library: INVALID_PATH');
    amounts = new uint[](path.length);
    amounts[0] = amountIn;
    for (uint i; i < path.length - 1; i++) {
        (uint reserveIn, uint reserveOut) = getReserves(factory, path[i], path[i + 1]);
        amounts[i + 1] = getAmountOut(amounts[i], reserveIn, reserveOut);
    }
}
```

#### getAmountsIn

`getAmountsOut`과 반대로, `getAmountsIn`은 목표 토큰의 특정 양(`amountOut`)이 필요할 때 필요한 중간 토큰들의 양을 계산합니다. 이는 `getAmountIn` 메서드를 반복적으로 호출합니다.

```solidity
// performs chained getAmountIn calculations on any number of pairs
function getAmountsIn(address factory, uint amountOut, address[] memory path) internal view returns (uint[] memory amounts) {
    require(path.length >= 2, 'UniswapV2Library: INVALID_PATH');
    amounts = new uint[](path.length);
    amounts[amounts.length - 1] = amountOut;
    for (uint i = path.length - 1; i > 0; i--) {
        (uint reserveIn, uint reserveOut) = getReserves(factory, path[i - 1], path[i]);
        amounts[i - 1] = getAmountIn(amounts[i], reserveIn, reserveOut);
    }
}
```

### ERC20-ERC20

#### addLiquidity 유동성 추가 (Adding Liquidity)

```solidity
function addLiquidity(
    address tokenA,
    address tokenB,
    uint amountADesired,
    uint amountBDesired,
    uint amountAMin,
    uint amountBMin,
    address to,
    uint deadline
) external virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
    (amountA, amountB) = _addLiquidity(tokenA, tokenB, amountADesired, amountBDesired, amountAMin, amountBMin);
    address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
    TransferHelper.safeTransferFrom(tokenA, msg.sender, pair, amountA);
    TransferHelper.safeTransferFrom(tokenB, msg.sender, pair, amountB);
    liquidity = IUniswapV2Pair(pair).mint(to);
}
```

Router02는 사용자와 직접 상호작용하므로 인터페이스는 사용자 시나리오를 염두에 두고 설계되었습니다. `addLiquidity`는 8개의 파라미터를 제공합니다:

* `address tokenA`: 토큰 A
* `address tokenB`: 토큰 B
* `uint amountADesired`: 예치하고자 하는 토큰 A의 양
* `uint amountBDesired`: 예치하고자 하는 토큰 B의 양
* `uint amountAMin`: 예치할 토큰 A의 최소 양
* `uint amountBMin`: 예치할 토큰 B의 최소 양
* `address to`: 유동성 토큰을 받을 주소
* `uint deadline`: 요청 만료 시간

사용자가 제출한 트랜잭션은 불확실한 시간에 채굴자에 의해 패킹될 수 있으므로, 제출 시점의 토큰 가격은 트랜잭션 패킹 시점의 가격과 다를 수 있습니다. `amountMin`은 채굴자나 봇에 의한 악용을 방지하기 위해 가격 변동 범위를 제어하며, 마찬가지로 `deadline`은 지정된 시간 후에 트랜잭션이 만료되도록 보장합니다.

유동성 추가 시 사용자가 제공한 토큰 가격이 실제 가격과 다르면, 그들은 더 낮은 환율로만 유동성 토큰을 받게 되며, 잉여 토큰은 전체 풀에 기여하게 됩니다. `_addLiquidity`는 최적의 환율을 계산하는 데 도움을 줍니다. 유동성을 처음 추가하는 경우 거래 쌍 컨트랙트가 먼저 생성되고, 그렇지 않음 현재 풀 잔액을 기준으로 주입할 최적의 토큰 양이 계산됩니다.

```solidity
// **** ADD LIQUIDITY ****
function _addLiquidity(
    address tokenA,
    address tokenB,
    uint amountADesired,
    uint amountBDesired,
    uint amountAMin,
    uint amountBMin
) internal virtual returns (uint amountA, uint amountB) {
    // create the pair if it doesn't exist yet
    if (IUniswapV2Factory(factory).getPair(tokenA, tokenB) == address(0)) {
        IUniswapV2Factory(factory).createPair(tokenA, tokenB);
    }
    (uint reserveA, uint reserveB) = UniswapV2Library.getReserves(factory, tokenA, tokenB);
    if (reserveA == 0 and reserveB == 0) {
        (amountA, amountB) = (amountADesired, amountBDesired);
    } else {
        uint amountBOptimal = UniswapV2Library.quote(amountADesired, reserveA, reserveB);
        if (amountBOptimal <= amountBDesired) {
            require(amountBOptimal >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
            (amountA, amountB) = (amountADesired, amountBOptimal);
        } else {
            uint amountAOptimal = UniswapV2Library.quote(amountBDesired, reserveB, reserveA);
            assert(amountAOptimal <= amountADesired);
            require(amountAOptimal >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
            (amountA, amountB) = (amountAOptimal, amountBDesired);
        }
    }
}
```

마지막으로 핵심 컨트랙트의 `mint` 메서드가 호출되어 유동성 토큰을 발행합니다.

#### removeLiquidity 유동성 제거 (Removing Liquidity)

먼저 유동성 토큰이 페어 컨트랙트로 전송됩니다. 수신된 유동성 토큰 대 전체 토큰의 비율을 기준으로, 유동성이 나타내는 두 토큰의 해당 양이 계산됩니다. 유동성 토큰을 소각한 후 사용자는 해당 비율의 토큰을 받습니다. 만약 이것이 사용자가 설정한 최소 기대치(amountAMin/amountBMin)보다 낮으면 트랜잭션이 revert됩니다.

```solidity
// **** REMOVE LIQUIDITY ****
function removeLiquidity(
    address tokenA,
    address tokenB,
    uint liquidity,
    uint amountAMin,
    uint amountBMin,
    address to,
    uint deadline
) public virtual override ensure(deadline) returns (uint amountA, uint amountB) {
    address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
    IUniswapV2Pair(pair).transferFrom(msg.sender, pair, liquidity); // send liquidity to pair
    (uint amount0, uint amount1) = IUniswapV2Pair(pair).burn(to);
    (address token0,) = UniswapV2Library.sortTokens(tokenA, tokenB);
    (amountA, amountB) = tokenA == token0 ? (amount0, amount1) : (amount1, amount0);
    require(amountA >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
    require(amountB >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
}
```

#### removeLiquidityWithPermit 서명으로 유동성 제거 (Removing Liquidity with Signature)

일반적으로 유동성을 제거하려면 사용자는 두 가지 작업을 수행해야 합니다:

* `approve`: Router 컨트랙트가 유동성 토큰을 사용할 수 있도록 승인
* `removeLiquidity`: Router 컨트랙트를 호출하여 유동성 제거

첫 번째 승인 시 최대 토큰 양을 승인하지 않았다면, 각 유동성 제거마다 두 번의 상호작용이 필요하며, 이는 사용자가 가스비를 두 번 지불해야 함을 의미합니다. `removeLiquidityWithPermit` 메서드를 사용하면 사용자는 `approve`를 별도로 호출하지 않고도 서명을 통해 Router 컨트랙트가 토큰을 사용할 수 있도록 승인할 수 있으며, 유동성 제거 메서드를 한 번만 호출하여 작업을 완료할 수 있어 가스 비용을 절약할 수 있습니다. 또한 오프체인 서명은 가스비가 발생하지 않으므로, 각 서명은 특정 양의 토큰만 승인할 수 있어 보안을 강화합니다.

```solidity
function removeLiquidityWithPermit(
    address tokenA,
    address tokenB,
    uint liquidity,
    uint amountAMin,
    uint amountBMin,
    address to,
    uint deadline,
    bool approveMax, uint8 v, bytes32 r, bytes32 s
) external virtual override returns (uint amountA, uint amountB) {
    address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
    uint value = approveMax ? uint(-1) : liquidity;
    IUniswapV2Pair(pair).permit(msg.sender, address(this), value, deadline, v, r, s);
    (amountA, amountB) = removeLiquidity(tokenA, tokenB, liquidity, amountAMin, amountBMin, to, deadline);
}
```

#### swapExactTokensForTokens

거래에는 두 가지 일반적인 시나리오가 있습니다:

1. 지정된 양의 토큰 A(입력)를 사용하여 최대 양의 토큰 B(출력)로 교환
1. 지정된 양의 토큰 B(출력)를 받기 위해 최소 양의 토큰 A(입력) 사용

이 메서드는 첫 번째 시나리오를 구현하여, 지정된 입력 토큰을 기준으로 최대 출력 토큰으로 교환합니다.

```solidity
function swapExactTokensForTokens(
    uint amountIn,
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline
) external virtual override ensure(deadline) returns (uint[] memory amounts) {
    amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);
    require(amounts[amounts.length - 1] >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
    TransferHelper.safeTransferFrom(
        path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
    );
    _swap(amounts, path, to);
}
```

먼저 Library 컨트랙트의 `getAmountsOut` 메서드를 사용하여 교환 경로(path)에 따라 각 거래의 출력 토큰 양을 계산하고, 마지막 거래에서 받은 양(`amounts[amounts.length - 1]`)이 예상되는 최소 출력(`amountOutMin`)보다 작지 않음을 보장합니다. 그런 다음 토큰은 전체 교환 프로세스를 시작하기 위해 첫 번째 거래 쌍 주소로 전송됩니다.

사용자가 WETH를 DYDX로 스왑하려고 하고, 오프체인에서 계산된 최상의 교환 경로가 WETH → USDC → DYDX라고 가정하면, `amountIn`은 WETH 양, `amountOutMin`은 예상되는 최소 DYDX 양, `path`는 [WETH 주소, USDC 주소, DYDX 주소], `amounts`는 [amountIn에 해당하는 WETH 양, USDC 양, DYDX 양]입니다. `_swap` 실행 중에 각 중간 거래에서 받은 토큰은 다음 거래 쌍 주소로 전송되며, 최종 거래가 완료되고 `_to` 주소가 마지막 거래의 출력 토큰을 받을 때까지 계속됩니다.

```solidity
// requires the initial amount to have already been sent to the first pair
function _swap(uint[] memory amounts, address[] memory path, address _to) internal virtual {
    for (uint i; i < path.length - 1; i++) {
        (address input, address output) = (path[i], path[i + 1]);
        (address token0,) = UniswapV2Library.sortTokens(input, output);
        uint amountOut = amounts[i + 1];
        (uint amount0Out, uint amount1Out) = input == token0 ? (uint(0), amountOut) : (amountOut, uint(0));
        address to = i < path.length - 2 ? UniswapV2Library.pairFor(factory, output, path[i + 2]) : _to;
        IUniswapV2Pair(UniswapV2Library.pairFor(factory, input, output)).swap(
            amount0Out, amount1Out, to, new bytes(0)
        );
    }
}
```

#### swapTokensForExactTokens

이 메서드는 두 번째 거래 시나리오를 구현하여, 지정된 출력 토큰을 얻기 위해 최소 입력 토큰을 교환합니다.

```solidity
function swapTokensForExactTokens(
    uint amountOut,
    uint amountInMax,
    address[] calldata path,
    address to,
    uint deadline
) external virtual override ensure(deadline) returns (uint[] memory amounts) {
    amounts = UniswapV2Library.getAmountsIn(factory, amountOut, path);
    require(amounts[0] <= amountInMax, 'UniswapV2Router: EXCESSIVE_INPUT_AMOUNT');
    TransferHelper.safeTransferFrom(
        path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
    );
    _swap(amounts, path, to);
}
```

위와 유사하게, 여기서는 Library의 `getAmountsIn` 메서드를 먼저 사용하여 각 교환에 필요한 최소 입력 토큰 양을 역으로 계산하고, 첫 번째 토큰에 대해 계산된 양(수수료 차감 후)이 사용자가 제공할 의사가 있는 최대 양(`amountInMax`)보다 크지 않음을 보장합니다. 그런 다음 토큰은 전체 교환 프로세스를 시작하기 위해 첫 번째 거래 쌍 주소로 전송됩니다.

### ERC20-ETH

#### ETH 지원 (ETH Support)

핵심 컨트랙트는 ERC20 토큰 거래만 지원하므로, 주변 컨트랙트는 ETH 거래를 지원하기 위해 ETH와 WETH 간의 변환을 수행해야 합니다. 이를 수용하기 위해 대부분의 메서드는 ETH 버전으로 제공됩니다. 교환은 주로 두 가지 작업을 포함합니다:

* 주소 변환: ETH에는 컨트랙트 주소가 없으므로, ETH와 WETH 간의 변환을 위해 WETH 컨트랙트의 입금(deposit) 및 출금(withdraw) 메서드가 사용됩니다.
* 토큰 양 변환: ETH 양은 `msg.value`를 통해 얻어지며, 해당 WETH 양을 계산한 후 표준 ERC20 인터페이스를 사용할 수 있습니다.

#### FeeOnTransferTokens

일부 토큰은 전송 과정에서 수수료를 차감하여 전송된 양과 실제 수신된 양 사이에 차이가 발생합니다. 따라서 중간 교환에 필요한 토큰 양을 계산하는 것이 간단하지 않습니다. 이런 경우 실제 수령된 토큰 수량(actual received token amounts)을 확인하기 위해 `transfer` 메서드 대신 `balanceOf` 메서드를 사용해야 합니다. Router02는 수수료 차감 전송 토큰(Inclusive Fee On Transfer Tokens)에 대한 지원을 도입했습니다. 더 자세한 설명은 [공식 문서](https://docs.uniswap.org/protocol/V2/reference/smart-contracts/common-errors#inclusive-fee-on-transfer-tokens)를 참조하세요.
