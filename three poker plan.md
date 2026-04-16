# GRAND TABLE · 3 Card Poker — 게임 기획서 (Game Design Document)

> **버전**: v5 배당 / v6 Lightning
> **장르**: 비디오 포커 · 인스턴트 아케이드
> **플랫폼**: Web — Desktop + Mobile (React + Tailwind CSS)
> **Total RTP**: 92.22% (Core 79.88% + Lightning 12.34%)
> **배리언스**: MEDIUM-HIGH

---

## 01. 게임 개요

### 게임 컨셉

전체 유저가 라운드를 공유하는 **공유 라운드 시스템(Shared Round System)** 기반의 3장 카드 포커.
각 라운드는 BET(10초) → HOLD(20초) → RESULT 순으로 자동 진행되며, 유저는 **베팅 금액 선택**과 **홀드할 카드 선택** 두 가지만 결정한다.

### 기본 정보

| 항목 | 내용 |
|------|------|
| 게임명 | GRAND TABLE · 3 Card Poker |
| 카드 구성 | 표준 52장 덱 (조커 없음) |
| 핸드 카드 수 | 3장 |
| 라운드 공유 | 전체 유저 동일 라운드 동시 진행 |
| 설계 버전 | v5 배당 / v6 Lightning |
| localStorage 키 | `grand_table_mults_v5` |

### RTP 요약

| 구성 | 값 |
|------|-----|
| Core RTP | 79.88% |
| Lightning RTP | 12.34% |
| **Total RTP** | **92.22%** |

---

## 02. 게임 플로우

총 3단계 페이즈가 순차적으로 자동 진행된다. 유저 인터랙션이 없어도 타이머 만료 시 자동으로 다음 단계로 넘어간다.

### Phase 1 · BET (10초)

유저가 베팅 금액을 선택하는 단계.

| 항목 | 내용 |
|------|------|
| 제한 시간 | 10초 (`BET_SECS = 10`) |
| 유저 액션 | 베팅 금액 선택 (또는 무베팅 관람) |
| 타이머 만료 | 자동으로 Phase 2(HOLD)로 전환 |
| 무베팅 처리 | 베팅 없이 관람 가능 (`currentBet = 0`) |
| 라이트닝 생성 | 딜 시작 시점에 번개 랭크 1개 생성 |
| 칩 선택지 | $5, $10, $50, $100, $500 |

### Phase 2 · HOLD (20초)

3장의 카드가 공개되고, 유저가 남길 카드(홀드)를 선택한다.

| 항목 | 내용 |
|------|------|
| 제한 시간 | 20초 (`HOLD_SECS = 20`) |
| 유저 액션 | 0~3장 카드 홀드 선택 + 홀드 확정 버튼 |
| 카드 딜 타이밍 | 순차 딜: 190ms 간격, 총 ~1150ms 후 HOLD 시작 |
| 홀드 확정 | "HOLD 확정" 버튼 → 즉시 타이머 종료 + 라이트닝 진입 |
| 타이머 만료 | 홀드 미확정 시 자동으로 현재 선택 상태로 처리 |
| 타이머 사운드 | 6초 이상: 일반 틱 / 5초 이하: 긴급 틱(timerUrgent) |

> **딜 애니메이션 순서**: 카드 데이터 주입(50ms) → 앞면 공개 + 사운드(110ms) → 다음 카드 반복 (190ms 간격)

### Lightning Phase (4초, 자동)

HOLD 확정 또는 타이머 만료 즉시 진입. 라이트닝 배수 연출 후 자동으로 카드를 공개한다.

| 항목 | 내용 |
|------|------|
| 진입 조건 | HOLD 확정 또는 HOLD 타이머 만료 |
| 연출 시간 | 총 4,000ms (4초) |
| 카드 공개 시점 | 라이트닝 연출 시작 후 4,000ms |
| 카드 교체 | 홀드되지 않은 카드를 덱에서 새 카드로 교체 |
| 매칭 계산 | 최종 카드 기준으로 라이트닝 배수 적용 여부 사전 계산 |

### Phase 3 · RESULT (자동 4초 후 다음 라운드)

최종 패를 평가하고 배당을 지급한다.

