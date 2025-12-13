[English](./README.md) | [中文](./README_zh.md) | [한국어](./README_ko.md)

# Uniswap v4 스마트 컨트랙트 심층 분석

###### 태그: `uniswap` `uniswap-v4` `smart contract` `solidity`

# 워크플로우 (Workflow)

다음 다이어그램은 Uniswap v4 컨트랙트의 워크플로우를 보여줍니다:

![](./assets/uniswap-v4-workflow.png)

Uniswap v2/v3와 유사하게, Uniswap v4 컨트랙트도 두 개의 저장소로 나뉩니다:

* [v4-core](./v4-core/en/README_ko.md) [[Github](https://github.com/Uniswap/v4-core)]: Uniswap v4의 핵심 컨트랙트를 포함하며, 주로 다음을 포함합니다:
    * [PoolManager.sol](./v4-core/en/PoolManager_ko.md): 모든 Uniswap v4 풀을 관리하는 싱글톤 컨트랙트로, 생성, 유동성 수정, 스왑 등을 포함한 풀의 모든 외부 인터페이스를 제공합니다.
    * 라이브러리 컨트랙트:
        * [Pool.sol](./v4-core/en/PoolLibrary_ko.md): 풀 라이브러리 컨트랙트, 유동성 수정, 스왑 등 각 풀에 대한 특정 작업을 실행하는 데 사용됩니다.
        * [Position.sol](./v4-core/en/PositionLibrary_ko.md): 포지션 라이브러리 컨트랙트, 유동성 및 수수료 수정과 같은 각 포지션과 관련된 특정 작업을 실행하는 데 사용됩니다.
        * [Hooks.sol](./v4-core/en/HooksLibrary_ko.md): 훅 라이브러리 컨트랙트, Uniswap v4 컨트랙트의 훅 기능을 실행하는 데 사용됩니다.
        * [CurrencyDelta.sol](./v4-core/en/CurrencyDeltaLibrary_ko.md): CurrencyDelta 라이브러리 컨트랙트, 플래시 어카운팅(Flash Accounting)과 관련된 작업을 실행하는 데 사용됩니다.
        * [BalanceDelta.sol](./v4-core/en/BalanceDelta_ko.md)：BalanceDelta는 잔액 델타 유형을 정의합니다.

* [v4-periphery](./v4-periphery/en/README_ko.md) [[Github](https://github.com/Uniswap/v4-periphery/)]: Uniswap v4의 주변 컨트랙트를 포함하며, 주로 다음을 포함합니다:
    * [PositionManager.sol](./v4-periphery/en/PositionManager_ko.md): PositionManager 컨트랙트, 포지션의 발행, 소각 및 유동성 수정을 관리하는 데 사용되며, [PoolManager](./v4-core/en/PoolManager_ko.md)를 호출하여 특정 작업을 실행합니다.
        * 외부 컨트랙트는 v4-core PoolManager 컨트랙트를 직접 호출하는 대신 PositionManager 컨트랙트를 통해 포지션을 운영합니다.
        * 여러 작업을 단일 트랜잭션으로 결합하는 것을 지원하여 트랜잭션의 원자성을 보장하고 가스 소비를 줄입니다.
    * [V4Router.sol](./v4-periphery/en/V4Router_ko.md): V4Router 컨트랙트, 거래 작업을 실행하는 데 사용되며, [PoolManager](./v4-core/en/PoolManager_ko.md)를 호출하여 특정 거래 작업을 실행합니다.
        * 단일 홉(single-hop) 및 멀티 홉(multi-hop) 거래를 지원합니다.
        * 정확한 입력 또는 출력 토큰 수량 지정을 지원합니다.
