# Kickru 기술 아키텍처 분석

## 1. 확인되는 제품 구조

Kickru 공개 가이드에서 확인되는 핵심은 다음과 같다.

- 게임 진행은 현재 브라우저에 자동 저장
- 전체 커리어가 서버에 자동 백업되지는 않음
- 저장 파일 내보내기/가져오기 제공
- 은퇴 후 랭킹은 서버에 등록
- 상위권 기록은 서버 검증을 거침
- 2026-10-12 패치에는 "서버가 커리어를 처음부터 다시 계산해 검증"한다고 명시

따라서 서비스 성격은 명백히 **Local-first Game + Server-backed Ranking**에 가깝다.

---

# 2. 권장 전체 아키텍처

```text
┌──────────────── Browser ────────────────┐
│                                        │
│ UI                                     │
│  ↓                                     │
│ Game Application Layer                 │
│  ↓                                     │
│ Deterministic Simulation Core          │
│  ├─ season                             │
│  ├─ growth                             │
│  ├─ transfer                           │
│  ├─ contract                           │
│  ├─ event                              │
│  └─ narrative                          │
│                                        │
│ Local Persistence                      │
│  ├─ IndexedDB                          │
│  ├─ snapshots                          │
│  └─ export/import                      │
└────────────────┬───────────────────────┘
                 │ retirement submission
                 ▼
┌──────────────── Server ─────────────────┐
│ API validation                         │
│ Replay / verification                  │
│ Ranking DB                             │
│ moderation                             │
└────────────────────────────────────────┘
```

---

# 3. 프론트엔드 권장 스택

Kickru의 실제 프레임워크를 공개 자료만으로 확정할 수는 없다.

새로 구현한다면 다음 정도면 충분하다.

```text
Next.js or Vite + React
TypeScript
Zustand / Redux Toolkit (필요한 경우)
IndexedDB (Dexie 등)
Zod
Vitest
Playwright
```

이 장르에서 핵심은 SSR이 아니라 simulation core이므로 Next.js가 반드시 필요한 것은 아니다.

단순 게임이라면:

```
Vite + React + TypeScript
```

가 더 가볍다.

랭킹·공략·SEO 페이지까지 한 프로젝트로 통합한다면 Next.js가 편리하다.

---

# 4. Simulation Core를 UI와 분리해야 하는 이유

나쁜 구조:

```tsx
<button onClick={() => {
  player.ovr += Math.random() * 3;
  if (...) setTransfer(...)
}}>
```

좋은 구조:

```ts
const result = advanceSeason(state, command, rules);
dispatch({ type: "SEASON_ADVANCED", result });
```

이렇게 분리하면 동일 simulation core를:

- 브라우저
- 서버 검증
- 테스트
- 밸런스 시뮬레이터

에서 공통으로 사용할 수 있다.

이것이 이 장르의 가장 중요한 기술 결정이다.

---

# 5. Deterministic Simulation

Kickru의 상위 랭킹을 서버가 "커리어 처음부터 다시 계산"한다는 공개 설명은 매우 중요하다.

정확한 내부 구현은 공개되지 않았지만, 비슷한 방식을 만들려면 최소한 결과를 재현할 수 있어야 한다.

## 권장 구조

커리어 시작:

```ts
careerSeed = cryptoRandomSeed();
```

플레이 중 저장:

```ts
ReplayLog = {
  seed,
  initialConfig,
  commands: [
    {season: 1, type: "SELECT_TRAINING", value: "shooting"},
    {season: 3, type: "ACCEPT_TRANSFER", offerId: "..."}
  ]
}
```

서버:

```ts
const state = createCareer(initialConfig, seed);

for (const command of commands) {
  state = applyCommand(state, command);
}

assert(state.retirementSummary === submittedSummary);
```

이 방식이면 클라이언트가 최종 OVR을 99로 수정해 보내도 서버 재계산 결과와 달라진다.

---

# 6. RNG 설계

랜덤 시뮬레이션을 재현하려면 난수 호출 순서가 안정적이어야 한다.

그러나 한 기능을 추가하면 random 호출 횟수가 바뀌어 과거 세이브가 전혀 다른 결과를 낼 수 있다.

이를 피하려면 subsystem seed를 분리한다.

```text
careerSeed
 ├─ matchSeed(season, fixture)
 ├─ growthSeed(season)
 ├─ transferSeed(season)
 ├─ eventSeed(season)
 └─ injurySeed(season)
```

예:

```ts
deriveSeed(careerSeed, "transfer", seasonId)
```

