# Blackjack — 게임 기획서 (GDD)

> **프로젝트명**: 프로젝트 블랙잭 오리지널 (PROJECT_BLACKJACK-ORIGINAL)
> **장르**: 카지노 캐쥬얼 카드 게임 (Blackjack / 21)
> **플랫폼**: Web — Desktop + Mobile (반응형)
> **게임 방식**: 싱글 플레이어 vs 딜러 (AI)
> **RTP**: 99.5% (최적 플레이 기준, 하우스 엣지 ~0.5%)
> **추출일**: 2026-04-12
> **소스**: Notion `프로젝트 블랙잭 오리지널` → `게임 별 기획함` (7 sub-pages)
> **Figma**: https://www.figma.com/make/pKcuqg9d0AqsNCH514WOEF/PROJECT_BLACKJACK-ORIGINAL

---

## 01. 게임 컨셉 및 개요

### 1.1 게임 장르

- **장르**: 카지노 캐쥬얼 카드 게임 (Blackjack / 21)
- **타겟**: 실제 카지노 규칙을 선호하는 블랙잭 플레이어
- **플랫폼**: 웹 (데스크탑 & 모바일 반응형)
- **게임 방식**: 싱글 플레이어 vs 딜러 (AI)

### 1.2 핵심 컨셉

- 6덱 슈 시스템 (Cut Card 75% 위치)
- 블랙잭 배당 3:2 (150% 페이아웃)
- H17 딜러 규칙 (Soft 17에서 히트)
- DAS (Double After Split) 허용
- 최대 4핸드까지 리스플릿 가능
- A,A 스플릿 후 특수 규칙 (각 1장만 받고 자동 스탠드)
- Late Surrender 지원
- 실시간 카드 애니메이션 & 사운드

### 1.3 게임 목표

- **플레이어 목표**: 딜러보다 21에 가까운 숫자를 만들되, 21을 초과하지 않기
- **블랙잭**: 처음 2장이 Ace + 10 value 카드 = 3:2 배당
- **승리 조건**:
  - 플레이어 핸드 > 딜러 핸드 (21 이하)
  - 딜러 버스트 (21 초과)
  - 플레이어 블랙잭 (딜러가 블랙잭 아닐 때)
- **패배 조건**:
  - 플레이어 버스트
  - 플레이어 핸드 < 딜러 핸드
  - 딜러 블랙잭 (플레이어가 블랙잭 아닐 때)
- **푸시 (무승부)**: 플레이어 = 딜러 → 베팅 금액 반환

---

## 02. 플레이 루프

### 2.1 메인 게임 루프

```
[IDLE] (대기 상태)
  ↓
베팅 금액 선택 ($1 ~ $1000, 칩 선택기)
  ↓
[BETTING] 베팅 확인 버튼 클릭
  ↓
[INITIAL_DEAL] (P-D-P-D 순서로 카드 딜링, 각 0.3초 간격)
  ├─ 플레이어: 2장 오픈
  └─ 딜러: 1장 오픈 (Up Card) + 1장 뒷면 (Hole Card)
  ↓
[딜러 Up Card 체크]
  ├─ Ace → [INSURANCE_OFFER] (보험 제안)
  │   ↓ (Yes/No 선택)
  │   ├─ 딜러 BJ → [DEALER_BLACKJACK] → 보험 2:1 페이아웃
  │   └─ 딜러 BJ 아님 → [PLAYER_TURN]
  │
  ├─ 10-value (10/J/Q/K) → 딜러 피크 (Hole Card 확인)
  │   ├─ BJ → [DEALER_BLACKJACK] → 즉시 정산
  │   └─ BJ 아님 → [PLAYER_TURN]
  │
  └─ 기타 (2~9) → [PLAYER_TURN]
  ↓
[PLAYER_TURN] (플레이어 액션 선택)
  ├─ HIT (히트): 카드 1장 받기
  │   ├─ 21 이하 → 계속 플레이
  │   └─ 21 초과 → BUST (다음 핸드로)
  │
  ├─ STAND (스탠드): 현재 핸드 유지 → 다음 핸드
  │
  ├─ DOUBLE (더블 다운): 베팅 2배 + 카드 1장만 받고 자동 스탠드
  │   └─ 조건: 첫 2장만, 잔액 충분
  │
  ├─ SPLIT (스플릿): 같은 숫자 카드 2장 분리 → 2개 핸드로 플레이
  │   ├─ 조건: 같은 value (10=J=Q=K 동일 취급)
  │   ├─ 최대 4핸드까지 가능
  │   ├─ A,A 스플릿 → 각 1장만 받고 자동 스탠드
  │   └─ DAS: 스플릿 후에도 더블 다운 가능
  │
  └─ SURRENDER (서렌더): 베팅의 50% 돌려받고 포기 (처음 2장만 가능)
  ↓
[모든 핸드 완료 확인]
  ├─ 모든 핸드 BUST → 즉시 정산 (딜러 플레이 스킵)
  └─ 1개 이상 살아있음 → [DEALER_TURN]
  ↓
[DEALER_TURN] (딜러 자동 플레이)
  ├─ Hole Card 공개 (0.8초 대기)
  ├─ 딜러 규칙 (H17):
  │   ├─ Hard/Soft 16 이하 → 무조건 히트
  │   ├─ Soft 17 (A-6) → 히트 (H17 규칙)
  │   └─ Hard 17+ → 스탠드
  ├─ 각 카드 1초 간격으로 딜링
  └─ 버스트 or 17+ 도달 → [SETTLEMENT]
  ↓
[SETTLEMENT] (정산)
  ├─ 각 핸드별 결과 계산:
  │   ├─ 블랙잭: 베팅 × 2.5 (3:2 배당)
  │   ├─ 일반 승리: 베팅 × 2 (1:1 배당)
  │   ├─ 푸시: 베팅 반환
  │   ├─ 패배: 0
  │   └─ 서렌더: 베팅 × 0.5 (이미 지급됨)
  ├─ 결과 팝업 표시 (1.5초 후)
  │   ├─ 패배: 2초 표시
  │   ├─ 승리: 4초 표시
  │   └─ 푸시: 3초 표시
  └─ 잔액 업데이트 → [IDLE]
```

