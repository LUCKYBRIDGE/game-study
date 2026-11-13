# Kickru 실제 문제와 해결 방식 분석

이 문서는 Kickru 패치노트에서 **실제로 드러난 문제**를 개발 관점에서 역분석한다.

각 항목은 다음 형식을 사용한다.

- 확인된 문제
- Kickru가 공개한 해결 방향
- 가능한 기술 원인
- 새 게임을 만든다면 권장 구현

내부 소스코드를 본 것이 아니므로 "가능한 기술 원인"은 추론이다.

---

# 1. 춘추제 ↔ 추춘제 이적 시 한 해를 더 뛰는 문제

## 확인된 문제

고교·대학에서 시즌제가 다른 해외 구단으로 이동할 때 춘추제/추춘제 차이 때문에 한 해를 통째로 더 뛰는 문제가 있었다.

## 공개된 해결

시즌제가 다른 해외 구단으로 가면 **진행 중인 시즌 중간에 합류**하도록 변경했다.

## 원인 추정

```
careerYear++
→ targetLeague.nextSeason
```

처럼 "연도"를 기준으로 이동했을 가능성이 있다.

하지만:

- K리그: 2028
- EPL: 2028/29

은 같은 시간축에서 겹친다.

## 권장 해결

모든 리그를 absolute timeline에 올린다.

```ts
currentDate = { year: 2028, month: 12 };
targetSeason = findSeasonContainingDate(targetLeague, currentDate);
```

---

# 2. 준프로 계약 후 저장 파일이 손상으로 판정

## 확인된 문제

준프로 계약 후 산하 고교로 옮긴 커리어가 "호환되지 않거나 손상된 저장 파일"로 열리지 않았다.

패치 후 이미 열리지 않던 저장도 다시 열리게 됐다.

## 원인 추정

선수 상태가:

```
school player
+ pro contract
+ affiliate school
```

이라는 기존 validator가 예상하지 못한 조합이 됐을 가능성이 높다.

## 권장 해결

### 잘못된 방식

```ts
if (player.isPro && player.schoolId) throw InvalidSave;
```

### 좋은 방식

도메인 상태 자체를 명시적으로 표현한다.

```ts
careerStage:
  | "SCHOOL_AMATEUR"
  | "SCHOOL_QUASI_PRO"
  | "PRO"
```

그리고 옛 저장은 migration으로 변환한다.

---

# 3. 브라우저 저장 공간 부족

## 확인된 문제

"브라우저 저장 공간이 부족합니다" 오류로 저장되지 않는 문제가 있었고, 은퇴 커리어 보관 용량을 크게 줄였다.

## 원인

브라우저 storage quota는 무한하지 않다.

은퇴 커리어마다 전체 월드 상태와 모든 상세 기록을 저장하면 빠르게 커진다.

## 해결 전략

```
active career = full state
retired career = compressed summary
```

### 버려도 되는 데이터

- 시즌 중 임시 UI 상태
- 경기 계산 cache
- pending events
- 전체 club object 복사본
- 반복 가능한 derived 값

---

# 4. 준프로 기록이 전반기/후반기로 두 줄 표시

## 확인된 문제

준프로 K리그 기록이 전반기/후반기로 나뉘어 두 줄로 나오던 문제가 수정됐다.

## 원인 추정

season segment를 record identity로 사용했을 가능성이 있다.

## 권장

```ts
SeasonRecordKey =
  playerId
  + competitionId
  + competitionSeasonId
```

전반/후반은 필요할 때만 segment metadata로 둔다.

---

# 5. 팬 반응이 상황과 맞지 않는 문제

## 확인된 문제

프리미어리그 선수에게 K리그 댓글이 뜨는 등 맥락에 안 맞는 팬 반응이 있었고, 리그·구단·학교 소속에 맞는 반응만 나오도록 수정됐다.

## 원인

댓글 pool에서 텍스트만 랜덤 선택하면 이런 문제가 발생한다.

## 권장

모든 reaction에 조건을 붙인다.

```ts
FanReaction {
  text;
  conditions: {
    leagueIds?;
    clubIds?;
    eventTypes?;
    outcome?;
    rivalOnly?;
    minPerformance?;
  }
}
```

```text
candidate reactions
→ context filter
→ weight
→ random
```

---

# 6. 기자회견 결과와 팬 반응 불일치

## 확인된 문제

답변이 성공했는데도 맥락 없이 욕만 달리는 문제가 수정됐다.

