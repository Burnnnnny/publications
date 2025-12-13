# v4-core

v4-core [[Github](https://github.com/Uniswap/v4-core)]: Uniswap v4 핵심 컨트랙트를 포함하며, 주로 다음을 포함합니다:

* [PoolManager.sol](./PoolManager_ko.md): 모든 Uniswap v4 풀을 관리하는 싱글톤 컨트랙트로, 생성, 유동성 수정, 거래 및 기타 작업을 포함한 풀의 모든 외부 인터페이스를 제공합니다.
* 라이브러리 컨트랙트:
    * [Pool.sol](./PoolLibrary_ko.md): 풀 라이브러리 컨트랙트, 유동성 수정, 거래 등 각 풀에 대한 작업을 구체적으로 실행하는 데 사용됩니다.
    * [Position.sol](./PositionLibrary_ko.md): 포지션 라이브러리 컨트랙트, 유동성 및 수수료 업데이트와 같은 각 포지션과 관련된 작업을 구체적으로 실행하는 데 사용됩니다.
    * [Hooks.sol](./HooksLibrary_ko.md): 훅 라이브러리 컨트랙트, Uniswap v4 컨트랙트의 훅 기능을 실행하는 데 사용됩니다.
    * [CurrencyDelta.sol](./CurrencyDeltaLibrary_ko.md): CurrencyDelta 라이브러리 컨트랙트, 플래시 어카운팅(Flash Accounting)과 관련된 작업을 실행하는 데 사용됩니다.
    * [BalanceDelta.sol](./BalanceDelta_ko.md): BalanceDelta는 회계를 위한 잔액 델타 유형을 정의합니다.
