[English](./README.md) | [中文](./README_zh.md)

# VRGDA 알고리즘 분석

> 이 글은 Paradigm의 블로그 [VRGDA](https://www.paradigm.xyz/2022/08/vrgda)를 간단히 정리한 것입니다. 자세한 내용은 원문을 읽어보시기 바랍니다.

## 개요

[VRGDA](https://www.paradigm.xyz/2022/08/vrgda)는 Variable Rate Gradual Dutch Auctions(가변 비율 점진적 더치 경매)의 약자로, Paradigm이 제안한 토큰 발행 메커니즘입니다. 목적은 맞춤형 토큰 발행 모델을 통해 점진적 더치 경매의 효과를 달성하는 것입니다. 시장 수요가 예상을 초과하면 가격이 상승하고, 반대로 시장 수요가 예상보다 낮으면 가격이 하락하며, 시장 수요가 예상과 일치하면 가격은 설정된 목표 가격과 같습니다.

[Art Gobblers](https://artgobblers.com/) 프로젝트에서는 두 가지 유형의 NFT가 VRGDA를 사용하여 경매됩니다. 상한선이 10,000개로 고정된 Gobbler와 상한선이 없는 Page입니다. 이 두 토큰의 발행 속도는 아래 그림과 같습니다:

![](./assets/vrgda_schedule_examples.png)

## 정의

### 함수

더치 경매의 효과를 얻으려면 시간 $t$가 증가함에 따라 가격 $p$가 하락하는 경향을 보이도록 가격 $p$와 시간 $t$의 함수를 찾아야 합니다.

Paradigm의 [GDA 기사](https://www.paradigm.xyz/2022/04/gda)에서 언급했듯이 많은 함수가 요구 사항을 충족할 수 있습니다.

$$
p_n(t) = k \cdot \alpha^n e^{-\lambda t} \quad \text{(1)}
$$

여기서는 다음 함수를 선택합니다:

$$
p_n(t) = p_0 \cdot c^t \text{, 0 < c < 1} \quad \text{(2)}
$$

여기서 $p_0$는 초기 가격이고, $0 \lt c \lt 1$이므로 시간 $t$가 증가함에 따라 가격 $p_n$은 $p_0$보다 작아집니다.

$c = 0.7$이고 $t$의 시간 단위가 일(day)이라고 가정하면, 가격은 매일 전날 가격의 70%로 감소합니다.

가격(수직축) 대 시간(수평축)의 곡선은 아래 그림과 같습니다:

![](./assets/p1.jpg)

### 매개변수

다음 매개변수를 정의합니다:

* $p_0$ - 목표 가격. NFT가 계획된 속도로 경매되는 경우, 즉 시장 수요가 예상과 일치하는 경우 각 NFT는 이 가격에 판매됩니다.
* $k$ - 단위 시간 내에 NFT가 판매되지 않으면 NFT 가격이 이 비율만큼 감소합니다.
* $f(t)$ - 토큰 발행 모델, 시간 $t$ 내에 발행될 것으로 예상되는 NFT 수를 나타냅니다.

식 (1)에서 $c$를 $1-k$로 대체합니다:

$$
p_n(t) = p_0 \cdot (1-k)^t \quad \text{(3)}
$$

$k=0.30$, $t$의 시간 단위는 일(day), 초기 가격은 $p_0$라고 가정하면, 거래가 없을 경우 가격은 매일 30%씩 감소합니다(전날 가격의 70%).

### 토큰 모델

지금까지 식 (3)은 시간에 따라 가격이 감소하는 효과를 달성했습니다.

가격에 시장 수요를 반영하기 위해 $t$를 $t - s_n$으로 대체합니다. 여기서 $s_n$은 경매된 토큰 수 $n$에 의해 정의된 함수입니다.

우리는 $s_n$이 다음과 같은 효과를 얻을 수 있기를 바랍니다:

1. 경매 진행이 예상보다 빠르면 다음 NFT의 시작 가격은 $p_0$보다 큽니다.
2. 경매 진행이 예상보다 느리면 다음 NFT의 시작 가격은 $p_0$보다 작습니다.
3. 경매 진행이 예상과 일치하면 다음 NFT의 시작 가격은 $p_0$와 같습니다.

경매 진행이 예상과 일치한다고 가정하면 다음을 유도할 수 있습니다:

$$
p_0 \cdot (1-k)^{t_n - s_n} = p_0 \quad \text{(4)}
$$

따라서 $t_n = s_n$이므로 $s_n$은 시간과 토큰 수량의 함수, 즉 토큰 수량과 시간 함수 $f(t)$의 역함수로 볼 수 있습니다:

$$
s_n = t_n = f^{-1}(n) \quad \text{(5)}
$$

여기서 토큰 수량과 시간 함수 $f(t)$는 토큰 발행 모델입니다.

$0 \lt (1-k) \lt 1$이므로 경매 진행이 예상보다 빠르면 $t < s_n$, 즉 $t - s_n < 0$이 되어 $(1-k)^{t - s_n} \gt 1$이 되고, $p_n \gt p_0$, 즉 시작 가격이 $p_0$보다 크다는 것을 알 수 있습니다. 마찬가지로 경매 진행이 예상보다 느리면 $p_n \lt p_0$, 즉 시작 가격이 $p_0$보다 작아집니다.

특히 시장이 과열되면 경매 가격이 급격히 상승하여 시장의 열기를 식히고 고래가 대량으로 경매하는 것을 방지합니다.

이것은 경매 가격과 시장 열기 간의 상호 작용을 달성합니다.

### VRGDA 공식

VRGDA의 최종 공식은 다음과 같습니다:

$$
vrgda_n(t) = p_0(1-k)^{t - f^{-1}(n)} \quad \text{(6)}
$$

여기서,

* $p_0$ - 목표 가격.
* $k$ - 단위 시간당 가격 감소 비율.
* $f^{-1}(n)$ - 시간과 토큰 수량의 함수, 즉 토큰 발행 모델 $f(t)$의 역함수.
* $t$ - 경매 시작부터 현재까지의 시간.

## 일반적인 토큰 발행 모델

다음으로 몇 가지 일반적인 토큰 발행 모델과 함께 VRGDA 알고리즘을 소개합니다.

### 선형 (Linear)

하루에 판매되는 예상 NFT 수가 $r$이라고 가정하면 토큰 수량과 시간의 함수(발행 모델)는 다음과 같습니다:

$$
f(t) = rt
$$

해당 곡선은 다음과 같습니다:

![](./assets/vrgda_linear_issuance_schedule.png)

그 역함수(시간과 토큰 수량의 함수)는 다음과 같습니다:

$$
f^{-1}(n) = \frac{n}{r}
$$

공식 (6)에 대입하면 선형 발행 속도에 대한 VRGDA 공식은 다음과 같습니다:

$$
{linearvrgda}_n(t) = p_0(1-k)^{t - \frac{n}{r}} \quad \text{(7)}
$$

### 제곱근 (Square Root)

1번째 NFT의 예상 시간은 1일째, 2번째 NFT는 4일째, $n$번째 NFT는 $n^2$일째라고 가정합니다. 즉, 토큰 수량과 시간의 함수는 다음과 같습니다:

$$
f(t) = \sqrt{t}
$$

해당 곡선은 다음과 같습니다:

![](./assets/vrgda_sqrt_issuance_schedule.png)

그 역함수는 다음과 같습니다:

$$
f^{-1}(n) = n^2
$$

공식 (6)에 대입하면:

$$
{sqrtvrgda}_n(t) = p_0(1-k)^{t - n^2} \quad \text{(8)}
$$

### 로지스틱 함수 (Logistic Function)

로지스틱 함수([Logistic Function](https://en.wikipedia.org/wiki/Logistic_function))는 S자형 함수로, 다음과 같이 단순화됩니다:

$$
l(t) = \frac{1}{1 + e^{-t}}
$$

그림과 같이 모든 입력을 $(0,1)$ 범위로 압축할 수 있습니다:

![](./assets/logistic_function.jpg)

로지스틱 함수를 상한선이 있는 발행 모델로 사용할 수 있습니다.

먼저 함수를 수직축을 따라 0.5만큼 아래로 이동합니다($t=0$일 때 $l(t)=0.5$이므로). $t \gt 0$을 취하면 $0 \lt l(t) \lt 0.5$가 되고 함수는 다음과 같습니다:

$$
l(t) = \frac{1}{1 + e^{-t}} - 0.5 \text{, t > 0}
$$

총 토큰 수를 $L - 1$로 하려면 위 공식을 $2L$로 스케일링하고 발행 속도를 제어하기 위해 시간 승수 $s$를 도입하여 발행 함수를 얻습니다:

$$
f(t) = \frac{2L}{1 + e^{-st}} - L \text{, t > 0}
$$

$t = \frac{1}{s}$일 때 다음을 얻습니다:

$$
\frac{f(\frac{1}{s})}{L} = \frac{2}{1 + e^{-1}} - 1 \approx 0.46
$$

따라서 원하는 시간에 총 토큰 발행량의 46%를 달성하도록 $s$를 선택할 수 있습니다. 예를 들어 100일 후에 총 발행량의 46%를 원한다면 $\frac{1}{s} = 100$이므로 $s = \frac{1}{100} = 0.01$입니다.

$f(t)$의 역함수는 다음과 같습니다:

$$
f^{-1}(n) = -\frac{ln(\frac{2L}{L + n} - 1)}{s}
$$

공식 (6)에 대입하면 로지스틱 함수 기반의 VRGDA 공식은 다음과 같습니다:

$$
logisticvrgda_n(t) = p_0(1-k)^{t + \frac{ln(\frac{2L}{L + n} - 1)}{s}} \quad \text{(9)}
$$

해당 토큰 발행 곡선은 아래와 같습니다:

![](./assets/vrgda_logistic_issuance_schedule.png)

## Art Gobblers에서의 VRGDA

Art Gobblers 프로젝트에서 Gobbler와 Page NFT는 VRGDA를 사용하여 경매되며, 여기서:

* Gobbler는 10,000개의 고정된 상한선이 있는 로지스틱 함수를 발행 모델로 사용합니다.
* Page는 로지스틱과 선형 발행 모델의 조합을 사용하며, 특정 기간 후에는 상한선 없이 선형 발행으로 전환합니다.

Gobbler를 예로 들어 [소스 코드](https://github.com/artgobblers/art-gobblers)에서 VRGDA 경매 매개변수가 어떻게 정의되는지 살펴보겠습니다:

```solidity
constructor(
    // Mint config:
    bytes32 _merkleRoot,
    uint256 _mintStart,
    // Addresses:
    Goo _goo,
    Pages _pages,
    address _team,
    address _community,
    RandProvider _randProvider,
    // URIs:
    string memory _baseUri,
    string memory _unrevealedUri
)
    GobblersERC721("Art Gobblers", "GOBBLER")
    Owned(msg.sender)
    LogisticVRGDA(
        69.42e18, // Target price.
        0.31e18, // Price decay percent.
        // Max gobblers mintable via VRGDA.
        toWadUnsafe(MAX_MINTABLE),
        0.0023e18 // Time scale.
    )
```

여기서 마지막 몇 줄에 주목합니다:

* `69.42e18`: 목표 가격(초기 가격) $p_0$.
* `0.31e18`: 하루 안에 경매가 성공하지 못하면 가격이 전날 가격의 `1-0.31=69%`가 됨을 나타냅니다.
* `MAX_MINTABLE`: 경매 가능한 최대 토큰 수, 즉 $L - 1$ (참고: $L$ 아님).
* `0.0023e18`: 시간 승수 $s$, $\frac{1}{0.0023} \approx 435$는 [Gobbler 경매 계획](https://docs.google.com/spreadsheets/d/1i8hYuWyAymjbwx54fA1HcEMEwfEEyluX4tPssOXoyf4/edit#gid=1608966566)에 따라 435일 안에 총 토큰의 46%를 경매할 것으로 예상함을 나타내며, 15개월(약 435일) 경에 경매 물량의 46%에 도달합니다.

## 요약

VRGDA는 토큰 모델을 위한 맞춤형 점진적 더치 경매 방식을 제안하여 시장이 경매된 토큰의 가격 책정에 참여할 수 있도록 합니다. 사실 이 알고리즘은 NFT 경매뿐만 아니라 ERC20 토큰 경매에도 적용될 수 있습니다. VRGDA 알고리즘의 본질은 알고리즘을 통해 시장 가격을 동적으로 조정하는 AMM입니다.

## 참고 문헌

* Variable Rate GDAs: https://www.paradigm.xyz/2022/08/vrgda
* Gradual Dutch Auctions: https://www.paradigm.xyz/2022/04/gda
* Art Gobblers: https://www.paradigm.xyz/2022/09/artgobblers