### 2.2 게임 상태 (Phase) 전환

```typescript
type GamePhase =
  | 'idle'              // 대기 (베팅 입력 중)
  | 'betting'           // 베팅 확정 (사용 안 함 - 바로 shuffling/burning으로)
  | 'shuffling'         // 슈 셔플 중 (1초 애니메이션)
  | 'burning'           // 번 카드 제거 중 (0.2초)
  | 'initial_deal'      // 초기 카드 딜링 (P-D-P-D, 총 1.5초)
  | 'insurance_offer'   // 보험 제안 중 (딜러 Ace)
  | 'dealer_blackjack'  // 딜러 블랙잭 확정 (즉시 정산)
  | 'surrender_offer'   // 서렌더 제안 (사용 안 함 - player_turn에서 처리)
  | 'player_turn'       // 플레이어 액션 선택 중
  | 'dealer_turn'       // 딜러 자동 플레이 중
  | 'settlement';       // 정산 & 결과 표시
```

---

## 03. 입력 방식 / 반응성

### 3.1 입력 컨트롤 (데스크탑 & 모바일 공통)

#### 3.1.1 베팅 단계 (IDLE)

**하단 컨트롤 영역 (BottomControls)**

- **베팅 금액 조정 버튼**
  - `-` 버튼: 베팅 금액 감소 (최소 $10)
  - `+` 버튼: 베팅 금액 증가 (최대 $1000 or 잔액)
  - 중앙 골드바 떡칩: 현재 베팅 금액 표시 (`$XX`)
- **BET 버튼 (메인 액션)**
  - 위치: 하단 중앙 (풀 너비)
  - 색상: 골드 그라디언트 (`from-yellow-500 to-amber-600`)
  - 텍스트: `BET $XX` (현재 베팅 금액 표시)
  - 클릭 시: 게임 시작 (카드 딜링 시작)

#### 3.1.2 플레이어 턴 (PLAYER_TURN)

**2행 레이아웃 (모바일 최적화)**

**상단 행 (옵션 버튼):**
- `DOUBLE` / `SPLIT` / `SURRENDER`
- 조건 충족 시에만 활성화
- 비활성화 시: 투명도 50% + 클릭 불가

**하단 행 (메인 액션):**
- `HIT` (좌측, 파란색 `from-blue-600 to-blue-700`)
- `STAND` (우측, 그린 `from-green-600 to-green-700`)
- 큰 터치 영역 (h-14)

#### 3.1.3 보험 선택 (INSURANCE_OFFER)

**모달 팝업 (중앙 오버레이)**
- 제목: "INSURANCE?"
- 메시지: "Dealer shows Ace. Take insurance?"
- 금액 표시: `$XX` (베팅의 50%)
- 버튼:
  - `YES` (그린)
  - `NO` (레드)

#### 3.1.4 결과 확인 (SETTLEMENT)

**WinPopup 컴포넌트**
- 배경 오버레이 (검정 60% 투명도)
- 결과 카드:
  - WIN: 골드 그라디언트 + 트로피 아이콘
  - LOSE: 레드 그라디언트 + X 아이콘
  - PUSH: 블루 그라디언트 + Equal 아이콘
  - BLACKJACK: 엠버 그라디언트 + 다이아몬드 아이콘
- 배당 금액 표시: `+$XXX`
- 자동 닫힘 (2~4초)

### 3.2 반응성 (Responsive Design)

#### 3.2.1 화면 레이아웃 구조

```
┌────────────────────────────────────┐
│ TopBar (고정 높이 60px)            │ ← 게임명, Min/Max Bet, 사운드 토글
├────────────────────────────────────┤
│                                    │
│  Dealer Area (상단 1/3)            │ ← 딜러 카드 + 핸드 밸류
│                                    │
├────────────────────────────────────┤
│                                    │
│  Player Area (하단 2/3)            │ ← 플레이어 카드들 (멀티핸드 스크롤)
│                                    │
│  [Hand 1] [Hand 2] [Hand 3] ...    │
│                                    │
├────────────────────────────────────┤
│ BottomControls (고정 170px)        │ ← 칩 선택기 + 베팅/액션 버튼
│  [칩 8개]                          │
│  [골드바] [-] [+]                  │
│  [BET / HIT / STAND / ...]         │
└────────────────────────────────────┘
```

#### 3.2.2 카드 크기 (Responsive)