| 항목 | 내용 |
|------|------|
| 배당 공식 | `win = bet × baseMult + bet × lightningMult` (가산 모델) |
| 라이트닝 보너스 | 기본 족보 WIN일 때만 라이트닝 보너스 가산 |
| 자동 전환 | RESULT 표시 후 4초 후 자동으로 다음 라운드(BET) 시작 |
| 히스토리 저장 | 최근 60라운드 이력 유지 |

### 카드 공개(Reveal) 상세 타이밍

| 단계 | 딜레이 | 동작 |
|------|--------|------|
| 라이트닝 종료 | 0ms | `doActualReveal()` 호출 |
| 뒷면 전환 | 0ms | 홀드되지 않은 카드 뒷면으로 뒤집기 |
| 카드 데이터 주입 | 380ms | 새 카드 데이터 주입 |
| 앞면 공개 시작 | 380 + 55ms | 카드별 순차 공개 시작 |
| 카드 N 공개 | 435 + N × 200ms | 카드별 200ms 간격으로 순차 공개 |
| 결과 계산 | 1000 + (N-1) × 200ms | 족보 평가 + 배당 계산 + 잔액 업데이트 |

---

## 03. 족보 & 배당

### 족보 계층 (우선순위 순)

| 순위 | 족보 | 배수 | 조건 | 색상 코드 |
|------|------|------|------|-----------|
| 1위 | 스트레이트 플러시 (SF) | ×55 | 같은 무늬 + 연속 숫자 (RSF Q-K-A 포함) | `#ffd700` (골드) |
| 2위 | 트리플 | ×28 | 같은 숫자 3장 | `#c084fc` (퍼플) |
| 3위 | 스트레이트 | ×6.5 | 연속 숫자 (무늬 무관) | `#4ade80` (그린) |
| 4위 | 플러시 | ×4 | 같은 무늬 (숫자 무관) | `#e05c5c` (레드) |
| 5위 | 페어 | ×1.2 | 같은 숫자 2장 (랭크 무관 통합) | `#34d399` (틸) |
| 패배 | 하이카드 (HC) | ×0 (LOSS) | A하이 포함, 위 조건 미충족 전부 | — |

> **페어 통합 정책**: v5부터 페어는 Hi/Mid/Lo 구분 없이 단일 ×1.2 배수로 통합. A 하이(ACE_HI)는 하이카드(HC)로 분류, ×0 LOSS 처리.

> **RSF(Royal Straight Flush) 처리**: Q-K-A 동색은 SF와 동일 족보로 ×55 배수 적용. 별도 족보 미분리.

### 정적 핸드 분포 (C(52,3) = 22,100)

| 족보 | 배수 | 핸드 수 | 확률 | RTP 기여 | 비고 |
|------|------|---------|------|----------|------|
| SF | ×55 | 48 | 0.22% | 11.95% | RSF(4) + SF(44) |
| 트리플 | ×28 | 52 | 0.24% | 6.59% | |
| 스트레이트 | ×6.5 | 720 | 3.26% | 21.18% | |
| 플러시 | ×4 | 1,096 | 4.96% | 19.84% | |
| 페어 | ×1.2 | 3,744 | 16.94% | 20.33% | Hi+Mid+Lo 통합 |
| 하이카드 (HC) | ×0 | 16,440 | 74.39% | 0% | ACE_HI 포함 LOSS |
| **합계** | — | **22,100** | **100%** | **79.88%** | Core RTP (정확값) |

> **정적 RTP 계산식**: (48×55 + 52×28 + 720×6.5 + 1096×4 + 3744×1.2) ÷ 22,100 = 17,652.8 ÷ 22,100 = **79.875...% ≈ 79.88%**

### 핸드 평가 로직 (우선순위)

1. **SF** — 플러시 AND 스트레이트 동시 충족
2. **TRIPLE** — `vals[0] === vals[1] === vals[2]`
3. **STRAIGHT** — `(mid=lo+1 AND hi=lo+2) OR (lo=1 AND mid=12 AND hi=13)`
4. **FLUSH** — 3장 모두 동일 무늬
5. **PAIR** — 2장 동일 숫자 (`lo=mid OR mid=hi`)
6. **HC** — 위 조건 전부 미충족 (×0)

