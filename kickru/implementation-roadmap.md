# Kickru류 웹게임 실제 개발 로드맵

## 목표

처음부터 실제 축구 전체를 구현하지 않는다.

가장 먼저 확인해야 할 것은:

> "시즌 버튼을 누르고 선택을 하며 한 선수의 인생이 달라지는 과정 자체가 재미있는가?"

이다.

---

# Phase 0. Simulation Prototype

UI 없이 TypeScript만으로 만든다.

필수:

- 선수 상태
- 나이
- OVR
- 잠재력
- 한 시즌 성장
- 출전 시간
- 은퇴

CLI 또는 테스트:

```ts
for (let season = 0; season < 20; season++) {
  state = advanceSeason(state);
}
```

1000개 커리어를 돌려 성장 분포가 정상인지 본다.

---

# Phase 1. 최소 플레이 가능 버전

## 기능

- 선수 생성
- 한 포지션
- 가상 리그 1개
- 가상 구단 10개
- 시즌 진행
- 성장
- 이적 제안
- 은퇴
- 커리어 요약

아직 필요 없는 것:

- 실제 구단
- 국가대표
- 컵대회
- 상점
- 랭킹
- 애니메이션
- 상세 경기 중계

이 단계에서 "한 번 더 하고 싶다"가 안 나오면 콘텐츠를 늘려도 해결되지 않는다.

---

# Phase 2. Choice Engine

추가:

- 집중 육성
- 이적
- 임대
- 계약
- 주장
- 포지션 전환
- 기자회견

이때부터 event engine을 데이터 기반으로 만든다.

---

# Phase 3. Career Memory

추가:

- 데뷔
- 첫 골
- 첫 우승
- 대형 이적
- 라이벌 이적
- 부상
- 복귀
- 수상

모든 중요한 사건을 `CareerMoment`로 저장한다.

은퇴:

```
CareerMoment[]
→ select important moments
→ narrative templates
```

---

# Phase 4. 실제 리그 구조

이제 리그 1개씩 추가한다.

순서 권장:

1. 단순 홈앤어웨이 리그
2. 컵
3. 승강
4. 플레이오프
5. 스플릿
6. 춘추제/추춘제
7. 대륙 대회

한 번에 전세계 리그를 넣으면 디버깅이 불가능해진다.

---

# Phase 5. Save System

처음부터 필요하지만 이 단계에서 강화한다.

- IndexedDB
- autosave
- export/import
- schema migration
- recovery
- retired career compression

## 반드시 테스트할 것

- 새 버전에서 6개월 전 save 열기
- 잘못된 JSON
- 일부 필드 없음
- quota 부족
- 여러 커리어
- 같은 이름 선수

---

# Phase 6. Replay / Determinism

랭킹 전 반드시 구현한다.

- career seed
- command log
- deterministic RNG
- simulation replay

검증 테스트:

```
client final state
===
replayed final state
```

1000회 자동 검사.

---

# Phase 7. Ranking

처음부터 실시간 멀티플레이를 만들지 않는다.

은퇴 결과만 전송한다.

```
POST ranking submission
→ server replay
→ validate
→ store summary
```

이렇게 하면 서버 비용도 낮다.

---

# Phase 8. Live Ops

콘텐츠를 코드 변경 없이 추가할 수 있어야 한다.

데이터 파일:

```
data/
├─ clubs
├─ leagues
├─ competitions
├─ events
├─ fan-reactions
├─ items
└─ awards
```

---

# 1. 추천 코드 구조

```
src/
├─ app/
├─ ui/
│
├─ game/
│  ├─ engine/
│  │  ├─ advanceSeason.ts
│  │  ├─ rng.ts
│  │  └─ commands.ts
│  │
│  ├─ systems/
│  │  ├─ growth/
│  │  ├─ match/
│  │  ├─ transfer/
│  │  ├─ contract/
│  │  ├─ aging/
│  │  ├─ awards/
│  │  ├─ events/
│  │  └─ narrative/
│  │
│  ├─ rules/
│  └─ types/
│
├─ data/
├─ persistence/
├─ replay/
└─ ranking/
```

---

# 2. 테스트 우선순위

## Unit

