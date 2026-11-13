# Kickru 게임 시스템 구현 분석

## 1. 목표

Kickru와 유사한 "선수 생애 시뮬레이션"을 새로 만든다고 가정하고, 실제 코드 설계가 가능한 수준까지 시스템을 분해한다.

---

# 2. 전체 게임 상태

게임의 중심은 화면이 아니라 하나의 `CareerState`다.

```ts
type CareerState = {
  schemaVersion: number;
  rulesVersion: number;

  player: PlayerState;
  club: ClubState;
  season: SeasonState;
  world: WorldState;

  contract: ContractState | null;
  nationalTeam: NationalTeamState | null;

  finances: FinanceState;
  inventory: InventoryState;

  flags: CareerFlags;
  history: CareerHistory;
  pendingEvents: CareerEvent[];

  rng: RandomState;
};
```

핵심 원칙은 **모든 결과가 CareerState를 입력으로 받아 새로운 CareerState를 만드는 순수 로직에 가깝게 구성하는 것**이다.

```text
state(t)
  + command
  + random seed
  + rules
    ↓
simulation
    ↓
state(t+1)
```

UI에서 직접 능력치를 바꾸면 이후 검증·세이브 마이그레이션·테스트가 어려워진다.

---

# 3. 선수 생성

공개 가이드 기준 Kickru는 시작 후보가 모두 OVR 45, 표시되는 6개 능력치 합계 240이며 분포가 다르다.

이 구조는 "초기 총량은 동일하지만 스타일만 다르게" 만드는 방식이다.

## 추천 생성 로직

```ts
const TOTAL_STAT_POINTS = 240;

function generateCandidate(
  position: Position,
  archetype: Archetype,
  seed: number
): PlayerCandidate
```

예:

```text
빠른 윙어
Speed +12
Dribble +8
Shooting +2
Passing  0
Defense -10
Physical -12
```

단, 최종 합은 항상 일정하게 정규화한다.

### 왜 좋은가

- 처음부터 "좋은 카드/나쁜 카드"가 생기지 않음
- 리세마라 부담 감소
- 선택이 강함이 아니라 플레이 스타일 선택이 됨

---

# 4. 숨은 성장 특성

Kickru는 공개적으로 숨은 성장 특성, 늦게 피는 선수, 개인별 노화 체질이 있다고 설명한다.

이런 시스템은 단순 `potential` 하나보다 다음처럼 나누는 것이 좋다.

```ts
type DevelopmentProfile = {
  potential: number;          // 성장 상한
  growthRate: number;         // 기본 성장 속도
  breakoutAge: number | null; // 대기만성형 개화 시점
  agingStartAge: number;      // 노화 시작
  agingRate: number;          // 노화 속도
  durability: number;         // 부상 내성
  consistency: number;        // 시즌 편차
};
```

## 중요한 설계

숨은 특성을 플레이어에게 직접 보여주지 않는다.

그러나 결과가 완전 랜덤처럼 보이면 불공정하게 느껴진다.

따라서:

```text
Hidden Trait
+ Observable Clue
= Mystery without unfairness
```

예를 들어 "회복이 빠르다", "훈련 반응이 좋다" 같은 간접 문구로 힌트를 줄 수 있다.

---

# 5. 성장 공식

공개 설명상 성장에는 나이, 출전 시간, 활약, 건강, 숨은 성장 특성이 영향을 준다.

권장 개념식:

```text
baseGrowth
× ageFactor
× playingTimeFactor
× performanceFactor
× healthFactor
× developmentProfile
× trainingFocus
× temporaryEffects
```

실제 구현은 곱셈만 쓰면 극단값이 생기기 쉽다.

따라서 코드에서는 가중합 후 clamp하는 편이 안전하다.

```ts
growthScore =
  base
  + ageBonus
  + minutesBonus
  + performanceBonus
  + trainingBonus
  + traitBonus
  - injuryPenalty;

growthScore = clamp(growthScore, MIN_GROWTH, MAX_GROWTH);
```

## 포텐 도달 방지

```ts
nextStat = Math.min(
  stat + delta,
  potentialAdjustedCap
);
```

출전이 적으면 `delta`를 크게 줄여 잠재력까지 자동 성장하지 않게 한다.

---

# 6. 출전 시간 모델

이 장르에서 가장 중요한 trade-off는:

```text
강한 팀의 벤치
vs
약한 팀의 주전
```

이다.

따라서 팀 선택은 OVR 차이만으로 결정하면 안 된다.

권장 모델:

```text
roleScore =
  playerOVR
  - squadPositionCompetition
  + reputation
  + managerTrust
  + recentForm
```

roleScore를 기반으로:

- 핵심 선수
- 주전
- 준주전
- 로테이션
- 후보

를 정한다.