- **데스크탑**: 72px × 106px
- **모바일 (< 640px)**: 60px × 88px
- **카드 스택**: 3장 이상 시 `2.5rem` 오버랩

#### 3.2.3 버튼 크기

- **액션 버튼 (HIT/STAND)**: `h-14` (56px)
- **옵션 버튼 (DOUBLE/SPLIT)**: `h-10` (40px)
- **칩**: `w-12 h-12` (48px)

#### 3.2.4 폰트 크기

- **게임명**: `text-xl` (20px)
- **핸드 밸류**: `text-2xl` (24px) / 모바일 `text-xl`
- **버튼 텍스트**: `text-base` (16px)
- **칩 금액**: `text-xs` (12px)

---

## 04. 게임 특징 / 규칙 / 판정 로직

### 4.1 슈 시스템 (Shoe Management) — *원본에서 취소선 처리됨*

#### 4.1.1 슈 구성
- **덱 수**: 6덱 (312장)
- **셔플**: Fisher-Yates 알고리즘 (O(n) 시간복잡도)
- **Cut Card**: 75% 위치 (234장째)
- **번 카드**: 셔플 후 첫 1장 폐기

#### 4.1.2 Cut Card 규칙

```typescript
IF (현재 위치 >= Cut Card 위치) {
  현재 라운드 완료 후 shoe.needsShuffle = true;
  다음 라운드 시작 전 [SHUFFLING] 페이즈 실행 (1초);
  새 슈 생성 → 번 카드 1장 제거;
}
```

#### 4.1.3 비상 셔플 (Emergency Shuffle)

```typescript
IF (shoe.currentPosition >= shoe.cards.length) {
  // 카드 부족 시 즉시 새 슈 생성
  새 슈 초기화;
  로그: "Emergency shuffle triggered";
}
```

### 4.2 카드 밸류 계산 (Hand Value Calculation)

#### 4.2.1 기본 밸류

- **2~10**: 숫자 그대로
- **J, Q, K**: 10
- **A**: 1 또는 11 (상황에 따라 자동 선택)

#### 4.2.2 Soft Hand vs Hard Hand

```typescript
FUNCTION calculateHandValue(cards):
  hardTotal = 0;
  softTotal = 0;
  numAces = 0;

  FOR card IN cards:
    IF card.rank == 'A':
      numAces++;
      hardTotal += 1;
      softTotal += 11;
    ELSE:
      hardTotal += card.value;
      softTotal += card.value;
    END IF
  END FOR

  // Ace를 11로 사용 가능한지 체크
  WHILE (softTotal > 21 AND numAces > 0):
    softTotal -= 10;
    numAces--;
  END WHILE

  isSoft = (numAces > 0 AND softTotal <= 21);
  finalValue = isSoft ? softTotal : hardTotal;
  isBust = (finalValue > 21);
  isBlackjack = (cards.length == 2 AND finalValue == 21);

  RETURN { hard, soft, isSoft, isBust, isBlackjack };
END FUNCTION
```

**예시:**
- `A-6` = Soft 17 (7 or 17)
- `A-6-10` = Hard 17 (Ace는 1로 카운트, 17)
- `A-A-9` = Hard 21 (A=1, A=1, 9 → 11)
- `A-10` (2장) = Blackjack (21)
- `A-5-5` (3장) = Hard 21 (블랙잭 아님)

### 4.3 블랙잭 (Blackjack / Natural 21)

#### 4.3.1 블랙잭 조건

```typescript
isBlackjack = (
  cards.length === 2 AND
  finalValue === 21 AND
  NOT hand.isSplit  // 스플릿 핸드는 블랙잭 불가!
);
```

#### 4.3.2 블랙잭 처리

- **플레이어 BJ + 딜러 BJ** → 푸시 (베팅 반환)
- **플레이어 BJ + 딜러 BJ 아님** → 3:2 배당
  ```typescript
  payout = bet + Math.floor(bet * 1.5);
  // 예: $100 베팅 → $250 수령 ($100 베팅 + $150 배당)
  ```
- **스플릿 후 A+10** → 일반 21로 취급 (블랙잭 아님, 1:1 배당)

### 4.4 딜러 규칙 (Dealer Play)

#### 4.4.1 H17 (Dealer Hits on Soft 17)

```typescript
shouldHit = (
  finalValue < 17 OR
  (dealerHitsSoft17 AND isSoft AND softValue === 17)
);

IF (shouldHit AND NOT isBust):
  drawCard();
  재귀 호출 dealerPlay();
ELSE:
  STAND → settle();
END IF
```

**예시:**
- Hard 16 → 히트
- Soft 17 (A-6) → **히트** (H17 규칙)
- Hard 17 → 스탠드
- Soft 18 (A-7) → 스탠드

#### 4.4.2 딜러 피크 (Dealer Peek)

```typescript
IF (dealerUpCard.rank == 'A'):
  [INSURANCE_OFFER] → 플레이어 선택;
  IF (holeCard.value == 10):
    [DEALER_BLACKJACK] → 즉시 정산;
  END IF
ELSE IF (dealerUpCard.value == 10):
  IF (holeCard.rank == 'A'):
    [DEALER_BLACKJACK] → 즉시 정산;
  END IF
END IF
```

### 4.5 플레이어 액션 상세

#### 4.5.1 HIT (히트)