### 스트레이트 유효 패턴

| 패턴 | 예시 | 비고 |
|------|------|------|
| 일반 연속 | A-2-3, 2-3-4, ... J-Q-K | A는 1로 취급 (LOW ACE) |
| Q-K-A | Q-K-A (무늬 무관) | HIGH ACE 패턴 — STRAIGHT만 해당 |
| K-A-2 | 불가 | 래핑(Wrapping) 불허 |

### 결과 분류 체계 (OutcomeResult)

| 분류 | 조건 | 해당 족보 |
|------|------|-----------|
| WIN | 배수 ≥ 2 | SF / TRIPLE / STRAIGHT / FLUSH |
| SMALL_WIN | 0 < 배수 < 2 | 페어 (×1.2) |
| LOSS | 배수 = 0 | 하이카드 (HC · ACE_HI 포함) |

> **PUSH 폐지**: v5부터 PUSH 분류 미사용. 코드에 타입은 남아 있으나 실제 반환되지 않음.

---

## 04. 라이트닝 시스템

### 설계 원칙

| 항목 | 값 |
|------|-----|
| 랭크 수 (N) | **N = 1** (매 라운드 단 1개 카드 랭크 선택) |
| 적용 방식 | **가산(Additive)**: `win = bet × baseMult + bet × lMult` |
| 버전 | v6 (v5 곱셈 모델에서 전면 재설계) |

### 배율 룰렛 테이블

| 배수 | 가중치 | 확률 | 색상 |
|------|--------|------|------|
| ×2 | 60 | 60% | `#60a5fa` (라이트 블루) |
| ×3 | 25 | 25% | `#c084fc` (퍼플) |
| ×5 | 15 | 15% | `#ffd700` (골드) |

**기대 배율**: E(mult) = 0.60×2 + 0.25×3 + 0.15×5 = 1.20 + 0.75 + 0.75 = **2.70**

### 배당 공식 (v6 가산 모델)

```
win = bet × baseMult + bet × lMult
```

- `lMult` 조건: `baseMult > 0` AND 라이트닝 매칭 시에만 보너스 가산
- LOSS(HC)이면 라이트닝 보너스 없음 (`lMult = 0` 처리)
- 핸드에 동일 랭크 카드가 여러 장이어도 배수는 1회만 가산 (최고값 1개)

### 예시 계산

| 베팅 | 족보 | 기본 배수 | 라이트닝 | 최종 승리금 | 계산식 |
|------|------|-----------|----------|-------------|--------|
| 100 | 플러시 | ×4 | ×3 매칭 | 700 | 100×4 + 100×3 = 700 |
| 100 | 스트레이트 | ×6.5 | ×5 매칭 | 1,150 | 100×6.5 + 100×5 = 1,150 |
| 100 | 트리플 | ×28 | ×2 매칭 | 3,000 | 100×28 + 100×2 = 3,000 |
| 100 | 페어 | ×1.2 | ×3 매칭 | 420 | 100×1.2 + 100×3 = 420 |
| 100 | 하이카드 | ×0 | 매칭 있어도 | 0 | baseMult=0이므로 lMult 미적용 |
| 100 | 플러시 | ×4 | 매칭 없음 | 400 | 100×4 + 0 = 400 |

### 트리거 확률 계산 (P_win_trigger, N=1)

```
P = Σ (핸드 수 × d(h)) / (22100 × 13)   [d = 핸드 내 고유 랭크 수]

SF       :  48 × 3 =   144   (d=3)
TRIPLE   :  52 × 1 =    52   (d=1, 3장 동일 랭크)
STRAIGHT : 720 × 3 = 2,160   (d=3)
FLUSH    :1096 × 3 = 3,288   (d=3)
PAIR     :3744 × 2 = 7,488   (d=2)

합계 = 13,132 / (22,100 × 13) = 13,132 / 287,300 ≈ 0.0457 (4.57%)
```

### Lightning RTP

```
Lightning RTP = P(win_trigger) × E(mult)
             = 0.0457 × 2.70 ≈ 12.34%
```

