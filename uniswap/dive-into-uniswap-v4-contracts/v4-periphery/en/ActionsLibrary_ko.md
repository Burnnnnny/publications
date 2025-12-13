# Actions 라이브러리 (Actions Library)

Actions 라이브러리는 Uniswap v4 periphery(주변) 컨트랙트에서 지원하는 모든 작업을 정의합니다.

```solidity
// pool actions
// liquidity actions
uint256 internal constant INCREASE_LIQUIDITY = 0x00;
uint256 internal constant DECREASE_LIQUIDITY = 0x01;
uint256 internal constant MINT_POSITION = 0x02;
uint256 internal constant BURN_POSITION = 0x03;
uint256 internal constant INCREASE_LIQUIDITY_FROM_DELTAS = 0x04;
uint256 internal constant MINT_POSITION_FROM_DELTAS = 0x05;

// swapping
uint256 internal constant SWAP_EXACT_IN_SINGLE = 0x06;
uint256 internal constant SWAP_EXACT_IN = 0x07;
uint256 internal constant SWAP_EXACT_OUT_SINGLE = 0x08;
uint256 internal constant SWAP_EXACT_OUT = 0x09;

// donate
// note this is not supported in the position manager or router
uint256 internal constant DONATE = 0x0a;

// closing deltas on the pool manager
// settling
uint256 internal constant SETTLE = 0x0b;
uint256 internal constant SETTLE_ALL = 0x0c;
uint256 internal constant SETTLE_PAIR = 0x0d;
// taking
uint256 internal constant TAKE = 0x0e;
uint256 internal constant TAKE_ALL = 0x0f;
uint256 internal constant TAKE_PORTION = 0x10;
uint256 internal constant TAKE_PAIR = 0x11;

uint256 internal constant CLOSE_CURRENCY = 0x12;
uint256 internal constant CLEAR_OR_TAKE = 0x13;
uint256 internal constant SWEEP = 0x14;

uint256 internal constant WRAP = 0x15;
uint256 internal constant UNWRAP = 0x16;

// minting/burning 6909s to close deltas
// note this is not supported in the position manager or router
uint256 internal constant MINT_6909 = 0x17;
uint256 internal constant BURN_6909 = 0x18;
```

- 유동성 관련 작업:
    - INCREASE_LIQUIDITY: 유동성 증가
    - DECREASE_LIQUIDITY: 유동성 감소
    - MINT_POSITION: 포지션 발행 (Mint)
    - BURN_POSITION: 포지션 소각 (Burn)
    - INCREASE_LIQUIDITY_FROM_DELTAS: 잔액(Balance)을 기반으로 유동성 증가
    - MINT_POSITION_FROM_DELTAS: 잔액을 기반으로 포지션 발행

- 스왑(교환) 관련 작업:
    - SWAP_EXACT_IN_SINGLE: 정확한 입력 토큰 금액, 출력 토큰 금액 계산, 단일 홉 거래 완료
    - SWAP_EXACT_IN: 정확한 입력 토큰 금액, 거래 경로에 따라 출력 토큰 금액 계산, 다중 홉 거래 완료
    - SWAP_EXACT_OUT_SINGLE: 정확한 출력 토큰 금액, 입력 토큰 금액 계산, 단일 홉 거래 완료
    - SWAP_EXACT_OUT: 정확한 출력 토큰 금액, 거래 경로에 따라 입력 토큰 금액 계산, 다중 홉 거래 완료

- 기부 작업:
    - DONATE: 토큰 기부

- 정산 관련 작업:
    - SETTLE: 단일 토큰 부채 정산
    - SETTLE_ALL: 단일 토큰의 모든 부채 정산
    - SETTLE_PAIR: 토큰 쌍 부채 정산
    - TAKE: 단일 토큰 잔액 인출
    - TAKE_ALL: 단일 토큰의 모든 잔액 인출
    - TAKE_PORTION: 단일 토큰 잔액의 일부 인출
    - TAKE_PAIR: 토큰 쌍 잔액 인출
    - CLOSE_CURRENCY: 단일 토큰 정산 또는 인출
    - CLEAR_OR_TAKE: 단일 토큰 포기 또는 인출
    - SWEEP: 토큰 전송 (빼내기)

- 래핑(Wrapping) 작업:
    - WRAP: ETH 래핑 (WETH로 변환)
    - UNWRAP: WETH 언래핑 (ETH로 변환)

- ERC6909 토큰 발행/소각 작업:
    - MINT_6909: ERC6909 발행
    - BURN_6909: ERC6909 소각