## 핵심 원인

"선택"과 "반응"을 별도 랜덤 시스템으로 만들면 결과가 충돌한다.

## 권장

반응은 choice가 아니라 **resolved outcome**을 참조해야 한다.

```text
choice
→ roll
→ outcome
→ article
→ fan reaction
```

절대로:

```
choice
→ article random
→ reaction random
```

으로 따로 돌리지 않는다.

---

# 7. 이적 제안이 비현실적

## 확인된 문제

- 하부리그에서 갑자기 빅클럽
- 전성기 유럽 유망주에게 MLS 제안 과다
- 주전 젊은 선수에게 훨씬 약한 구단 제안
- 라이벌 이적 제안 과다

등이 여러 패치에서 수정됐다.

## 중요한 교훈

단일 "선수 OVR → 랜덤 구단" 모델은 실패한다.

## 권장 4요소

```
eligibility
+ club tier distance
+ player performance
+ career context
```

라이벌, 연령, 지역, 리그, 계약 상황은 확률 보정보다 **후보 필터**로 먼저 제거하는 것이 안정적이다.

---

# 8. 연봉 무한 인플레이션

## 확인된 문제

연봉 인상 요청과 조기 재계약으로 연봉이 계속 불어났다.

## 원인

```text
newSalary = currentSalary × 1.2
```

같은 누적 방식.

## 공개된 해결 방향

구단 수준과 선수 기량에 맞는 범위 안에서만 오르고, 이미 최고 수준이면 거절. 30대 이후 인상 폭 감소.

## 권장 공식

현재 연봉이 아니라 target market value에서 재계산한다.

```text
salary = f(club, league, ability, role, age, reputation)
```

---

# 9. 발롱도르 평가 왜곡

## 확인된 문제

- 트로피 없이 기록만 좋으면 지나치게 높은 순위
- 대표팀 우승 반영 누락
- 특정 포지션 편향
- 5대리그 밖 UCL 우승 가치 부족

등이 수정됐다.

## 교훈

수상 시스템은 단일 score가 시간이 지나며 계속 보정될 가능성이 높다.

따라서 coefficient를 코드 상수로 흩뿌리지 말고 데이터화해야 한다.

```ts
AwardRule {
  performanceWeight,
  trophyWeights,
  competitionWeights,
  positionNormalization,
  participationThreshold
}
```

---

# 10. 주장 제안이 계속 반복

## 확인된 문제

이미 주장을 맡고 있는 구단에서도 주장 제안이 반복됐다.

## 원인

이벤트 발생 조건에서 기존 상태를 exclusion으로 보지 않음.

## 해결

```ts
conditions: [
  notAlreadyCaptain,
  noCooldown,
  clubTenure >= 2
]
```

모든 event는 `conditions`뿐 아니라 `exclusions`를 가져야 한다.

---

# 11. 은퇴 권고와 다른 전환점 충돌

## 확인된 문제

은퇴 권고와 전환점이 함께 떴을 때 전환점을 누를 수 없는 문제가 있었다.

## 원인

둘 다 modal/UI lock을 소유했을 가능성이 높다.

## 권장 해결

이벤트 우선순위 큐:

```text
RETIREMENT_DECISION priority 100
CONTRACT_DECISION   priority 80
CAREER_TURNING      priority 50
FLAVOR_EVENT        priority 10
```

같은 phase에서는 한 이벤트만 active.

---

# 12. 이적 전 리그가 순위 화면에 표시

## 확인된 문제

이적 후에도 리그 순위에 이적 전 리그가 나오는 문제가 수정됐다.

## 원인

UI가 "현재 player.club"과 "시즌 기록의 club"을 혼합.

## 권장

시즌 결과 화면은 mutable current state를 참조하지 않고 **SeasonSnapshot**을 사용한다.

```ts
SeasonSummary {
  clubIdAtSeasonEnd;
  leagueId;
}
```

---

# 13. 같은 이름의 커리어 카드 이미지 덮어쓰기

## 확인된 문제

같은 이름을 가진 선수의 커리어 카드 이미지가 덮어써졌다.

## 원인

filename key가 playerName.

## 해결

항상 stable ID 사용.

```
career-{careerId}.webp
```

표시명과 storage key를 절대 동일시하지 않는다.

---

# 14. 우승 세리머니가 끊기거나 늦게 표시

## 확인된 문제