> **v5 폐기 사유 (곱셈 모델)**: 구 공식 `win = bet × (baseMult × lMult)` 적용 시 Total RTP ≈ 218% 초과 발생. 곱셈 모델은 `baseMult ≡ 1`인 Lightning Roulette 계열에서만 유효하며, 이 게임에는 적용 불가.

---

## 05. RTP 수학 모델

### 구성 요소

| 구성 | 계산 | 값 |
|------|------|-----|
| Core RTP | (48×55 + 52×28 + 720×6.5 + 1096×4 + 3744×1.2) / 22,100 | 79.88% |
| Lightning RTP | P(trigger) × E(mult) = 0.0457 × 2.70 | 12.34% |
| **Total RTP** | Core + Lightning | **92.22%** |

### 최적 플레이 시뮬레이션

C(52,3) = 22,100 핸드 × 3조합 HOLD2 × 49장 = 약 3.25M 연산. 브라우저 환경에서 ~50~150ms 내 완료.

| 모듈 | 역할 | 연산량 |
|------|------|--------|
| `outcomeCalc.ts` | 전수 열거로 핸드별 최적 EV 계산 | C(52,3) × 49 ≈ 1.08M |
| `rtpOptimizer.ts` | 파라메트릭 시뮬 + 이진 탐색으로 목표 RTP 배당 역산 | ~3.3M eval |
| `rtpSim.ts` | 심플 덱 빌더 유틸 | — |

### HOLD 전략 분류

| 홀드 타입 | 조건 | 설명 |
|-----------|------|------|
| HOLD 3 | EV(현재 패) ≥ EV(최선 HOLD2) AND EV(현재 패) ≥ baseEV | 현재 패가 최선 → 전부 홀드 |
| HOLD 2 | EV(최선 HOLD2) ≥ baseEV | 2장 홀드가 전부 교체보다 유리 |
| HOLD 0 | 위 조건 전부 미충족 | 전부 교체 (baseEV가 가장 높음) |

> **HOLD 1**: 수학적으로 HOLD 0보다 항상 열등 → 스킵 처리.

### 홀드 어드바이저 로직 (holdAdvisor.ts)

8가지 모든 홀드 조합(000 ~ 111)에 대해 기대 배당(EV)을 완전 열거법으로 계산하여 최적 홀드 전략을 반환한다.

**홀드 이유 분류**:
- 현재 패 유지 (3장 모두 홀드)
- 페어 유지 + 업그레이드 (트리플 도전)
- 플러시 도전 (같은 무늬 2장 유지)
- 스트레이트 도전 (연속 숫자 2장 유지)
- 하이카드 홀드 (A/K/Q 유지 후 2장 교체)
- 전체 교체 (새 카드 3장으로 재도전)

> **UI에서 제거됨**: holdAdvisor 연동(자동홀드 제안), OPT 뱃지는 단순화 요청으로 삭제 완료.

---

## 06. UI · 애니메이션

### 컴포넌트 구조

| 컴포넌트 | 역할 |
|----------|------|
| `ThreeCardPoker.tsx` | 메인 게임 컨트롤러 (상태 관리 · 페이즈 전환) |
| `ThreeCardPoker.css` | 전체 게임 스타일시트 |
| `LightningReveal.tsx` | 라이트닝 배수 연출 오버레이 |
| `GameHeader.tsx` | 게임 상단 헤더 (잔액 · 라운드 정보) |
| `HoldWaitPanel.tsx` | HOLD 페이즈 대기 패널 |
| `MiniScoreRoad.tsx` | 미니 스코어로드 (최근 결과 시각화) |
| `StatsPanel.tsx` | 통계 패널 |
| `WinPopup.tsx` | 승리 팝업 |

### LightningReveal 애니메이션 타이밍

반투명 오버레이 방식: 배경·카드가 흐릿하게 비치는 컴팩트 중앙 패널. 전체 화면을 완전히 덮지 않는다.

| 타이밍 상수 | 시간 (ms) | 동작 |
|-------------|-----------|------|
| T_FLASH | 0 | 진입 플래시 효과 (패널 내부 흰색 플래시) |
| T_HEADER | 110 | LIGHTNING STRIKE 헤더 등장 |
| T_RANK_BASE | 340 | 첫 번째 랭크 타일 등장 |
| T_RANK_STEP | 240 | 각 랭크 타일 등장 간격 (N=1이므로 1회) |
| T_MATCH | 860 | MATCH 배너 등장 (매칭 시) / 파티클 발사 |
| T_SCAN | 1,140 | 스캔라인 스윕 효과 |
| T_OUTRO | 3,450 | 페이드아웃 시작 (~550ms) → 총 4,000ms |