```typescript
조건:
  - hand.status == 'active'
  - NOT hand.isSplitAces (A,A 스플릿은 히트 불가)

실행:
  card = drawCard();
  hand.cards.push(card);
  hand.handValue = calculateHandValue(hand.cards);

  IF (hand.handValue.isBust):
    hand.status = 'bust';
    moveToNextHand();
  END IF
```

#### 4.5.2 STAND (스탠드)

```typescript
조건:
  - hand.status == 'active'

실행:
  hand.status = 'stand';
  moveToNextHand();
```

#### 4.5.3 DOUBLE (더블 다운)

```typescript
조건:
  - hand.cards.length == 2 (첫 2장만)
  - balance >= hand.bet (잔액 충분)
  - NOT hand.isSplit OR config.allowDAS (스플릿 후는 DAS 설정 확인)

실행:
  balance -= hand.bet;
  hand.bet *= 2;
  hand.isDoubled = true;

  card = drawCard();
  hand.cards.push(card);
  hand.handValue = calculateHandValue(hand.cards);

  IF (hand.handValue.isBust):
    hand.status = 'bust';
  ELSE:
    hand.status = 'doubled';
  END IF

  moveToNextHand(); // 자동으로 다음 핸드
```

**중요**: 더블 다운 시 카드 1장만 받고 자동 스탠드!

#### 4.5.4 SPLIT (스플릿)

```typescript
조건:
  - hand.cards.length == 2
  - card1.value == card2.value (10=J=Q=K 동일 취급)
  - playerHands.length < config.maxSplitHands (최대 4핸드)
  - balance >= hand.bet
  - NOT (hand.isSplit AND card.rank == 'A' AND NOT config.allowResplitAces)

실행:
  card1 = hand.cards[0];
  card2 = hand.cards[1];
  isAces = (card1.rank == 'A');

  balance -= hand.bet;

  // 2개 핸드로 분리
  hand1 = createHand([card1], bet, isSplit=true, isSplitAces=isAces);
  hand2 = createHand([card2], bet, isSplit=true, isSplitAces=isAces);

  // 각 핸드에 새 카드 1장씩 딜링
  hand1.cards.push(drawCard());
  hand2.cards.push(drawCard());

  // A,A 스플릿 특수 처리
  IF (isAces):
    hand1.status = 'stand';
    hand2.status = 'stand';
    // A+10 = 21 (블랙잭 아님!)
    hand1.handValue.isBlackjack = false;
    hand2.handValue.isBlackjack = false;

    activeHandIndex += 2; // 두 핸드 모두 스킵
  END IF

  playerHands.splice(activeHandIndex, 1, hand1, hand2);
```

**A,A 스플릿 특수 규칙:**
- 각 핸드에 카드 **1장만** 받음
- 자동 스탠드 (추가 히트 불가)
- A+10 = 21 (블랙잭 아님, 1:1 배당)

**리스플릿 예시:**

```
초기: [9♠, 9♥]
스플릿 →
  Hand 1: [9♠, 9♦] ← 또 9!
  Hand 2: [9♥, ?]

Hand 1 스플릿 →
  Hand 1: [9♠, 3♠]
  Hand 2: [9♦, ?]
  Hand 3: [9♥, ?]

최대 4핸드까지 반복 가능
```

#### 4.5.5 SURRENDER (서렌더)

```typescript
조건:
  - hand.cards.length == 2 (첫 2장만)
  - playerHands.length == 1 (스플릿 안 한 상태)
  - config.allowSurrender == true

실행:
  payout = Math.floor(hand.bet * 0.5);
  balance += payout; // 베팅의 50% 반환
  hand.status = 'surrendered';

  [SETTLEMENT] → 즉시 정산;
```

**Late Surrender**: 딜러가 블랙잭 체크 **후** 가능 (블랙잭이면 서렌더 불가)

#### 4.5.6 INSURANCE (보험)

```typescript
조건:
  - dealerUpCard.rank == 'A'
  - config.allowInsurance == true

제안:
  insuranceBet = Math.floor(totalBet / 2);

  IF (플레이어 수락 AND balance >= insuranceBet):
    balance -= insuranceBet;

    // 딜러 Hole Card 확인
    IF (dealerHasBlackjack):
      payout = insuranceBet * 3; // 2:1 배당 (원금 + 2배)
      balance += payout;

      // 메인 베팅 처리
      IF (playerHasBlackjack):
        balance += playerBet; // 푸시
      ELSE:
        // 메인 베팅은 잃음 (이미 차감됨)
      END IF

      [SETTLEMENT];
    ELSE:
      // 보험금은 잃음 (이미 차감됨)
      [PLAYER_TURN];
    END IF
  END IF
```

**보험 페이아웃:**
- 베팅: $100
- 보험금: $50
- 딜러 BJ 시: $50 × 3 = **$150** 수령
- 메인 베팅 손실: -$100
- **순손실**: -$100 + $150 = **+$50**

### 4.6 정산 로직 (Settlement)

#### 4.6.1 정산 순서