이렇게 하면 경기 로직을 수정해도 이적 RNG가 같이 밀리지 않는다.

---

# 7. 저장 구조

공개 가이드상 브라우저 저장을 사용한다.

규모가 커질수록 localStorage보다는 IndexedDB가 적합하다.

## 권장

```text
IndexedDB
├─ activeCareer
├─ retiredCareerSummaries
├─ settings
└─ replayLogs
```

### activeCareer

전체 상태 저장.

### retiredCareerSummaries

은퇴 커리어 전체 state를 그대로 보관하지 않는다.

Kickru 패치에서 브라우저 저장 공간 부족 문제를 해결하며 은퇴 커리어 보관 용량을 크게 줄였다고 밝힌다.

따라서 은퇴 후에는:

```text
full state
→ compact summary
```

로 변환하는 것이 적절하다.

예:

```ts
RetiredCareerSummary {
  playerIdentity
  seasonSummaries
  awards
  majorMoments
  finalStats
}
```

임시 이벤트 큐, 월별 중간 상태, fixture simulation cache 등은 버린다.

---

# 8. Save Schema Version

실제 패치에서 준프로 계약 후 산하 고교로 이동한 특정 커리어가 호환되지 않는 저장으로 판정됐고, 이후 기존 파일도 다시 열 수 있게 수정됐다.

이런 문제가 생기는 가장 흔한 이유:

- union type 변경
- enum 값 추가/삭제
- required field 추가
- 이전 상태 조합을 validator가 허용하지 않음

따라서 저장 데이터에는 반드시 버전이 필요하다.

```ts
SaveFile {
  schemaVersion: 8,
  rulesVersion: "2026.11.13",
  data: CareerState
}
```

로드:

```text
v4
→ migrateV4toV5
→ migrateV5toV6
→ migrateV6toV7
→ migrateV7toV8
→ validate
```

절대 최신 Zod schema로 바로 parse만 하면 안 된다.

---

# 9. Schema와 Rules를 분리

매우 중요하다.

```
schemaVersion
rulesVersion
```

은 다른 개념이다.

### schemaVersion

파일 구조가 바뀜.

### rulesVersion

성장, 연봉, 이적 등 게임 규칙이 바뀜.

Kickru 패치에는:

- 진행 중 커리어에도 즉시 적용
- 다음 시즌부터 적용
- 새 커리어부터 적용

이 세 방식이 모두 등장한다.

따라서 시스템적으로도 rule activation policy가 필요하다.

```ts
RuleChange {
  id,
  effectiveDate,
  policy:
    | "ALL_CAREERS"
    | "NEXT_SEASON"
    | "NEW_CAREERS_ONLY"
}
```

---

# 10. Calendar Engine

춘추제/추춘제 차이에서 "한 해를 통째로 더 뛰는 문제"가 실제 수정됐다.

원인은 season number 중심 설계에서 발생하기 쉽다.

나쁜 모델:

```ts
season: 2031
```

좋은 모델:

```ts
CompetitionSeason {
  id: "EPL-2031-32",
  start: 2031-08,
  end: 2032-05
}
```

```ts
PlayerTimeline {
  currentDate: 2032-01
}
```

이적:

```text
K League 2031 ends
→ current date 2031-12
→ Europe 2031/32 is already in progress
→ join remaining fixtures
```

즉 season transition이 아니라 timeline transition으로 구현한다.

---

# 11. League Rule Engine

리그마다:

- 경기 수
- 스플릿
- 승점 반감
- 승강 수
- 승강 플레이오프
- B팀 제한
- 국제대회 참가 자격

이 다르다.

따라서:

```ts
LeagueRules {
  scheduleGenerator
  postseason
  qualification
  promotion
  relegation
  specialEligibility
}
```

를 데이터/전략 패턴으로 분리한다.

규정 변경도:

```text
KLEAGUE_2026
KLEAGUE_2027
```

처럼 연도별 rule set으로 유지해야 과거 커리어 재현이 가능하다.

---

# 12. Ranking Verification

클라이언트에서 돌아가는 게임의 랭킹에는 항상 조작 문제가 있다.

Kickru가 공개한 대응은 단계적으로 강화됐다.

1. 랭킹 운영 정책
2. 자동화/도배 기록 삭제
3. 상위 기록 서버 검증
4. 검증 불가능한 기록 수동 확인
5. 상위권 커리어를 처음부터 다시 계산

유사 구현에서는 다음 payload가 필요하다.

```ts
RankingSubmission {
  saveSchemaVersion;
  rulesVersion;
  seed;
  initialConfig;
  commands;
  claimedSummary;
  clientBuild;
}
```