### LightningReveal 오버레이 스타일

| 속성 | 값 |
|------|-----|
| 백드롭 배경 | `rgba(0, 0, 0, 0.68)` (반투명) |
| 백드롭 블러 | `blur(5px)` |
| 패널 최대 너비 | 400px (92% 뷰폭) |
| zIndex | 9000 (portal로 `document.body`에 직접 마운트) |

### 랭크 타일 스타일 정책

**전체 활성화 정책**: 매칭 여부에 관계없이 모든 랭크 타일은 항상 활성화된 스타일로 표시. HIT! 뱃지만 실제 매칭 시에만 표시.

### MATCH / NO MATCH 배너 정책

| 조건 | 표시 여부 | 내용 |
|------|-----------|------|
| 라이트닝 매칭 O | 표시 | MATCH! 텍스트 + 배당 부스트 배수 (×N) |
| 라이트닝 매칭 X | 미표시 | NO MATCH 문구 없음 |

### 페이즈 표시 (PhaseSteps)

| 스텝 | 라벨 | 컬러 | 활성 조건 |
|------|------|------|-----------|
| 0 | BET | `#d4a843` (골드) | phase = "betting" |
| 1 | HOLD | `#4ade80` (그린) | phase = "dealing" / "holding" / "lightning" / "revealing" |
| 2 | RESULT | `#d4a843` (골드) | phase = "result" |

### 카드 색상