```typescript
FOR EACH hand IN playerHands:
  result = 'lose';
  payout = 0;

  // 1. 서렌더 (이미 정산됨)
  IF (hand.status == 'surrendered'):
    result = 'surrender';
    payout = 0; // 이미 50% 반환됨
    CONTINUE;

  // 2. 플레이어 버스트
  IF (hand.handValue.isBust):
    result = 'lose';
    payout = 0;
    CONTINUE;

  // 3. 플레이어 블랙잭
  IF (hand.handValue.isBlackjack AND NOT hand.isSplit):
    IF (dealerHasBlackjack):
      result = 'push';
      payout = hand.bet; // 베팅 반환
    ELSE:
      result = 'blackjack';
      payout = hand.bet + Math.floor(hand.bet * 1.5); // 3:2 배당
    END IF
    CONTINUE;

  // 4. 딜러 블랙잭
  IF (dealerHasBlackjack):
    result = 'lose';
    payout = 0;
    CONTINUE;

  // 5. 딜러 버스트
  IF (dealerValue.isBust):
    result = 'win';
    payout = hand.bet * 2; // 1:1 배당 (베팅 + 배당)
    CONTINUE;

  // 6. 점수 비교
  playerValue = hand.handValue.isSoft ? hand.handValue.soft : hand.handValue.hard;
  dealerValue = dealerHand.handValue.isSoft ? dealerHand.handValue.soft : dealerHand.handValue.hard;

  IF (playerValue > dealerValue):
    result = 'win';
    payout = hand.bet * 2; // 1:1 배당
  ELSE IF (playerValue == dealerValue):
    result = 'push';
    payout = hand.bet; // 베팅 반환
  ELSE:
    result = 'lose';
    payout = 0;
  END IF

  balance += payout;
  totalPayout += payout;
END FOR
```

#### 4.6.2 배당 테이블

| 결과 | 조건 | 배당 | 예시 (베팅 $100) |
|------|------|------|------------------|
| **Blackjack** | A+10 (2장, 비스플릿) | **3:2** | $100 → **$250** |
| **Win** | 플레이어 > 딜러 or 딜러 BUST | **1:1** | $100 → **$200** |
| **Push** | 플레이어 = 딜러 | **0** | $100 → **$100** (반환) |
| **Lose** | 플레이어 < 딜러 or 플레이어 BUST | **-1** | $100 → **$0** |
| **Surrender** | 포기 | **-0.5** | $100 → **$50** |
| **Insurance (당첨)** | 딜러 BJ | **2:1** | $50 → **$150** |

#### 4.6.3 더블 다운 배당 (CRITICAL!)

```typescript
// 올바른 로직
IF (hand.isDoubled):
  // hand.bet는 이미 2배로 증가됨
  IF (playerWin):
    payout = hand.bet * 2; // 예: $200 베팅 → $400 수령
  ELSE IF (push):
    payout = hand.bet;     // 예: $200 베팅 → $200 반환
  ELSE IF (lose):
    payout = 0;            // 예: $200 베팅 → $0
  END IF
END IF
```

**예시:**
- 초기 베팅: $100
- 더블 다운 → `hand.bet = $200` (잔액 -$100 추가 차감)
- **WIN**: `payout = $200 × 2 = $400`
- **PUSH**: `payout = $200` (베팅 전액 반환)
- **LOSE**: `payout = $0` (총 $200 손실)

---

## 05. 보상 / 수치테이블

### 5.1 베팅 범위

| 항목 | 값 | 설명 |
|------|-----|------|
| **최소 베팅** | $10 | config.minBet |
| **최대 베팅** | $1000 | config.maxBet |
| **초기 잔액** | $10,000 | balance 초기값 |

> 5.2 — TODO (원본 누락)

### 5.3 배당률 (RTP 99.5% - 최적 플레이 기준)

| 액션 | 배당 | 수익률 |
|------|------|--------|
| **Blackjack** | 3:2 | +150% |
| **일반 승리** | 1:1 | +100% |
| **Push** | 0:0 | 0% |
| **Insurance (당첨)** | 2:1 | +200% (보험금 기준) |
| **Surrender** | -0.5:1 | -50% |
| **패배** | -1:1 | -100% |

### 5.4 게임 설정 (GameConfig)

```typescript
config = {
  numDecks: 6,                    // 6덱 슈
  cutCardPenetration: 0.75,       // 75% 위치
  dealerHitsSoft17: true,         // H17 규칙
  blackjackPayout: 1.5,           // 3:2 배당
  allowDAS: true,                 // DAS 허용
  allowResplitAces: true,         // A,A 리스플릿 허용
  allowSurrender: true,           // 레이트 서렌더
  allowInsurance: true,           // 보험 허용
  maxSplitHands: 4,               // 최대 4핸드
  minBet: 10,
  maxBet: 1000,
  resplitSameValue: true          // 10=J=Q=K 동일
};
```

### 5.5 하우스 엣지 (House Edge)

| 규칙 세트 | 하우스 엣지 |
|-----------|-------------|
| **현재 설정** (6덱, H17, DAS, Surrender) | **~0.5%** |
| 베이직 스트래티지 미사용 | ~2-3% |
| 6:5 블랙잭 (불리) | ~1.4% |
| S17 (유리) | ~0.3% |

---

## 06. UI / UX 흐름

### 6.1.1 화면 구조 (3단 레이아웃)