이 값이 실제 시즌 출전 분량의 기대값이 된다.

---

# 7. 경기/시즌 시뮬레이션

Kickru의 장점은 한 시즌을 압축하는 것이다.

모든 경기를 실시간으로 계산하더라도 UI에서는 중요한 경기만 확대해서 보여주면 된다.

## 권장 구조

```text
Season
  ├─ league fixtures
  ├─ domestic cups
  ├─ continental cups
  ├─ national team
  └─ special playoffs
```

각 경기:

```ts
simulateMatch(match, worldState, seed)
→ MatchResult
```

선수 개인 결과:

```ts
simulatePlayerMatch(
  player,
  role,
  opponentStrength,
  tactics,
  fitness,
  seed
)
→ PlayerMatchResult
```

## 결과 생성 항목

- 출전 여부
- 선발/교체
- 출장 시간
- 골
- 도움
- 평점
- 클린시트
- 카드
- 부상
- MOM 등

---

# 8. 공격포인트 모델

Kickru 패치노트에는 슈팅·패스와 플레이 성향에 따라 골/도움 비중을 다르게 하고, 포지션별 공격포인트 패턴을 조정했다고 나온다.

따라서 단일 `goalChance = OVR` 식은 피한다.

예:

```text
ST goal weight
  shooting 0.45
  physical 0.20
  speed    0.15
  dribble  0.10
  teamAtk  0.10

CAM assist weight
  passing  0.45
  dribble  0.20
  vision   0.15
  teamAtk  0.20
```

실제 게임에서는 포지션별 coefficient table로 데이터화한다.

---

# 9. OVR 계산

공개 설명대로 OVR은 포지션별 능력치 가중평균이다.

```ts
const POSITION_WEIGHTS = {
  ST:  { speed:.15, shooting:.35, passing:.10, dribble:.15, defense:.05, physical:.20 },
  CAM: { speed:.10, shooting:.15, passing:.30, dribble:.25, defense:.05, physical:.15 },
  CB:  { speed:.10, shooting:.02, passing:.08, dribble:.05, defense:.45, physical:.30 }
};
```

```ts
function calculateOVR(stats, position) {
  const w = POSITION_WEIGHTS[position];
  return round(weightedAverage(stats, w));
}
```

포지션 전환 이벤트는 능력치를 바꾸지 않고 이 가중치만 바꾸어도 즉시 OVR 변화가 발생한다.

---

# 10. 이적 제안 엔진

Kickru 패치에서 반복적으로 수정된 부분 중 하나가 "비현실적인 이적 제안"이다.

이 문제는 확률 하나로 처리하면 계속 튀는 결과가 나온다.

## 권장 2단계 구조

### 1단계: 후보 구단 필터

```text
league eligibility
age range
registration rules
contract state
military status
loan status
club budget
club need by position
club tier distance
```

### 2단계: 관심 점수

```text
interest =
 playerAbility
 + recentPerformance
 + potentialValue
 + reputation
 + tacticalFit
 + clubNeed
 - transferDifficulty
 - agePenalty
```

그 후 softmax 또는 weighted random으로 제안 구단을 선택한다.

## 중요

"하부리그 → 세계 최고 구단" 같은 결과를 막으려면 단순 확률 감소가 아니라 **tier jump 제한**이 필요하다.

```ts
if (targetTier - currentTier > MAX_NORMAL_JUMP) {
  require(exceptionalSeason === true);
}
```

---

# 11. 계약 시스템

연봉은 무한 상승하면 안 된다.

Kickru 패치에서도 반복 인상으로 연봉이 끝없이 커지는 문제가 수정됐다.

권장 공식:

```text
salaryTarget =
  clubWageLevel
× roleMultiplier
× abilityMultiplier
× reputationMultiplier
× leagueMultiplier
× ageCurve
```

재계약은 기존 연봉에 단순 퍼센트를 계속 곱하지 말고 매번 `salaryTarget`을 새로 계산한다.

이렇게 해야 인플레이션이 누적되지 않는다.

---

# 12. 노화

노화는 일정 나이부터 매년 -1 같은 방식이면 너무 예측 가능하다.

Kickru처럼 개인별 차이를 두는 것이 좋다.

```ts
decline =
  ageBase
  * agingProfile
  * positionAgingCurve
  * injuryHistory
  * antiAgingEffects
  * yearlyNoise
```

포지션별 curve를 별도로 둔다.

예:

- 윙어: 속도 중심이므로 빠른 하락
- CB: 늦은 하락
- GK: 가장 늦은 하락

---

# 13. 이벤트 엔진

이 장르에서 이벤트는 if문 묶음으로 만들면 빠르게 유지보수 불가능해진다.

권장 데이터 구조:

```ts
type CareerEventDefinition = {
  id: string;
  priority: number;
  cooldown?: number;

  conditions: Condition[];
  exclusions?: Condition[];

  choices: EventChoice[];
};
```

예:

```json
{
  "id": "training_direction_change",
  "priority": 30,
  "cooldown": 3,
  "conditions": [
    {"type":"minAge","value":18},
    {"type":"maxOccurrences","value":3}
  ]
}
```

## 이벤트 중복 방지

Kickru 패치에서 은퇴 권고와 다른 전환점이 동시에 떠 충돌한 문제가 확인된다.

따라서 이벤트는:

```text
candidate events
→ filter
→ sort priority
→ conflict group
→ choose one
```

구조로 처리한다.

```ts
event.conflictGroup = "season_transition";
```

같은 그룹에서는 하나만 표시한다.

---

# 14. 기자회견 / Risk Choice

Kickru는 기자회견 선택 전에 성공 확률과 성공/실패 결과를 보여준다.

이 방식은 매우 좋은 선택 UX다.

```ts
Choice {
  probability: 0.75,
  successEffect: {...},
  failureEffect: {...}
}
```

안전한 답:

```
성공률 높음
보상 작음
리스크 작음
```

도발적 답:

```
성공률 낮음
보상 큼
리스크 큼
```

즉 expected value를 사용자가 판단할 수 있다.

---

# 15. 수상 시스템

발롱도르 같은 수상은 공격포인트 총합으로만 계산하면 포지션 편향이 생긴다.

권장 구조:

```text
seasonScore =
  normalizedPerformanceByPosition
  + trophyScore
  + majorCompetitionScore
  + individualAwardScore
  + decisiveContribution
```

## 포지션별 정규화

```text
ST → goal contribution
CM → rating + assists + influence
CB → rating + defensive record
GK → rating + clean sheets + saves
```

최종 점수만 공통 scale로 변환한다.

---

# 16. 시즌 달력

Kickru가 실제로 겪은 중요한 문제다.

춘추제와 추춘제를 단순히 "seasonYear" 하나로 표현하면 이적 시 한 시즌을 통째로 더 뛰거나 누락할 수 있다.

따라서:

```ts
type CompetitionSeason = {
  startDate: GameDate;
  endDate: GameDate;
  registrationWindows: DateRange[];
}
```

실제 게임 시간도:

```ts
GameDate { year, month, phase }
```

처럼 가져가는 편이 안전하다.

이적은 "다음 시즌"이 아니라 **실제 합류 날짜**를 계산해야 한다.

---

# 17. 승강/스플릿/플레이오프

리그마다 규칙이 달라 generic league 하나로 처리하면 금방 막힌다.

추천:

```ts
LeagueFormat {
  regularRounds
  splitRule?
  pointHalving?
  promotionRule
  relegationRule
  playoffs[]
}
```

예:

```text
K League
regular season
→ ranking
→ promotion/relegation playoff
→ result
```

그리고 B팀, 상무 등 예외 규칙은 club metadata와 eligibility rule로 분리한다.

---

# 18. 커리어 히스토리

시즌 결과 화면용 데이터와 "인생 이야기"용 데이터를 구분해야 한다.

```ts
CareerHistory {
  seasons: SeasonSummary[];
  moments: CareerMoment[];
}
```

```ts
CareerMoment {
  type:
    | "debut"
    | "first_goal"
    | "transfer"
    | "rival_transfer"
    | "injury"
    | "comeback"
    | "final_revenge"
    | "ballon_dor"
    | "retirement";

  seasonId: string;
  clubId?: string;
  relatedClubId?: string;
  importance: number;
  payload: Record<string, unknown>;
}
```

은퇴 시 importance가 높은 순간을 골라 템플릿으로 서사를 만든다.

---

# 19. 랜덤성

이 장르에서는 모든 `Math.random()` 호출을 흩뿌리면 안 된다.

권장:

```ts
rng.next("growth");
rng.next("injury");
rng.next("transfer");
rng.next("match");
```

혹은 subsystem별 seed stream을 만든다.

이유:

- 버그 재현
- 저장 복구
- 랭킹 검증
- 자동 테스트
- 밸런스 시뮬레이션

이 모두가 쉬워진다.

---

# 20. 핵심 구현 원칙

가장 중요한 것은 다음 네 가지다.

1. **데이터와 규칙을 UI에서 분리**
2. **랜덤을 seed 기반으로 관리**
3. **모든 시즌 결과를 history에 남김**
4. **예외 규칙을 if문으로 흩뿌리지 않고 rule 데이터로 관리**

Kickru류 게임은 화면보다 규칙의 수가 훨씬 빠르게 늘어난다. 초기부터 이 네 가지를 지키지 않으면 콘텐츠 추가 속도가 곧 버그 증가 속도가 된다.