패치노트에서 화면 전에 미리 불러오고 파일을 가볍게 했다고 설명.

## 실제 해결 방향

- preload
- asset optimization

## 권장

```ts
await Promise.all(scene.assets.map(preload));
transitionTo(scene);
```

네트워크 상태가 나쁘면 fallback still image를 사용할 수도 있다.

---

# 15. 승강 플레이오프 출전이 시즌 기록에서 누락될 수 있는 문제

Kickru는 승강 플레이오프를 실제 경기로 치르고 출전 기록에 포함한다고 명시했다.

이런 특별 경기는 "정규리그 이후 이벤트"로만 처리하면 개인 기록 집계에서 빠질 수 있다.

## 권장

모든 경기는 동일 `MatchResult` 파이프라인으로 처리하고 competition phase만 다르게 한다.

```
REGULAR
SPLIT
PROMOTION_PLAYOFF
RELEGATION_PLAYOFF
FINAL
```

---

# 16. 기존 커리어와 신규 규칙 호환

패치노트를 보면 적용 정책이 자주 다르다.

- 진행 중 커리어 즉시 적용
- 다음 시즌부터 적용
- 새 커리어부터 적용
- 기존 랭킹은 유지
- 과거 수상 결과는 재계산하지 않음

## 핵심 문제

라이브 서비스 게임에서는 규칙을 수정하는 순간 과거 데이터의 의미가 바뀐다.

## 권장

```ts
CareerMeta {
  createdWithRulesVersion;
  currentRulesVersion;
}
```

그리고 rule마다 migration policy를 명시한다.

---

# 17. 랭킹 조작

## 확인된 문제

- 같은 닉네임으로 베스트11 거의 모든 포지션 차지
- 자동 등록 의심 기록
- 매크로/도배
- 조작된 기록

## 공개된 해결

- 의심 기록 제거
- 운영 정책
- 상위권 서버 검증
- 확인 불가능한 기록 수동 검수
- 서버가 커리어를 처음부터 다시 계산

## 구현 교훈

클라이언트 최종 상태를 신뢰하면 안 된다.

가장 강력한 방법은:

```
initial state
+ seed
+ user commands
→ server replay
```

이다.

---

# 18. "다른 후보 보기"를 누른 커리어의 랭킹 검증 실패

## 확인된 문제

선수 생성에서 다른 후보 보기를 사용한 경우 랭킹 검증이 실패하던 문제가 있었다.

## 중요한 추론

이 문제는 **클라이언트에서 소비된 난수 순서와 서버 replay의 난수 순서가 달라졌을 가능성**을 강하게 시사한다.

예:

```
candidate generation
→ reroll
→ RNG consumes N values
```

서버가 reroll command를 재현하지 않았다면 이후 모든 RNG 결과가 달라진다.

## 권장

후보 생성도 command log에 반드시 남긴다.

```ts
{type:"REROLL_CANDIDATES", count:1}
```

또는 candidate generation seed를 별도 분리한다.

이 사례는 deterministic simulation 설계에서 매우 중요한 참고점이다.

---

# 19. 노화/부상 밸런스

Kickru는:

- 부상 빈도 감소
- 대신 아주 드문 큰 부상
- 선수별 노화 체질
- 대기만성형
- 노화 방지 아이템

등을 여러 번 조정했다.

## 교훈

단순 평균값 조정보다 **분포 형태**가 재미를 만든다.

예:

```
many careers → ordinary
few careers  → dramatic
very few     → legendary
```

평균 성장률만 바꾸지 말고 분산과 tail probability를 설계해야 한다.

---

# 20. 개발상 가장 중요한 교훈

Kickru의 패치 기록은 이 장르의 실제 난점을 잘 보여준다.

문제 대부분은 UI가 아니라:

- 시간축
- 상태 조합
- 규칙 예외
- 저장 호환
- 랜덤 재현
- 맥락 연결
- 데이터 집계

에서 발생한다.

따라서 새 프로젝트에서도 UI부터 만들기보다:

```
Domain Model
→ Simulation
→ Save/Replay
→ Tests
→ UI
```

순서로 개발하는 것이 훨씬 안전하다.

---

# 출처

- https://kickru.com/guide
- https://kickru.com/guides
- https://kickru.com/patch-notes
- https://kickru.com/guides/transfers
- https://kickru.com/guides/season-calendar
- https://kickru.com/guides/positions
- https://kickru.com/guides/ballon-dor