```
BlackjackEngine (로직 엔진)
└─ BlackjackTablePro (UI 메인)
    ├─ TopBar (고정 헤더)
    ├─ Game Table (중앙 게임 영역)
    │   ├─ Dealer Area
    │   │   ├─ "DEALER" 라벨
    │   │   ├─ PlayingCard[] (딜러 카드들)
    │   │   └─ 핸드 밸류 표시
    │   │
    │   └─ Player Area
    │       ├─ Hand[] (플레이어 핸드들 - 가로 스크롤)
    │       │   ├─ PlayingCard[]
    │       │   ├─ 핸드 밸류
    │       │   ├─ CasinoChip (베팅 금액)
    │       │   └─ Active 표시 (골드 링)
    │       │
    │       └─ Insurance Offer (모달)
    │
    ├─ BottomControls (고정 하단)
    │   ├─ CasinoChipSelector (칩 선택기)
    │   ├─ Bet Amount Display (골드바 떡칩)
    │   ├─ +/- 버튼
    │   └─ Action 버튼들
    │       ├─ BET (idle 상태)
    │       ├─ HIT / STAND (player_turn)
    │       ├─ DOUBLE / SPLIT / SURRENDER (조건부)
    │
    └─ WinPopup (결과 모달)
```

### 6.1.2 PlayingCard 컴포넌트

```typescript
interface PlayingCardProps {
  card: Card;
  hidden?: boolean;          // 딜러 Hole Card
  delay?: number;            // 딜링 애니메이션 딜레이 (초)
  flipDelay?: number;        // 뒤집기 애니메이션 딜레이 (초)
}

// 카드 크기
size = {
  desktop: '72px × 106px',
  mobile: '60px × 88px'
};

// 카드 디자인
- 앞면: suit 아이콘 + rank 텍스트
- 뒷면: 파란 그라디언트 + 패턴
- 애니메이션:
  - 딜링: translateY(-100px) → 0, opacity 0 → 1
  - 뒤집기: rotateY(0deg → 180deg)
```

### 6.1.3 CasinoChip 컴포넌트 (3D 디자인)

**특징:**
- 3D 원근감 (`perspective: 1000px`)
- 16개 스트라이프 인레이 패턴
- 홀로그램 효과 (무지개 광택)
- 세라믹 광택 (상단 하이라이트)
- 엠보싱 금액 텍스트
- 바닥 그림자 + 측면 레이어

**상태:**
- `isSelected`: 골드 링 강조 (`ring-amber-400`)
- `isDisabled`: 흑백 + 투명도 30%
- `hover`: 스케일 110% + Y축 -4px

### 6.1.4 TopBar 컴포넌트

**레이아웃:**

```
┌────────────────────────────────────────────────────┐
│ [Spade Icon] BLACKJACK PRO | 블랙잭                │
│ Min: $10 - Max: $1000              [Sound] [?] [Home] │
└────────────────────────────────────────────────────┘
```

**기능:**
- 게임명 표시 (영문 + 한글)
- Min/Max Bet 표시
- 사운드 토글 버튼
- 도움말 / 홈 버튼 (옵션)

### 6.1.5 BottomControls 컴포넌트

**레이아웃 (IDLE 상태):**

```
┌─────────────────────────────────────────────────┐
│ [칩1][칩5][칩10][칩25][칩50][칩100][칩500][칩1K]│ ← 가로 스크롤
├─────────────────────────────────────────────────┤
│        [$100 골드바]  [-] [+]                   │ ← 베팅 조정
├─────────────────────────────────────────────────┤
│              [BET $100]                         │ ← 메인 액션
└─────────────────────────────────────────────────┘
```

**레이아웃 (PLAYER_TURN 상태):**

```
┌─────────────────────────────────────────────────┐
│    [DOUBLE]     [SPLIT]     [SURRENDER]         │ ← 상단 행
├─────────────────────────────────────────────────┤
│      [HIT]                [STAND]               │ ← 하단 행
└─────────────────────────────────────────────────┘
```

**버튼 색상:**
- BET: `from-yellow-500 to-amber-600` (골드)
- HIT: `from-blue-600 to-blue-700` (블루)
- STAND: `from-green-600 to-green-700` (그린)
- DOUBLE: `from-purple-600 to-purple-700` (퍼플)
- SPLIT: `from-orange-600 to-orange-700` (오렌지)
- SURRENDER: `from-red-600 to-red-700` (레드)

### 6.2 애니메이션 고정 높이 유지

**문제점 방지:**
- 카드 딜링/뒤집기 애니메이션 중 레이아웃 이동 방지
- UI 영역 고정 높이 설정

```typescript
// Dealer Cards Container
<div className="h-[88px] sm:h-[106px]">

// Hand Value Display
<div className="h-8">

// Player Hands Container
<div className="h-[200px] overflow-x-auto">

// BottomControls
<div className="h-[170px]">
```

### 6.3 멀티핸드 레이아웃

#### 6.3.1 핸드 배치 (가로 스크롤)

```
┌──────────────────────────────────────────┐
│  [Hand 0]  [Hand 1]  [Hand 2]  [Hand 3]  │ ← 좌우 스크롤
│   Active     ----     ----      ----     │
│   [칩]      [칩]      [칩]      [칩]     │
└──────────────────────────────────────────┘
```

#### 6.3.2 Active Hand 표시

```typescript
IF (handIndex == activeHandIndex):
  // 골드 링 강조
  className = "ring-4 ring-amber-400 ring-offset-4 ring-offset-slate-900";
  animate = "animate-pulse";
END IF
```