| 무늬 | 기호 | 색상 |
|------|------|------|
| Spades | ♠ | 검정 |
| Clubs | ♣ | 검정 |
| Hearts | ♥ | 빨강 (#dc2626) |
| Diamonds | ♦ | 빨강 (#dc2626) |

### 족보별 컬러 코드

| 족보 | 컬러 | 용도 |
|------|------|------|
| SF | `#ffd700` (골드) | 카드 테두리 글로우, 결과 텍스트, 파티클 |
| TRIPLE | `#c084fc` (퍼플) | 동일 |
| STRAIGHT | `#4ade80` (그린) | 동일 |
| FLUSH | `#e05c5c` (레드) | 동일 |
| PAIR | `#34d399` (틸) | 동일 |
| HC / 기타 | `rgba(120,105,85,.75)` | 비활성 상태 |

### 결과 모달 (Result Modal)

- `ReactDOM.createPortal`로 `document.body`에 직접 마운트 (스태킹 컨텍스트 탈출)
- 라이트닝 적용 시 기본 배수 × 번개 배수 분해 표시: `×baseMult × ⚡lMult = ×totalMult`
- RESULT 카운트다운 표시: "N초 후 다음 라운드"

### 관전 모드 (Observer)

- 베팅 없이 관람 시 카드 위에 딤 오버레이 표시
- HOLD/LIGHTNING/REVEALING: 진행 상황 프로그레스 바
- RESULT: 이번 라운드 결과 미리보기 + "다음 BET까지 N초" 표시

---

## 07. 사운드 시스템

Web Audio API 기반 합성 사운드 엔진 (`lib/soundManager.ts`). 외부 파일 없이 실시간 합성.

### 사운드 카테고리

| 카테고리 | 설명 |
|----------|------|
| `ui` | 패널 탭, 칩 선택, 홀드 토글 |
| `game` | 라운드 시작, 카드 딜, 홀드 확정, 카드 공개 |
| `timer` | 타이머 틱, 긴급 틱 |
| `winLose` | 소액 승리, 중간 승리, 대형 승리, 패배 |

### 사운드 이벤트 목록

| 이벤트 | 트리거 조건 | 타이밍 |
|--------|-------------|--------|
| `roundStart` | 딜 시작 (Phase 1 → 2 전환) | 베팅 마감 직후 |
| `cardDeal(n)` | 카드 n번 앞면 공개 | 각 카드 딜 시 (110ms + n×190ms) |
| `timerTick` | 홀드 타이머 틱 (6초 이상) | 1초마다 |
| `timerUrgent` | 홀드 타이머 긴급 틱 (5초 이하) | 1초마다 |
| `holdToggle` | 카드 홀드 선택/해제 | 즉시 |
| `holdConfirm` | 홀드 확정 버튼 | 즉시 |
| `cardReveal(n)` | 새 카드 공개 (결과 단계) | 각 카드 공개 시 + 80ms |
| `resultReveal` | 결과 계산 완료 시점 | 족보 평가 직후 |
| `lose` | HC (×0) 패배 | resultReveal + 320ms |
| `winSmall` | PAIR (×1.2) 소액 승리 | resultReveal + 320ms |
| `winMedium` | ×2 이상 ~ ×10 미만 승리 | resultReveal + 320ms |
| `winBig` | 총 배수 ×10 이상 대형 승리 | resultReveal + 320ms |

> **winBig 기준**: `totalMult` (baseMult + lightningMult) ≥ 10

### 사운드 합성 세부

- `tone()`: 오실레이터 기반 주파수 스윕 (sine / triangle / square)
- `noise()`: 대역 통과 필터 노이즈 버스트
- `winSmall`: 청명한 상승 3음 차임 (G5→B5→D6), ~0.8s
- `winMedium`: 소프트 베이스 + 5음 아르페지오 + 코인 3개 + 코드 홀드, ~1.6s
- `winBig`: 서브베이스 더블 펀치 + 7음 아르페지오 + 코인 폭포 12개 + 풀 코드 홀드, ~2.8s

---

## 08. 상태 정의

### GamePhase (게임 페이즈)

```
'betting' → 'dealing' → 'holding' → 'lightning' → 'revealing' → 'result' → 'betting' (반복)
```

| 페이즈 | 설명 | 유저 인터랙션 |
|--------|------|---------------|
| `betting` | 베팅 금액 선택 (10초) | 칩 선택, BET 확정 |
| `dealing` | 카드 순차 딜 (~1.15초) | 없음 |
| `holding` | 홀드 카드 선택 (20초) | 카드 토글, HOLD 확정 |
| `lightning` | 라이트닝 연출 (4초) | 없음 |
| `revealing` | 카드 교체 공개 | 없음 |
| `result` | 결과 표시 (4초) | 없음 |

### HUD 상태 표시

| 항목 | BET 페이즈 | HOLD 페이즈 | LIGHTNING | RESULT |
|------|-----------|-------------|-----------|--------|
| 타이머 컬러 | 3초↓ 빨강, 5초↓ 노랑, 그 외 골드 | 4초↓ 빨강, 8초↓ 노랑, 그 외 그린 | 블루 | — |
| 타이머 값 | `Ns` | `Ns` | ⚡ | — |

---

## 09. 파일 구조

### 핵심 파일

| 파일 경로 | 역할 | 버전 |
|-----------|------|------|
| `/components/ThreeCardPoker.tsx` | 메인 게임 컴포넌트 (상태·로직·렌더) | v5+ |
| `/components/ThreeCardPoker.css` | 게임 전용 스타일시트 | v5+ |
| `/components/LightningReveal.tsx` | 라이트닝 배수 연출 오버레이 | v2 |
| `/components/Spec.tsx` | 공식 기획서 (Spec 컴포넌트) | v5 |
| `/lib/gameConfig.ts` | 배당 테이블 설정 (소스 오브 트루스) | v5 |
| `/lib/lightningSystem.ts` | 라이트닝 배수 생성·계산·RTP 공식 | v6 |
| `/lib/outcomeCalc.ts` | 전수 열거 핸드 확률 계산기 | v5 |
| `/lib/rtpOptimizer.ts` | 파라메트릭 시뮬 + 이진 탐색 최적화 | v5 |
| `/lib/rtpSim.ts` | 시뮬레이션 덱 빌더 유틸 | — |
| `/lib/holdAdvisor.ts` | 홀드 어드바이저 (UI에서 제거됨) | — |
| `/lib/soundManager.ts` | 사운드 이벤트 관리자 | — |

### 제거된 기능

| 제거된 항목 | 이유 |
|-------------|------|
| STAGE 1 / STAGE 2 / REPORT / 확률 탭바 | 단순화 요청 |
| holdAdvisor 연동 (자동홀드 제안) | 단순화 요청 |
| OPT 뱃지 (최적 홀드 표시) | 단순화 요청 |
| NO MATCH 문구 | 디자인 개선 |

### 설정 동기화 주의사항

`gameConfig.ts DEFAULT_MULTS`와 `rtpOptimizer.ts CURRENT_TABLE`은 항상 동일한 값을 유지해야 한다. 불일치 시 RTP 계산 오류 발생.

---

## 10. QA 체크리스트

### 게임 플로우

- [ ] BET 타이머가 10초 후 자동으로 딜을 시작하는가
- [ ] HOLD 타이머가 20초 후 자동으로 라이트닝 페이즈로 전환하는가
- [ ] 홀드 확정 버튼 클릭 시 즉시 타이머 종료 + 라이트닝 페이즈 진입하는가
- [ ] 라이트닝 연출 4초 후 정확히 카드가 공개되는가
- [ ] RESULT 화면이 4초 후 자동으로 다음 BET 페이즈로 전환하는가
- [ ] 카드 딜 애니메이션이 190ms 간격으로 3장 순차 공개되는가
- [ ] 베팅 없이 관람만 해도 게임이 정상 진행되는가
- [ ] 잔액 부족 시 베팅이 처리되지 않고 관람 처리되는가

### 족보 & 배당 계산

- [ ] SF (Q-K-A 동색 RSF 포함) → ×55 정상 지급
- [ ] 트리플 → ×28 정상 지급
- [ ] 스트레이트 (A-2-3 포함) → ×6.5 정상 지급
- [ ] 스트레이트 Q-K-A (HIGH ACE) → ×6.5 정상 지급
- [ ] K-A-2 패턴이 스트레이트로 인정되지 않는가 (불허)
- [ ] 플러시 → ×4 정상 지급
- [ ] 페어 → ×1.2 정상 지급 (Hi/Mid/Lo 구분 없음)
- [ ] 하이카드 및 ACE_HI → ×0 (LOSS) 처리
- [ ] SF가 플러시/스트레이트보다 우선 판정

### 라이트닝 시스템

- [ ] 매 라운드 정확히 1개의 랭크가 생성되는가
- [ ] 배율이 ×2 / ×3 / ×5 중 하나로 생성되는가
- [ ] 라이트닝 배당이 가산 모델인가: `win = bet×baseMult + bet×lMult`
- [ ] HC(×0) 패배 시 라이트닝 매칭이 있어도 보너스가 지급되지 않는가
- [ ] 최종 카드 기준으로 라이트닝 매칭이 판정되는가 (초기 카드 기준 아님)
- [ ] 라이트닝 연출이 4초간 표시되고 닫히는가
- [ ] 매칭 시 MATCH! 배너 표시, 비매칭 시 NO MATCH 문구 미표시

### 잔액 & 히스토리

- [ ] 베팅 시 잔액이 즉시 차감되는가
- [ ] 승리 시 win 금액이 잔액에 정상 가산되는가
- [ ] 히스토리가 최근 60라운드까지 유지되는가

### 수학 모델 검증

- [ ] 정적 RTP 계산 결과가 ~79.88% (17,652.8 ÷ 22,100)
- [ ] Lightning RTP = 0.0457 × 2.70 ≈ 12.34%
- [ ] Total RTP 목표 92.22%에 근사
- [ ] 라이트닝이 곱셈이 아닌 가산 방식으로 구현

---

## 부록: 구현 코드 참조

### 핵심 타입 정의

```typescript
// gameConfig.ts
interface GameMults {
  SF:       number;   // Straight Flush (RSF 포함) ×55
  TRIPLE:   number;   // Three of a Kind ×28
  STRAIGHT: number;   // Straight ×6.5
  FLUSH:    number;   // Flush ×4
  PAIR:     number;   // 페어 통합 ×1.2
}

// lightningSystem.ts
interface LightningRank {
  rank: number;   // 1=A, 2-10, 11=J, 12=Q, 13=K
  mult: number;   // 2 / 3 / 5
}

// ThreeCardPoker.tsx
type SuitKey   = 'spades' | 'hearts' | 'diamonds' | 'clubs';
type GamePhase = 'betting' | 'dealing' | 'holding' | 'lightning' | 'revealing' | 'result';

interface Card        { suit: SuitKey; value: number; id: string; }
interface HandResult  { name: string; nameKor: string; multiplier: number; descriptionKor: string; }
interface RoundHistory{ id: number; finalCards: Card[]; heldIndices: number[]; hand: HandResult; bet: number; win: number; }

// holdAdvisor.ts
interface HoldAdvice {
  holdIndices: number[];   // 홀드 권장 카드 인덱스 (0~2)
  ev: number;              // 기대 배당 배수
  reason: string;
  detail: string;
  evLabel: string;
  allEVs: number[];
}

// outcomeCalc.ts
type OutcomeResult = 'win' | 'small_win' | 'push' | 'loss';

interface OutcomeCalcResult {
  overall:    OutcomeProb;
  byInitHand: Record<string, OutcomeProb & { count: number; avgOptEV: number }>;
  holdBreak:  HoldTypeBreakdown;
  rtp:        number;
  elapsedMs:  number;
}

// soundManager.ts
type SoundCategory = 'ui' | 'game' | 'timer' | 'winLose';
```

### 핵심 함수

```typescript
// gameConfig.ts — 배당 테이블 읽기/쓰기
loadMults(): GameMults                    // localStorage에서 배당 테이블 로드
saveMults(m: GameMults): void            // 배당 테이블 저장 + 이벤트 발행
resetMults(): void                       // 기본값으로 초기화

// lightningSystem.ts — 라이트닝 생성/적용
generateLightningRanks(): LightningRank[]         // N=1 랭크 무작위 선택 + 배율 롤
applyLightningToHand(cards, ranks): number        // 최종 패에서 최고 매칭 배율 반환
calcTriggerProb(nRanks): number                   // P(win_trigger) ≈ 0.0457
computeLightningRTP(nRanks): number               // ≈ 0.1234 (12.34%)

// holdAdvisor.ts — 최적 홀드 계산
calcOptimalHold(cards, deck, M): HoldAdvice       // 8조합 완전 열거 → 최적 홀드 반환
evalMult(cards, M): number                        // 패 평가 → 배당 배수 반환
calcAllHoldDists(cards, deck, M): HoldDist[]      // 8조합 결과 분포

// rtpOptimizer.ts — RTP 최적화
computeStaticRTP(m: MultTable): number            // 정적 RTP (홀드 무시)
runOptimalSimWith(m: MultTable): SimDetail         // 최적 플레이 RTP 시뮬레이션
optimizeMultipliers(targetRTP): OptimizeResult     // 이진 탐색으로 목표 RTP 배당 역산

// outcomeCalc.ts — 결과 확률 계산
computeOutcomeProbabilities(M): OutcomeCalcResult  // C(52,3) 전수 열거
classify(mult: number): OutcomeResult              // 배수 → 결과 분류

// ThreeCardPoker.tsx — 메인 게임 로직
evaluateHand(cards, M): HandResult                 // 3장 패 → 족보 + 배당 반환
startDeal(): void                                  // 딜 시작 (카드 생성, 타이머 시작)
doReveal(): void                                   // HOLD → Lightning 전환
doActualReveal(): void                             // 카드 교체 + 결과 계산
enterBetting(): void                               // 다음 라운드 BET 초기화
```

### 구 설계 문서 불일치 안내

프로젝트에 포함된 `GAME_DESIGN.md`와 `docs/MATHEMATICAL_MODEL.md`는 이전 "Water Pour" / "Overflow Game" 컨셉 (RTP 96.5%, 레벨 기반 생존 확률) 기반 설계 문서로, 현재 구현된 **GRAND TABLE · 3 Card Poker**와는 전혀 다른 게임이다. `lib/gameLogic.ts` 역시 동일한 Water Pour 게임용 코드이며 현재 게임에서 사용되지 않는다.

현재 게임의 정확한 수학 모델과 설계는 `gameConfig.ts`, `lightningSystem.ts`, `Spec.tsx`를 기준으로 한다.
