[English](./README.md) | [中文](./README_zh.md)

# 조건부 토큰 (Conditional Tokens) 심층 분석

[조건부 토큰(Conditional Tokens)](https://conditional-tokens.readthedocs.io/en/latest/index.html)은 Gnosis 팀이 [ERC-1155](https://ethereum.org/developers/docs/standards/tokens/erc-1155/) 표준을 기반으로 개발한 스마트 컨트랙트 프레임워크입니다. 핵심 목적은 복잡한 "조합 예측 시장(Combinatorial Prediction Markets)"을 지원하고 중첩된 조건을 다룰 때 전통적인 예측 시장이 직면하는 유동성 파편화 문제를 해결하는 것입니다.

이 글에서는 이 시스템의 설계 철학과 작동 메커니즘을 자세히 설명합니다.

## 조건부 토큰의 설계 원칙

### 조합 예측 시장의 "경로 의존성" 문제

예측 시장에서 우리는 단일 사건(예: "누가 선거에서 이길까?")뿐만 아니라 연쇄 사건(예: "A가 선거에서 이기면 주식 시장이 오를까 내릴까?")에도 관심을 가집니다. 이를 조합 예측 시장(Combinatorial Prediction Markets)이라고 합니다.

두 가지 독립적인 예측 질문이 있다고 가정해 봅시다:

  * 질문 A (선거): 결과가 앨리스(Alice)인가 밥(Bob)인가?
  * 질문 B (경제): 결과가 호황(High)인가 불황(Low)인가?

우리는 "앨리스가 당선되고 **그리고(AND)** 경제가 호황인" 시나리오를 예측하고 싶습니다.

**전통적인 방법의 한계**

과거의 컨트랙트 설계에서는 이 조합을 달성하기 위해 토큰을 중첩해야 했습니다. 하지만 이는 **경로 의존성(Path Dependence)**이라는 심각한 문제를 야기합니다.

* **경로 1:** 먼저 자금을 담보로 "앨리스"를 나타내는 토큰을 생성한 다음, "앨리스 토큰"을 담보로 사용하여 "앨리스 & 호황" 토큰을 생성합니다.

![](./assets/path_1.png)

* **경로 2:** 먼저 자금을 담보로 "호황"을 나타내는 토큰을 생성한 다음, "호황 토큰"을 담보로 사용하여 "호황 & 앨리스" 토큰을 생성합니다.

![](./assets/path_2.png)

"앨리스 & 호황"과 "호황 & 앨리스"는 논리적으로 완전히 동일한 현실 세계의 결과를 나타내지만, 블록체인상에서는 완전히 다른 두 개의 토큰입니다.

![](./assets/path_3.png)

이는 다음과 같은 문제로 이어집니다:

  * **유동성 파편화:** 같은 결과에 베팅하는 사람들이 서로 다른 순서로 생성되었다는 이유만으로 호환되지 않는 토큰을 보유하게 되어, 같은 풀에서 거래할 수 없습니다.
  * **열악한 사용자 경험:** 사용자는 특정 순서로 작업하지 않으면 포지션을 합칠 수 없습니다.

### 설계 목표: 경로 독립성 (대체 가능성)

위의 문제를 해결하기 위해 조건부 토큰의 설계 목표는 명확합니다: **경로 독립성(Path Independence)**, 즉 **대체 가능성(Fungibility)**을 달성하는 것입니다.
시스템은 사용자가 A 다음에 B를 선택하든 B 다음에 A를 선택하든, 최종 조건 집합이 같다면 생성된 토큰 ID가 정확히 동일함을 보장해야 합니다.

수학적으로 표현하면 **교환 법칙(Commutativity)**을 만족해야 합니다:

$$Token(A + B) = Token(B + A)$$

### 왜 단순 XOR / 덧셈을 사용하지 않는가?

교환 법칙을 달성하기 위해 토큰 ID를 생성하는 수학적 연산을 선택해야 합니다.

**비트 XOR 또는 일반 덧셈**

가장 확실한 해결책은 XOR이나 모듈러 덧셈을 사용하는 것입니다.
A의 특징 값을 Hash(A), B의 특징 값을 Hash(B)라고 가정합니다.
연산: $ID = Hash(A) \oplus Hash(B)$.
XOR은 교환 법칙($A \oplus B = B \oplus A$)을 만족하므로 완벽해 보이며 매우 빠릅니다.

그러나 단순 알고리즘(XOR, 덧셈)은 선형적이어서 암호학의 **일반화된 생일 공격(Generalized Birthday Attacks)**(Wagner의 알고리즘)에 취약합니다.

**공격 논리:**

1.  공격자는 가치가 높은 ID(조건 A와 B로 생성됨)를 위조하고 싶어 합니다.
2.  공격자는 매우 적은 비용으로 대량의 "정크 조건"(C, D, E, F...)을 생성할 수 있습니다.
3.  연산이 선형적이므로 공격자는 알고리즘을 사용하여 정크 조건 중 조합(예: C+D+E)을 효율적으로 찾아내어 그 XOR 결과가 A+B의 결과와 정확히 같게 만들 수 있습니다. 이것이 해시 충돌입니다. 그런 다음 공격자는 정크 조건의 토큰을 사용하여 컨트랙트를 속이고 A+B 조건에 속한 실제 담보를 인출합니다.

### 최종 해결책: 타원 곡선 (Elliptic Curve)

"교환 법칙"과 "충돌 저항성"을 모두 만족시키기 위해 Gnosis는 **타원 곡선 암호(ECC)**의 점 덧셈을 선택했습니다.

**왜 타원 곡선인가?**

**교환 법칙 만족:**

타원 곡선에서 두 점 $P_1$과 $P_2$를 더하는 것은 $P_1 + P_2 = P_2 + P_1$이 되도록 기하학적으로 정의됩니다. 이는 조건이 결합되는 순서와 관계없이 최종 계산된 점(및 해당 ID)이 고유함을 의미합니다. 이것이 유동성 파편화 문제를 해결합니다. 따라서 사용자가 어떤 순서로 조건을 결합하든 생성된 토큰 ID는 동일합니다.

**비선형성과 보안:**

타원 곡선 점 덧셈은 복잡한 기하학적 접선 연산을 포함하며, 이는 고도로 비선형적인 연산입니다.
이는 단순한 선형 관계를 파괴하여 앞서 언급한 "일반화된 생일 공격"을 극도로 어렵고 실질적으로 불가능하게 만듭니다.
해커는 단순한 계산으로 두 개의 정크 조건을 짜맞추어 실제 조건의 ID와 충돌시킬 수 없습니다. 이것이 충돌 저항성을 보장합니다.

**ID 생성 흐름**

코드 수준에서 토큰 ID 생성 로직은 다음과 같습니다:

1.  각 기본 조건을 타원 곡선 위의 한 점으로 매핑합니다.
2.  조건이 결합될 때 타원 곡선 점 덧셈을 수행합니다.
3.  결과 점을 인코딩하여 컬렉션 ID(Collection ID)를 생성합니다.

## 조건부 토큰의 핵심 개념

### 조건 (Condition)

조건부 토큰 프레임워크에서 **조건(Condition)**은 예측 시장의 핵심 단위입니다. 조건은 미리 설정된 여러 대체 결과가 있는 질문(Question)이며, 일반적으로 미래의 특정 시점에 오라클에 의해 특정 결과(Outcome)로 해결됩니다.

### 결과 컬렉션 (Outcome Collection)

**결과 컬렉션(Outcome Collection)**은 특정 조건 이벤트 하의 모든 가능한 결과의 조합을 나타냅니다. 공집합이나 전체 집합은 포함하지 않습니다. 공집합은 실질적인 의미가 없으며, 전체 집합은 모든 결과가 발생함을 의미하여 예측 가치가 없기 때문입니다.

예를 들어, 세 가지 옵션(A, B, C)이 있는 조건 이벤트의 경우 결과 컬렉션은 다음을 포함합니다:

  * {A}: 결과가 A임;
  * {B}: 결과가 B임;
  * {C}: 결과가 C임;
  * {A, B}: 결과가 A 또는 B임;
  * {A, C}: 결과가 A 또는 C임;
  * {B, C}: 결과가 B 또는 C임.

#### 인덱스 세트 (Index Set)

`Index Set`을 사용하여 결과 컬렉션을 나타낼 수 있으며, 각 결과는 이진 비트에 해당합니다. 1은 결과가 포함됨을, 0은 포함되지 않음을 의미합니다. 하위 비트부터 상위 비트 순으로 각각 결과 A, B, C에 대응합니다. 예를 들어:

  * {A}는 인덱스 세트 0b001 = 1에 해당
  * {B}는 인덱스 세트 0b010 = 2에 해당
  * {A, B}는 인덱스 세트 0b011 = 3에 해당

#### 컬렉션 ID (Collection ID)

컬렉션 ID는 결과 컬렉션의 고유 ID를 나타내며 32바이트를 사용합니다.

조건 ID(Condition ID)와 인덱스 세트를 해싱한 후, 그 결과를 타원 곡선 위의 한 점으로 매핑하여 해당 조건 이벤트의 선택된 결과 집합에 대한 컬렉션 ID로 만듭니다.

서로 다른 조건 이벤트의 결과 컬렉션에 대해, 개별 조건 이벤트 결과 컬렉션의 컬렉션 ID를 별도로 계산할 수 있습니다. 그런 다음 타원 곡선 점 덧셈을 통해 **조합된** 조건 이벤트 결과 컬렉션의 컬렉션 ID를 계산합니다.

1.  **매핑 (Map):** 조건 ID + 인덱스 세트를 해싱하여 타원 곡선 위의 점 $P_{new}$로 결정론적으로 매핑합니다.
2.  **디코딩 (Decode):** `parentCollectionId`가 존재하면 `bytes32`에서 타원 곡선 위의 점 $P_{parent}$로 다시 디코딩합니다.
3.  **덧셈 (Add):** 타원 곡선 덧셈을 수행합니다: $P_{result} = P_{parent} + P_{new}$. 타원 곡선 덧셈은 교환 법칙을 만족하므로 조건 이벤트의 결합 순서와 관계없이 최종 $P_{result}$는 동일합니다.
4.  **인코딩 (Encode):** 최종 점 $P_{result}$를 무손실 압축(x 좌표 + y 좌표의 패리티 비트를 32바이트로 압축)하여 새로운 컬렉션 ID로 직접 반환합니다.

### 포지션 (Position)

조건부 토큰 프레임워크에서 사용자는 ERC-1155 표준 조건부 토큰을 보유함으로써 특정 포지션에 대한 지분을 나타내며, 여기서 포지션 ID는 ERC-1155 토큰의 토큰 ID 역할을 합니다.

**포지션(Position)**은 다음 두 가지 요소로 구성됩니다:

  * **담보 토큰 (Collateral Token):** 사용자가 시장에 진입하기 위해 사용하는 ERC-20 기본 토큰(예: USDC).
  * **결과 컬렉션 (Outcome Collection):** 사용자가 선택한 예측 결과.
      * 조합 조건 이벤트 결과 컬렉션이 필요한 경우 컬렉션 ID는 [컬렉션 ID](#컬렉션-id-collection-id)에 설명된 대로 계산됩니다.

**포지션 ID**는 담보 토큰 주소와 결과 컬렉션의 컬렉션 ID를 해싱하여 얻을 수 있습니다.

## 스마트 컨트랙트 구현 로직

조건부 토큰의 스마트 컨트랙트는 주로 `ConditionalTokens.sol`에 의해 구현됩니다. 이는 다음 핵심 작업을 통해 예측 시장의 수명 주기를 관리합니다:

  * 조건 준비 (Prepare Condition)
  * 조건 해결 (Resolve Condition)
  * 포지션 분할 (Split Position)
  * 포지션 병합 (Merge Positions)
  * 포지션 상환 (Redeem Positions)

### 조건 준비 (Preparing a Condition)

각 조건 이벤트는 다음 매개변수로 정의됩니다:

  * `oracle`: 조건의 결과를 해결할 책임이 있는 주소.
  * `questionId`: 질문을 고유하게 식별하는 해시값. 호출자가 설정하며 IPFS 해시 또는 기타 고유 식별자가 될 수 있습니다.
  * `outcomeSlotCount`: 조건에 대한 가능한 결과의 수(예: 3지선다의 경우 3), 256을 초과할 수 없음.

<!-- end list -->

```solidity
// @dev This function prepares a condition by initializing a payout vector associated with the condition.
/// @param oracle The account assigned to report the result for the prepared condition.
/// @param questionId An identifier for the question to be answered by the oracle.
/// @param outcomeSlotCount The number of outcome slots which should be used for this condition. Must not exceed 256.
function prepareCondition(address oracle, bytes32 questionId, uint outcomeSlotCount) external {
    // Limit of 256 because we use a partition array that is a number of 256 bits.
    require(outcomeSlotCount <= 256, "too many outcome slots");
    require(outcomeSlotCount > 1, "there should be more than one outcome slot");
    bytes32 conditionId = CTHelpers.getConditionId(oracle, questionId, outcomeSlotCount);
    require(payoutNumerators[conditionId].length == 0, "condition already prepared");
    payoutNumerators[conditionId] = new uint[](outcomeSlotCount);
    emit ConditionPreparation(conditionId, oracle, questionId, outcomeSlotCount);
}
```

`prepareCondition` 메서드는 "오라클 주소" + "질문 ID" + "결과 수"의 해시를 사용하여 고유한 조건 이벤트 ID인 `conditionId`를 생성합니다.

`payoutNumerators`는 각 `conditionId`에 대한 결과 배당금 비율 배열을 저장하는 매핑입니다. 빈 배열로 시작합니다. 예를 들어, 3가지 결과가 있는 조건의 경우 `payoutNumerators`는 결국 [1, 1, 0](처음 두 결과가 각각 50%를 받고 세 번째는 0을 받음을 의미)이 될 수 있습니다. `payoutNumerators[conditionId].length`를 확인하여 조건이 초기화되었는지 확인할 수 있습니다.

### 조건 해결 (Resolving a Condition)

이벤트가 종료된 후 오라클은 `reportPayouts` 메서드를 호출하여 조건의 최종 결과를 게시합니다.

```solidity
/// @dev Called by the oracle for reporting results of conditions. Will set the payout vector for the condition with the ID ``keccak256(abi.encodePacked(oracle, questionId, outcomeSlotCount))``, where oracle is the message sender, questionId is one of the parameters of this function, and outcomeSlotCount is the length of the payouts parameter, which contains the payoutNumerators for each outcome slot of the condition.
/// @param questionId The question ID the oracle is answering for
/// @param payouts The oracle's answer
function reportPayouts(bytes32 questionId, uint[] calldata payouts) external {
    uint outcomeSlotCount = payouts.length;
    require(outcomeSlotCount > 1, "there should be more than one outcome slot");
    // IMPORTANT, the oracle is enforced to be the sender because it's part of the hash.
    bytes32 conditionId = CTHelpers.getConditionId(msg.sender, questionId, outcomeSlotCount);
    require(payoutNumerators[conditionId].length == outcomeSlotCount, "condition not prepared or found");
    require(payoutDenominator[conditionId] == 0, "payout denominator already set");

    uint den = 0;
    for (uint i = 0; i < outcomeSlotCount; i++) {
        uint num = payouts[i];
        den = den.add(num);

        require(payoutNumerators[conditionId][i] == 0, "payout numerator already set");
        payoutNumerators[conditionId][i] = num;
    }
    require(den > 0, "payout is all zeroes");
    payoutDenominator[conditionId] = den;
    emit ConditionResolution(conditionId, msg.sender, questionId, outcomeSlotCount, payoutNumerators[conditionId]);
}
```

`reportPayouts` 메서드는 호출자가 오라클 주소(`msg.sender`)여야 하며, 전달된 `payouts` 배열의 길이가 준비된 조건의 `outcomeSlotCount`와 일치해야 합니다.

[조건 준비](#조건-준비-preparing-a-condition)에서 우리는 조건 이벤트가 `oracle address + questionId + outcomeSlotCount`에 의해 고유하게 결정된다는 것을 알고 있습니다. 따라서 `questionId`를 전달하고 `msg.sender`(오라클 주소) 및 `payouts.length`(outcomeSlotCount)와 결합하여 `conditionId`를 다시 계산할 수 있습니다.

`payoutNumerators` 배열은 오라클이 게시한 결과 배당금 비율(분자)로 업데이트되며, `payoutDenominator`는 모든 배당금 비율의 합(분모)으로 설정되어, 이후 정산 시 각 결과에 대한 실제 배당금 금액을 정확하게 계산할 수 있도록 합니다.

### 포지션 분할 (Split Position)

이것은 시장에 진입하는 작업입니다.

두 가지 시나리오가 있습니다:

1.  사용자가 담보(일반적으로 USDC와 같은 스테이블코인)를 잠가 상호 배타적인 결과 토큰 세트를 발행합니다.
2.  사용자가 결과 토큰 세트를 잠가 더 세분화된 하위 결과 토큰을 발행합니다. 예를 들어, "앨리스 OR 밥" 토큰을 별도의 "앨리스" 및 "밥" 토큰으로 분할합니다.

확률 보존(모든 가능한 결과의 확률 합은 1)에 따라 분할 전후의 포지션 가치는 동일해야 합니다.

1.  시나리오 1의 경우, 담보 1단위는 모든 가능한 결과에 대한 토큰으로 분할됩니다.
2.  시나리오 2의 경우, 결과 컬렉션 토큰 1단위는 하위 컬렉션에 대한 토큰으로 분할됩니다. 하위 컬렉션의 확률 합이 부모 컬렉션의 확률과 같기 때문입니다.

![](./assets/split_1.png)

```solidity
/// @dev This function splits a position. If splitting from the collateral, this contract will attempt to transfer `amount` collateral from the message sender to itself. Otherwise, this contract will burn `amount` stake held by the message sender in the position being split worth of EIP 1155 tokens. Regardless, if successful, `amount` stake will be minted in the split target positions. If any of the transfers, mints, or burns fail, the transaction will revert. The transaction will also revert if the given partition is trivial, invalid, or refers to more slots than the condition is prepared with.
/// @param collateralToken The address of the positions' backing collateral token.
/// @param parentCollectionId The ID of the outcome collections common to the position being split and the split target positions. May be null, in which only the collateral is shared.
/// @param conditionId The ID of the condition to split on.
/// @param partition An array of disjoint index sets representing a nontrivial partition of the outcome slots of the given condition. E.g. A|B and C but not A|B and B|C (is not disjoint). Each element's a number which, together with the condition, represents the outcome collection. E.g. 0b110 is A|B, 0b010 is B, etc.
/// @param amount The amount of collateral or stake to split.
function splitPosition(
    IERC20 collateralToken,
    bytes32 parentCollectionId,
    bytes32 conditionId,
    uint[] calldata partition,
    uint amount
) external {
    require(partition.length > 1, "got empty or singleton partition");
    uint outcomeSlotCount = payoutNumerators[conditionId].length;
    require(outcomeSlotCount > 0, "condition not prepared yet");

    // For a condition with 4 outcomes fullIndexSet's 0b1111; for 5 it's 0b11111...
    uint fullIndexSet = (1 << outcomeSlotCount) - 1;
    // freeIndexSet starts as the full collection
    uint freeIndexSet = fullIndexSet;
    // This loop checks that all condition sets are disjoint (the same outcome is not part of more than 1 set)
    uint[] memory positionIds = new uint[](partition.length);
    uint[] memory amounts = new uint[](partition.length);
    for (uint i = 0; i < partition.length; i++) {
        uint indexSet = partition[i];
        require(indexSet > 0 && indexSet < fullIndexSet, "got invalid index set");
        require((indexSet & freeIndexSet) == indexSet, "partition not disjoint");
        freeIndexSet ^= indexSet;
        positionIds[i] = CTHelpers.getPositionId(collateralToken, CTHelpers.getCollectionId(parentCollectionId, conditionId, indexSet));
        amounts[i] = amount;
    }

    if (freeIndexSet == 0) {
        // Partitioning the full set of outcomes for the condition in this branch
        if (parentCollectionId == bytes32(0)) {
            require(collateralToken.transferFrom(msg.sender, address(this), amount), "could not receive collateral tokens");
        } else {
            _burn(
                msg.sender,
                CTHelpers.getPositionId(collateralToken, parentCollectionId),
                amount
            );
        }
    } else {
        // Partitioning a subset of outcomes for the condition in this branch.
        // For example, for a condition with three outcomes A, B, and C, this branch
        // allows the splitting of a position $:(A|C) to positions $:(A) and $:(C).
        _burn(
            msg.sender,
            CTHelpers.getPositionId(collateralToken,
                CTHelpers.getCollectionId(parentCollectionId, conditionId, fullIndexSet ^ freeIndexSet)),
            amount
        );
    }

    _batchMint(
        msg.sender,
        // position ID is the ERC 1155 token ID
        positionIds,
        amounts,
        ""
    );
    emit PositionSplit(msg.sender, collateralToken, parentCollectionId, conditionId, partition, amount);
}
```

`splitPosition` 메서드는 다음 매개변수를 받습니다:

  * `collateralToken`: 담보 토큰의 주소.
  * `parentCollectionId`: 부모 결과 컬렉션 ID. 담보에서 분할하는 경우 이는 null입니다.
  * `conditionId`: 분할할 조건 이벤트 ID.
  * `partition`: 분할될 하위 결과 컬렉션의 `Index Sets`를 나타내는 배열.
  * `amount`: 분할할 금액.

<!-- end list -->

```solidity
// For a condition with 4 outcomes fullIndexSet's 0b1111; for 5 it's 0b11111...
uint fullIndexSet = (1 << outcomeSlotCount) - 1;
// freeIndexSet starts as the full collection
uint freeIndexSet = fullIndexSet;
```

위의 코드는 주어진 조건 이벤트에 대한 모든 가능한 결과의 전체 집합 `fullIndexSet`을 계산하고(예: 4개의 결과에 대해 `fullIndexSet = 0b1111`), 아직 하위 컬렉션에 할당되지 않은 결과를 추적하기 위해 `freeIndexSet`을 초기화합니다.

```solidity
for (uint i = 0; i < partition.length; i++) {
    uint indexSet = partition[i];
    require(indexSet > 0 && indexSet < fullIndexSet, "got invalid index set");
    require((indexSet & freeIndexSet) == indexSet, "partition not disjoint");
    freeIndexSet ^= indexSet;
    positionIds[i] = CTHelpers.getPositionId(collateralToken, CTHelpers.getCollectionId(parentCollectionId, conditionId, indexSet));
    amounts[i] = amount;
}
```

이 코드는 `partition` 배열을 반복하여 각 하위 컬렉션을 검증합니다:

  * 각 하위 컬렉션은 비어 있지 않아야 하며 전체 집합과 같지 않아야 합니다([결과 컬렉션](#결과-컬렉션-outcome-collection)에서 언급했듯이 공집합과 전체 집합은 실질적인 의미가 없습니다).

  * 하위 컬렉션은 서로 소(disjoint)여야 합니다(중복되는 결과가 없어야 함).
    이는 비트 `indexSet & freeIndexSet`을 사용하여 확인하며, 남은 할당되지 않은 결과는 XOR `freeIndexSet ^= indexSet`을 사용하여 업데이트됩니다.
    동시에 각 하위 컬렉션에 대한 포지션 ID가 계산되고 분할 수량이 `amounts` 배열에 저장됩니다. 분할 전후의 총 가치는 같아야 하므로 각 하위 컬렉션의 수량은 전달된 `amount`와 같습니다(하위 컬렉션의 확률 합이 부모 컬렉션의 확률과 같기 때문).

  * `freeIndexSet`이 0이면 전체 집합이 분할되고 있음을 의미합니다:

      * `parentCollectionId`가 null이면 담보에서 분할됨을 의미하므로 해당 양의 담보 토큰이 사용자 계정에서 차감됩니다.
      * 그렇지 않으면 해당 양의 부모 결과 컬렉션 토큰이 사용자 계정에서 차감됩니다.

  * `freeIndexSet`이 0이 아니면 부분 집합이 분할되고 있음을 의미하므로 해당 양의 부모 결과 컬렉션 토큰이 차감됩니다. 전체 집합 분할과 달리 여기서는 부모 집합의 인덱스 세트를 `fullIndexSet ^ freeIndexSet`으로 계산해야 하며, 이는 이미 분할된 결과 컬렉션을 나타냅니다.

마지막으로 ERC-1155 `_batchMint` 메서드가 호출되어 분할된 하위 결과 컬렉션 토큰을 사용자 계정으로 발행하며, 포지션 ID는 토큰 ID가 됩니다. 각 하위 컬렉션의 양은 `amount`입니다.

### 포지션 병합 (Merge Positions)

이것은 분할(Split)의 역연산입니다. 사용자가 모든 가능한 결과(또는 부모 컬렉션을 형성하는 집합)에 대한 토큰을 수집했을 때, 이를 다시 담보로 병합하거나 더 큰 결과 컬렉션 토큰으로 병합할 수 있습니다.

![](./assets/merge_1.png)

```solidity
function mergePositions(
    IERC20 collateralToken,
    bytes32 parentCollectionId,
    bytes32 conditionId,
    uint[] calldata partition,
    uint amount
) external {
    require(partition.length > 1, "got empty or singleton partition");
    uint outcomeSlotCount = payoutNumerators[conditionId].length;
    require(outcomeSlotCount > 0, "condition not prepared yet");

    uint fullIndexSet = (1 << outcomeSlotCount) - 1;
    uint freeIndexSet = fullIndexSet;
    uint[] memory positionIds = new uint[](partition.length);
    uint[] memory amounts = new uint[](partition.length);
    for (uint i = 0; i < partition.length; i++) {
        uint indexSet = partition[i];
        require(indexSet > 0 && indexSet < fullIndexSet, "got invalid index set");
        require((indexSet & freeIndexSet) == indexSet, "partition not disjoint");
        freeIndexSet ^= indexSet;
        positionIds[i] = CTHelpers.getPositionId(collateralToken, CTHelpers.getCollectionId(parentCollectionId, conditionId, indexSet));
        amounts[i] = amount;
    }
    _batchBurn(
        msg.sender,
        positionIds,
        amounts
    );

    if (freeIndexSet == 0) {
        if (parentCollectionId == bytes32(0)) {
            require(collateralToken.transfer(msg.sender, amount), "could not send collateral tokens");
        } else {
            _mint(
                msg.sender,
                CTHelpers.getPositionId(collateralToken, parentCollectionId),
                amount,
                ""
            );
        }
    } else {
        _mint(
            msg.sender,
            CTHelpers.getPositionId(collateralToken,
                CTHelpers.getCollectionId(parentCollectionId, conditionId, fullIndexSet ^ freeIndexSet)),
            amount,
            ""
        );
    }

    emit PositionsMerge(msg.sender, collateralToken, parentCollectionId, conditionId, partition, amount);
}
```

`mergePositions` 메서드의 매개변수와 로직은 `splitPosition`과 매우 유사합니다. 주요 차이점은 다음과 같습니다:

  * 먼저 `_batchBurn`을 호출하여 사용자의 계정에서 모든 하위 결과 컬렉션 토큰을 차감합니다.
  * 그런 다음 `freeIndexSet`의 값에 따라 다시 담보로 병합할지 아니면 더 큰 결과 컬렉션 토큰으로 병합할지 결정합니다.
      * `freeIndexSet`이 0이면 전체 집합이 병합되고 있음을 의미합니다:
          * `parentCollectionId`가 null이면 다시 담보로 병합됨을 의미하므로 해당 양의 담보 토큰이 사용자에게 전송됩니다.
          * 그렇지 않으면 해당 양의 부모 결과 컬렉션 토큰이 사용자를 위해 발행됩니다.
      * `freeIndexSet`이 0이 아니면 부분 집합이 병합되고 있음을 의미하므로 해당 양의 부모 결과 컬렉션 토큰이 사용자를 위해 발행됩니다.

### 포지션 상환 (Redeem Positions)

이것은 오라클이 결과를 게시한 후의 정산 작업입니다. "올바른 결과"에 대한 토큰을 보유한 사용자만 가치를 상환할 수 있습니다. 잘못된 결과에 대한 토큰은 가치가 없어집니다.

```solidity
function redeemPositions(IERC20 collateralToken, bytes32 parentCollectionId, bytes32 conditionId, uint[] calldata indexSets) external {
    uint den = payoutDenominator[conditionId];
    require(den > 0, "result for condition not received yet");
    uint outcomeSlotCount = payoutNumerators[conditionId].length;
    require(outcomeSlotCount > 0, "condition not prepared yet");

    uint totalPayout = 0;

    uint fullIndexSet = (1 << outcomeSlotCount) - 1;
    for (uint i = 0; i < indexSets.length; i++) {
        uint indexSet = indexSets[i];
        require(indexSet > 0 && indexSet < fullIndexSet, "got invalid index set");
        uint positionId = CTHelpers.getPositionId(collateralToken,
            CTHelpers.getCollectionId(parentCollectionId, conditionId, indexSet));

        uint payoutNumerator = 0;
        for (uint j = 0; j < outcomeSlotCount; j++) {
            if (indexSet & (1 << j) != 0) {
                payoutNumerator = payoutNumerator.add(payoutNumerators[conditionId][j]);
            }
        }

        uint payoutStake = balanceOf(msg.sender, positionId);
        if (payoutStake > 0) {
            totalPayout = totalPayout.add(payoutStake.mul(payoutNumerator).div(den));
            _burn(msg.sender, positionId, payoutStake);
        }
    }

    if (totalPayout > 0) {
        if (parentCollectionId == bytes32(0)) {
            require(collateralToken.transfer(msg.sender, totalPayout), "could not transfer payout to message sender");
        } else {
            _mint(msg.sender, CTHelpers.getPositionId(collateralToken, parentCollectionId), totalPayout, "");
        }
    }
    emit PayoutRedemption(msg.sender, collateralToken, parentCollectionId, conditionId, indexSets, totalPayout);
}
```

`redeemPositions` 메서드는 다음 매개변수를 받습니다:

  * `collateralToken`: 담보 토큰의 주소.
  * `parentCollectionId`: 부모 결과 컬렉션 ID. 담보로 상환하는 경우 이는 null입니다.
  * `conditionId`: 상환할 조건 이벤트 ID.
  * `indexSets`: 사용자가 보유한 결과 컬렉션 `Index Sets`를 나타내는 배열.

로직은 다음과 같습니다:

먼저 조건 이벤트 결과가 해결되었는지(`payoutDenominator[conditionId] > 0`) 확인합니다. `payoutDenominator`는 [조건 해결](#조건-해결-resolving-a-condition) 중에만 설정되므로 값이 0이면 결과가 게시되지 않았음을 의미하며 상환이 불가능합니다.

그런 다음 사용자가 제공한 `indexSets` 배열을 반복합니다. 각 결과 컬렉션 `indexSet`에 대해:

  * 유효한지 검증합니다(비어 있지 않고 전체 집합과 같지 않음).
  * 집합에 포함된 모든 결과의 배당금 비율을 합산하여 해당 결과 컬렉션의 배당금 비율 `payoutNumerator`를 계산합니다. `indexSet & (1 << j) != 0`은 집합에 j번째 결과가 포함되어 있는지 확인하며, 그렇다면 해당 배당금 비율이 `payoutNumerator`에 더해집니다. 결과의 배당금 비율이 0이면 합계에 영향을 미치지 않습니다.
      * 여기서는 indexSet이 유효한지 여부를 확인하지 않습니다. [포지션 분할](#포지션-분할-split-position) 및 [포지션 병합](#포지션-병합-merge-positions)에서 사용자가 유효한 결과 컬렉션 토큰만 보유할 수 있도록 보장했기 때문입니다.
      * 사용자가 유효하지 않은 indexSet이나 보유하지 않은 포지션을 상환하려고 하면 `_burn` 호출이 실패하여 보안을 보장합니다.
  * 해당 결과 컬렉션에 대한 사용자의 보유량 `payoutStake`를 가져옵니다.
      * 사용자가 해당 결과 컬렉션에 대한 토큰을 보유하고 있다면 지불해야 할 배당금 금액을 계산하고 사용자 계정에서 토큰을 소각합니다.
      * 실제 배당금 금액은 `payoutStake.mul(payoutNumerator).div(den)`, 즉 보유량에 배당금 비율을 곱한 값으로 계산됩니다.

마지막으로 사용자에게 지급할 배당금 금액이 있는 경우, `parentCollectionId`에 따라 담보 토큰으로 상환할지 아니면 부모 결과 컬렉션 토큰으로 발행할지 결정합니다.

  * `parentCollectionId`가 null이면 담보로 상환하며, 해당 양의 담보 토큰을 사용자에게 전송합니다.
  * 그렇지 않으면 해당 양의 부모 결과 컬렉션 토큰을 사용자를 위해 발행합니다.

## 요약

조건부 토큰의 핵심 가치는 비즈니스 문제를 해결하기 위해 수학적 깊이로 파고드는 데 있습니다:

1.  복잡한 조합 예측을 지원하기 위해 조건의 임의 중첩을 지원해야 합니다.
2.  중첩으로 인한 유동성 파편화를 피하기 위해 **경로 독립성**(교환 법칙을 만족하는 알고리즘 사용)을 보장해야 합니다.
3.  공격자가 알고리즘을 악용하여 토큰을 위조하는 것을 막기 위해 단순 XOR을 버리고 안전한 **타원 곡선 점 덧셈**을 선택했습니다.

이 전체 설계를 통해 개발자는 이더리움에서 유동성이 깊고, 구성 가능하며, 안전한 예측 시장 애플리케이션을 구축할 수 있습니다.

## 참고 문헌

* [조건부 토큰 문서](https://conditional-tokens.readthedocs.io/en/latest/)
* [조건부 토큰 컨트랙트](https://github.com/gnosis/conditional-tokens-contracts)