서버는 claimedSummary를 신뢰하지 않는다.

---

# 13. Replay Log vs Full Save

서버 검증을 위해 full save를 보내는 것보다 command log가 좋다.

장점:

- 조작 탐지
- 작은 payload
- 재현성
- 버그 분석
- 사용자의 선택 흐름 분석

단점:

- simulation core가 결정론적이어야 함
- 오래된 rulesVersion 유지 필요

절충안:

```
periodic snapshot
+ command log after snapshot
```

---

# 14. Event Sourcing 접근

Kickru류 커리어 게임은 event sourcing과 궁합이 좋다.

예:

```text
CareerStarted
SeasonAdvanced
TrainingChanged
TransferAccepted
InjuryOccurred
AwardWon
Retired
```

현재 state는 이벤트를 적용해 얻는다.

전체 event sourcing이 부담되면 최소한 "사용자 선택 command log"만 유지해도 충분하다.

---

# 15. 저장 압축

은퇴 기록을 많이 저장하면 브라우저 quota 문제가 생긴다.

권장:

- 중복 club 객체 대신 clubId만 저장
- 이미지 저장 금지
- static world data는 save에 넣지 않음
- derived 값 저장 최소화
- 은퇴 후 상세 fixture 제거
- 필요하면 JSON 압축

절대:

```
모든 시즌의 모든 경기 객체
+ 구단 전체 데이터
+ 이미지 base64
```

를 세이브에 넣지 않는다.

---

# 16. 성능

경기 수가 늘어도 시즌 진행 버튼 후 체감이 느려지면 안 된다.

### 일반 시즌

순수 계산만 수행.

### 중요한 경기

결과를 먼저 계산한 후 UI 애니메이션으로 재생.

Kickru가 결승 중계에서 "실제로 치른 결승의 스코어와 승부차기 결과를 그대로" 보여준다고 설명하는 점도 이 패턴과 잘 맞는다.

```text
simulate result
→ store event log
→ animate stored result
```

중계가 실제 simulation을 실시간으로 결정하는 구조보다 안정적이다.

---

# 17. Asset Preloading

Kickru 패치에는 우승 세리머니가 늦게 뜨거나 끊기는 문제를 해결하기 위해 화면 전 미리 불러오고 파일을 가볍게 했다고 명시돼 있다.

권장:

```ts
await preloadAssets(nextScene.assets);
showScene();
```

이미지는 WebP/AVIF, 크기별 variant를 사용하고 애니메이션은 필요한 프레임만 로딩한다.

---

# 18. Server Cost Control

이 구조는 서버 비용을 크게 줄일 수 있다.

서버가 담당하는 것:

- 랭킹 등록
- 검증
- 공개 랭킹 조회
- 피드백/운영

브라우저가 담당하는 것:

- 전체 커리어 진행
- 시즌 시뮬레이션
- 저장
- 대부분 UI

즉 MAU가 커져도 "게임 한 시즌마다 API"를 호출하지 않는다.

---

# 19. API 예시

```text
POST /api/ranking/submit
GET  /api/ranking?date=2026-11-13
GET  /api/ranking/best11
POST /api/feedback
```

랭킹 submit은 즉시 공개하지 않고:

```
received
→ validation queue
→ replay
→ score validation
→ moderation if needed
→ publish
```

로 처리할 수 있다.

---

# 20. 테스트 구조

```
tests/
├─ simulation/
├─ growth/
├─ transfer/
├─ contract/
├─ calendars/
├─ league-rules/
├─ save-migrations/
├─ ranking-replay/
└─ regression/
```

특히 모든 실제 버그는 regression fixture로 남긴다.

예:

```
save/quasi-pro-affiliate-school-v7.json
calendar/kleague-to-europe-midseason.json
event/retirement-plus-turningpoint.json
```

버그를 고친 뒤 이 fixture가 영구적으로 통과하도록 해야 한다.

---

# 21. 가장 중요한 기술적 결론

Kickru류 게임을 오래 운영할 때 가장 어려운 부분은 React 컴포넌트가 아니다.

난이도 순서는 오히려:

1. 시간/시즌 모델
2. 규칙 버전 관리
3. 세이브 호환성
4. 결정론적 시뮬레이션
5. 예외 규칙 관리
6. 밸런스
7. 랭킹 검증
8. UI

에 가깝다.

초기부터 simulation core를 독립 모듈로 만들지 않으면 업데이트가 많아질수록 기술 부채가 급격히 커진다.