```
calculateOVR
calculateGrowth
calculateSalary
calculateTransferInterest
awardScore
aging
```

## Property tests

예:

- stats는 0~99를 넘지 않는다.
- contract end < start가 될 수 없다.
- 한 시즌 경기 수가 음수가 될 수 없다.
- 은퇴 후 다시 계약할 수 없다.

## Regression

실제 발견된 버그를 저장한다.

```
regression/
├─ cross-calendar-transfer.test.ts
├─ event-conflict.test.ts
├─ save-migration.test.ts
├─ salary-inflation.test.ts
└─ reroll-replay.test.ts
```

---

# 3. Monte Carlo 밸런싱

이 장르에서는 사람이 10번 플레이해서 밸런스를 판단하면 부족하다.

```ts
for (let i = 0; i < 100_000; i++) {
  simulateCareer(randomSeed(i));
}
```

분석:

- 평균 은퇴 OVR
- 90+ OVR 비율
- 발롱도르 수상률
- 조기 은퇴율
- 빅클럽 진입률
- 35세 이상 생존률
- 포지션별 골/도움
- 리그별 우승 분포

---

# 4. Rare Event 설계

재미는 평균이 아니라 tail에서 나온다.

예:

```
70% 평범한 커리어
20% 좋은 커리어
8% 매우 특별한 커리어
2% 전설적인 커리어
```

단 수치는 예시다.

대기만성, 대형 부상, 깜짝 우승 같은 사건이 rare tail을 만든다.

---

# 5. 밸런스 디버그 화면

개발 버전에서 반드시 보여준다.

```
player OVR
hidden potential
growth profile
aging profile
club tier
transfer interest scores
award score breakdown
event eligibility
RNG seed
```

사용자에게는 숨기더라도 개발자는 봐야 한다.

---

# 6. 규칙 변경 배포 절차

새 규칙을 바로 코드에 덮어쓰지 않는다.

1. 변경 정의
2. rulesVersion 증가
3. 기존 커리어 적용 정책 결정
4. migration 여부 결정
5. Monte Carlo 비교
6. regression test
7. 배포
8. telemetry/피드백 확인

---

# 7. 추천 MVP 데이터 규모

첫 버전:

- 국가 1~2
- 리그 2
- 구단 20
- 포지션 4
- 이벤트 20
- 수상 5
- 아이템 0

게임이 재미있으면 이후:

- 포지션 10
- 리그 10+
- 국가대표
- 컵
- 대륙대회
- 실제 규칙 수준의 예외

로 확장한다.

---

# 8. UI 개발 순서

1. 선수 생성
2. 홈/현재 상태
3. 시즌 결과
4. 선택 modal
5. 이적/계약
6. 커리어 기록
7. 은퇴
8. 랭킹
9. 중요 경기 중계

중계 화면을 먼저 만들 필요가 없다.

---

# 9. 실제 개발에서 가장 위험한 것

## 너무 많은 실제 리그를 일찍 넣기

예외 규칙 폭발.

## 상태를 React state에만 보관

저장/재현/검증 불가능.

## Math.random 직접 사용

재현 불가능.

## save schema version 없음

업데이트 순간 오래된 세이브 파손.

## 현재 state만 저장

커리어 이야기와 서버 검증 어려움.

## 수상/이적 공식을 하나의 거대한 함수로 작성

밸런스 조절 불가능.

---

# 10. 첫 개발 목표

완성형 목표가 아니라 아래 vertical slice를 먼저 만든다.

```
16세 선수 생성
↓
5시즌 진행
↓
2번의 성장 선택
↓
1번의 이적 선택
↓
능력치와 출전 시간 차이
↓
커리어 요약
```

이 10분짜리 플레이가 재미있으면 전체 프로젝트를 확장할 가치가 있다.

---

# 11. 최종 개발 전략

Kickru류 게임은 "화면을 많이 만드는 프로젝트"가 아니다.

본질은:

```
Simulation Engine
+ Rule Data
+ Career History
+ Replayability
```

다.

게임이 커질수록 새 기능을 추가하는 속도보다 **새 규칙이 기존 규칙과 충돌하지 않는 구조**가 더 중요해진다.