#### 6.3.3 핸드 상태별 스타일

| 상태 | 표시 | 스타일 |
|------|------|--------|
| `active` | 골드 링 | `ring-amber-400` |
| `stand` | 그린 체크 | `text-green-400` "STAND" |
| `bust` | 레드 X | `text-red-500` "BUST" |
| `blackjack` | 엠버 다이아 | `text-amber-400` "BLACKJACK!" |
| `doubled` | 퍼플 2x | `text-purple-400` "DOUBLED" |
| `surrendered` | 오렌지 깃발 | `text-orange-400` "SURRENDER" |

### 6.4 모달 & 팝업

#### 6.4.1 Insurance Offer 모달

```typescript
<div className="bg-black/60 backdrop-blur-sm">
  <div className="bg-gradient-to-br from-blue-900 to-blue-950">
    <h3>INSURANCE?</h3>
    <p>Dealer shows Ace. Take insurance?</p>
    <div>Cost: $50</div>
    <button className="bg-green-600">YES</button>
    <button className="bg-red-600">NO</button>
  </div>
</div>
```

#### 6.4.2 WinPopup 컴포넌트

```typescript
// 결과별 색상
colors = {
  win: 'from-yellow-600 to-amber-700',
  lose: 'from-red-600 to-red-800',
  push: 'from-blue-600 to-blue-800',
  blackjack: 'from-amber-500 to-yellow-600',
  surrender: 'from-orange-600 to-orange-800'
};

// 아이콘
icons = {
  win: <Trophy />,
  lose: <X />,
  push: <Equal />,
  blackjack: <Diamond />,
  surrender: <Flag />
};

// 표시 시간
displayTime = {
  lose: 2000,      // 2초
  win: 4000,       // 4초
  push: 3000,      // 3초
  blackjack: 4000  // 4초
};
```

### 6.5 반응형 브레이크포인트

```css
/* Tailwind v4 Breakpoints */
sm: 640px   → 모바일 (세로)
md: 768px   → 태블릿
lg: 1024px  → 데스크탑
xl: 1280px  → 와이드 데스크탑
```

### 6.6 디자인 토큰 (globals.css)

```css
/* 주요 색상 */
--color-felt-green: #1a4d2e;
--color-gold: #f59e0b;
--color-amber: #d97706;
--color-slate-bg: #0f172a;

/* 그라디언트 */
--gradient-felt: linear-gradient(
  to bottom right,
  rgb(5 46 22),
  rgb(20 83 45),
  rgb(5 46 22)
);

--gradient-gold-chip: linear-gradient(
  to bottom right,
  #fde047,
  #fbbf24,
  #ca8a04
);

/* 그림자 */
--shadow-card: 0 4px 8px rgba(0, 0, 0, 0.3);
--shadow-chip: 0 2px 3px rgba(0, 0, 0, 0.3),
               0 4px 8px rgba(0, 0, 0, 0.2);
```

---

## 07. 사운드 & 연출 타이밍

### 7.1 오디오 시스템 (useGameAudio Hook)

#### 7.1.1 사운드 목록

| 사운드 이름 | 트리거 조건 | 타이밍 |
|-------------|-------------|--------|
| `playBetSound()` | BET 버튼 클릭 | 즉시 |
| `playCardDealSound()` | 카드 딜링 (P-D-P-D) | 각 카드 딜링 시 |
| `playCardFlipSound()` | Hole Card 뒤집기 | dealer_turn 시작 |
| `playHitSound()` | HIT 버튼 클릭 | 카드 딜링 0.1초 전 |
| `playStandSound()` | STAND 버튼 클릭 | 즉시 |
| `playDoubleDownSound()` | DOUBLE 버튼 클릭 | 즉시 |
| `playSplitSound()` | SPLIT 버튼 클릭 | 즉시 |
| `playSurrenderSound()` | SURRENDER 버튼 클릭 | 즉시 |
| `playInsuranceSound()` | Insurance YES/NO 클릭 | 즉시 |
| `playBlackjackSound()` | 블랙잭 달성 | 결과 팝업 표시 시 |
| `playWinSound()` | 일반 승리 | 결과 팝업 표시 시 |
| `playPushSound()` | 푸시 (무승부) | 결과 팝업 표시 시 |
| `playLoseSound()` | 패배 | 결과 팝업 표시 시 |
| `playBustSound()` | 플레이어 버스트 | 버스트 판정 즉시 |
| `playBetIncreaseSound()` | + 버튼 or 칩 클릭 | 즉시 |
| `playBetDecreaseSound()` | - 버튼 클릭 | 즉시 |

### 7.2 애니메이션 타이밍 차트

#### 7.2.1 Initial Deal 시퀀스 (총 1.5초)

```
T+0.0s: 번 카드 제거 (비표시)
T+0.1s: 플레이어 카드 1 딜링 (translateY 애니메이션 0.3s)
        ├─ playCardDealSound()
        └─ delay: 0ms

T+0.4s: 딜러 카드 1 딜링 (Up Card)
        ├─ playCardDealSound()
        └─ delay: 400ms

T+0.7s: 플레이어 카드 2 딜링
        ├─ playCardDealSound()
        └─ delay: 800ms

T+1.0s: 딜러 카드 2 딜링 (Hole Card, 뒷면)
        ├─ playCardDealSound()
        └─ delay: 1200ms

T+2.5s: 딜러 Up Card 체크 (Insurance/Peek)
```

