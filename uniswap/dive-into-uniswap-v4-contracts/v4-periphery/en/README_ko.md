# v4-periphery

v4-periphery [[Github](https://github.com/Uniswap/v4-periphery/)]: Uniswap v4 periphery(주변) 컨트랙트를 포함하며, 주로 다음을 포함합니다:

* [PositionManager.sol](./PositionManager_ko.md): PositionManager 컨트랙트, 포지션 생성, 소각 및 유동성 수정과 같은 작업을 관리하고 [PoolManager](../../v4-core/en/PoolManager_ko.md)를 호출하여 특정 작업을 실행하는 데 사용됩니다.
    * 외부 컨트랙트는 v4-core PoolManager 컨트랙트를 직접 호출하는 대신 PositionManager 컨트랙트를 통해 포지션을 운영합니다.
    * 여러 작업을 하나의 트랜잭션으로 결합하여 원자성을 보장하고 가스 소비를 줄이는 것을 지원합니다.

* [V4Router.sol](./V4Router_ko.md): V4Router 컨트랙트, 거래 작업을 실행하고 [PoolManager](../../v4-core/en/PoolManager_ko.md) 컨트랙트를 호출하여 특정 거래 작업을 실행하는 데 사용됩니다.
    * 단일 홉 및 다중 홉 거래를 지원합니다.
    * 입력 또는 출력 토큰 금액 지정을 지원합니다.