#### 7.2.2 Player Turn - HIT 액션

```
T+0.0s: HIT 버튼 클릭
        └─ playHitSound()

T+0.1s: 카드 딜링 시작
        └─ playCardDealSound()

T+0.3s: 카드 도착 (translateY 완료)

T+0.5s: 카드 뒤집기 (rotateY 180deg, 0.3s)
        └─ playCardFlipSound()

T+0.8s: 핸드 밸류 계산 & 표시
        ├─ IF (BUST): playBustSound()
        └─ 버스트 체크
```

#### 7.2.3 Player Turn - SPLIT 액션

```
T+0.0s: SPLIT 버튼 클릭
        ├─ playSplitSound()
        └─ 핸드 2개로 분리

T+0.3s: Hand 1 새 카드 딜링
        └─ playCardDealSound()

T+0.7s: Hand 2 새 카드 딜링
        └─ playCardDealSound()

T+1.0s: IF (A,A 스플릿):
          ├─ 두 핸드 모두 자동 스탠드
          └─ activeHandIndex += 2
        ELSE:
          └─ Hand 1 플레이 계속
```

#### 7.2.4 Dealer Turn 시퀀스

```
T+0.0s: [DEALER_TURN] 시작

T+0.8s: Hole Card 뒤집기 시작
        ├─ playCardFlipSound()
        └─ rotateY(0 → 180deg), 0.3s

T+1.1s: Hole Card 완전 공개
        └─ 딜러 핸드 밸류 계산

T+1.1s + N×1.0s: 딜러 히트 (N = 추가 카드 수)
  FOR EACH 히트:
    T+0.0s: shouldHit 체크
    T+0.1s: playCardDealSound()
    T+0.3s: 카드 도착
    T+1.0s: 다음 카드 or 스탠드
```

#### 7.2.5 Settlement 시퀀스

```
T+0.0s: 딜러 최종 카드 도착 or 모든 핸드 버스트

T+1.5s: [SETTLEMENT] 페이즈 시작
        ├─ 각 핸드 결과 계산
        └─ 총 배당 계산

T+1.5s: 결과 팝업 표시 (WinPopup)
        ├─ IF (Blackjack): playBlackjackSound()
        ├─ IF (Win): playWinSound()
        ├─ IF (Push): playPushSound()
        └─ IF (Lose): playLoseSound()

T+1.5s + X: 팝업 자동 닫힘
  X = {
    lose: 2000ms,
    win: 4000ms,
    push: 3000ms,
    blackjack: 4000ms
  }

T+1.5s + X: 게임 초기화
  ├─ IF (shoe.needsShuffle):
  │   └─ [SHUFFLING] → 1000ms → [IDLE]
  └─ ELSE:
      └─ [IDLE]
```

### 7.3 파티클 효과 (향후 구현 권장)

#### 7.3.1 칩 클릭 효과

```typescript
onChipClick = (value, event) => {
  const { clientX, clientY } = event;

  spawnParticles({
    x: clientX,
    y: clientY,
    count: 8,
    color: '#f59e0b',
    velocity: 2,
    lifetime: 0.5
  });
};
```

#### 7.3.2 블랙잭 달성 효과

```typescript
onBlackjack = () => {
  spawnStars({
    center: { x: '50%', y: '40%' },
    count: 20,
    color: '#fbbf24',
    animation: 'explode'
  });
};
```

#### 7.3.3 승리 코인 레인

```typescript
onWin = (winAmount) => {
  IF (winAmount > 100):
    coinRain({
      duration: 2000,
      density: 'high',
      coins: ['gold', 'silver']
    });
  END IF
};
```

### 7.4 화면 전환 효과

#### 7.4.1 페이드 인/아웃

```typescript
<motion.div
  initial={{ opacity: 0, scale: 0.9 }}
  animate={{ opacity: 1, scale: 1 }}
  exit={{ opacity: 0, scale: 0.9 }}
  transition={{ duration: 0.2 }}
>
  {/* 모달 내용 */}
</motion.div>
```

#### 7.4.2 결과 팝업 애니메이션

```typescript
<motion.div
  initial={{ y: -100, opacity: 0 }}
  animate={{ y: 0, opacity: 1 }}
  transition={{
    type: 'spring',
    damping: 15,
    stiffness: 300
  }}
>
  {/* WinPopup */}
</motion.div>
```

### 7.5 성능 최적화 가이드

#### 7.5.1 오디오 로딩

```typescript
useEffect(() => {
  preloadSounds([
    'bet', 'card_deal', 'card_flip',
    'hit', 'stand', 'double', 'split',
    'win', 'lose', 'blackjack', 'bust'
  ]);
}, []);
```

#### 7.5.2 애니메이션 최적화

```css
/* GPU 가속 활용 */
.card {
  transform: translateZ(0);
  will-change: transform, opacity;
}

/* 애니메이션 중 레이아웃 재계산 방지 */
.fixed-height-container {
  min-height: 106px;
}
```

#### 7.5.3 React 렌더링 최적화

```typescript
const PlayingCard = React.memo(({ card, hidden, delay }) => {
  // ...
}, (prev, next) => {
  return prev.card.id === next.card.id &&
         prev.hidden === next.hidden;
});
```
