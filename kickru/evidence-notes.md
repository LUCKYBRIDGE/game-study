# Kickru 분석 근거 및 기술 추론 노트

분석일: 2026-11-13

이 문서는 개발 문서에 사용한 핵심 공개 근거와, 그 근거로부터 어디까지 추론할 수 있는지를 구분한다.

## 신뢰도 표기

- **FACT**: Kickru 공식 페이지에 명시
- **STRONG INFERENCE**: 공개된 여러 동작을 구현하려면 높은 가능성으로 필요한 구조
- **RECOMMENDATION**: Kickru 내부 구현과 무관하게 새 프로젝트에서 권장하는 방식

---

## 1. 브라우저 로컬 저장

**FACT**

공식 게임 가이드는 진행 내용을 현재 브라우저에 자동 저장하며 서버에는 전체 커리어 저장 파일이 자동 백업되지 않는다고 설명한다.

출처: https://kickru.com/guide

**기술적 의미**

서비스는 적어도 커리어의 primary persistence를 브라우저 측에 두는 local-first 성격을 가진다.

**RECOMMENDATION**

새 구현에서는 localStorage보다 IndexedDB를 우선 검토한다. 커리어 기록이 길어지고 데이터가 많아질수록 quota·비동기 처리·구조화 저장에서 유리하다.

---

## 2. 저장 파일 Export / Import

**FACT**

가이드에서 저장 파일 내보내기/가져오기 기능을 안내한다.

**기술적 의미**

직렬화 가능한 Career State가 존재해야 한다.

따라서 게임 상태를 DOM/UI 상태와 강하게 결합하면 안 된다.

---

## 3. 저장 용량 부족 문제

**FACT**

2026-10-17 패치노트는 브라우저 저장 공간 부족으로 저장되지 않던 문제를 수정했고, 은퇴 커리어 보관 용량을 크게 줄였다고 설명한다.

출처: https://kickru.com/patch-notes

**기술적 교훈**

오래된 커리어를 full simulation state로 계속 보존하면 local-first 게임에서도 저장 한계에 부딪힌다.

**RECOMMENDATION**

active career와 retired career storage format을 분리한다.

---

## 4. 저장 호환성 문제

**FACT**

2026-11-07 패치에서 준프로 계약 후 산하 고교로 이동한 커리어가 "호환되지 않거나 손상된 저장 파일"로 열리지 않았고, 이미 열리지 않던 저장도 다시 열 수 있게 수정됐다.

**기술적 의미**

라이브 서비스에서는 데이터 구조가 진화하므로 단순 최신 schema validation만으로는 부족하다.

**RECOMMENDATION**

`schemaVersion + migration chain`을 초기부터 둔다.

---

## 5. 규칙 적용 정책이 서로 다름

**FACT**

패치마다 다음이 혼재한다.

- 진행 중인 커리어에 즉시 적용
- 다음 시즌부터 적용
- 새 커리어부터 적용
- 이미 기록된 과거 수상 결과는 유지

**기술적 의미**

save schema version과 game rules version은 별개로 관리해야 한다.

---

## 6. 서버 랭킹 검증

**FACT**

2026-10-12 패치에는 상위권 기록을 서버가 커리어 처음부터 다시 계산해 검증한다고 명시되어 있다.

2026-10-24 이후에도 상위권 서버 검증·검토 정책이 이어진다.

**STRONG INFERENCE**

서버가 최종 결과만 받아서는 "처음부터 다시 계산"할 수 없다.

따라서 적어도 다음 중 일부가 필요하다.

- 초기 조건
- 난수 정보
- 사용자 선택 이력
- 중간 상태/스냅샷
- 버전 정보

정확한 내부 형식은 공개되지 않았다.

**RECOMMENDATION**

`initialSeed + initialConfig + command log`를 가장 단순한 검증 기반으로 사용한다.

---

## 7. 후보 다시 뽑기 후 검증 실패

**FACT**

2026-10-12 패치에서 "다른 후보 보기"를 사용한 커리어의 랭킹 검증이 실패하던 문제가 수정됐다.

**STRONG INFERENCE**

후보 reroll이 난수 소비 순서 또는 재현 데이터와 관련되어 서버 계산과 클라이언트 결과가 어긋났을 가능성이 있다.

이 사실만으로 정확한 원인을 확정할 수는 없다.

**RECOMMENDATION**

모든 결과에 영향을 주는 사용자 행동을 replay command로 기록하고, subsystem별 deterministic seed를 분리한다.

---

## 8. 춘추제/추춘제 이적

**FACT**

2026-10-25 패치에서 시즌제가 다른 해외 구단으로 이동할 때 한 해를 통째로 더 뛰던 문제를 고쳐 시즌 중간 합류로 변경했다.

공략 페이지도 K리그↔유럽 이동 시 진행 중 시즌 후반에 합류한다고 설명한다.

출처:
- https://kickru.com/patch-notes
- https://kickru.com/guides/season-calendar

**기술적 교훈**

`seasonYear`만으로 세계 시간을 표현하기 어렵다.

**RECOMMENDATION**

절대 날짜 기반 timeline과 competition season을 분리한다.

---

## 9. 실제 리그 규칙

**FACT**

패치노트에서 실제 경기 수, 3·4회 대진, 스플릿, 승점 반감, 승강 플레이오프, B팀 제한 등 다양한 리그 규칙을 반영하고 있다.

**기술적 의미**

리그마다 조건문을 추가하는 방식은 장기적으로 복잡도가 폭발한다.

**RECOMMENDATION**

LeagueFormat / PromotionRule / QualificationRule 등 전략 데이터로 분리한다.

---

## 10. 맥락 기반 팬 반응

**FACT**

프리미어리그 선수에게 K리그 관련 댓글이 나오는 등 맥락 오류를 수정했고, 리그·구단·학교 소속에 맞는 반응만 나오도록 변경했다.

기자회견에서도 실제 성공/실패 결과를 따라 기사와 팬 반응이 나오도록 수정했다.

**기술적 의미**

텍스트 생성/선택 시스템은 global random pool이 아니라 context-filtered pool이어야 한다.

---

## 11. 연봉 인플레이션

**FACT**

연봉 인상 요청과 조기 재계약으로 연봉이 계속 불어나던 문제를 수정했다. 이후 구단 수준과 선수 기량의 범위 안에서 연봉이 정해지고, 30대 이후 인상 폭이 줄도록 바뀌었다.

**기술적 교훈**

현재 값에 multiplier를 누적하는 경제 모델은 inflation을 만든다.

**RECOMMENDATION**

매 계약마다 market target value를 새로 계산한다.

---

## 12. 이벤트 충돌

**FACT**

은퇴 권고와 전환점이 함께 발생할 때 전환점을 누를 수 없던 문제가 수정됐다.

**기술적 의미**

여러 이벤트가 동시에 modal/UI ownership을 요구할 수 있다.

**RECOMMENDATION**

event priority + conflict group + queue를 사용한다.

---

## 13. 자산 로딩 문제

**FACT**

우승 세리머니가 끊기거나 늦게 뜨는 문제를 개선하며 화면 전 미리 불러오고 파일도 가볍게 했다고 설명한다.

**기술적 교훈**

게임 화면에서 network-loaded art를 transition 순간 처음 요청하면 UX가 깨진다.

**RECOMMENDATION**

critical asset preloading과 image optimization을 사용한다.

---

## 14. 중요한 경기만 확대

**FACT**

결승·승강 플레이오프 등 특정 경기의 중계 기능을 확대해 왔다. 중계 화면에는 실제로 계산된 스코어와 승부차기 결과가 표시된다.

**STRONG INFERENCE**

전체 시즌 simulation과 presentation layer를 분리하는 구조가 적합하다.

**RECOMMENDATION**

먼저 MatchResult/EventLog를 계산하고, 중요한 경기만 해당 log를 애니메이션으로 재생한다.

---

## 15. 성장 시스템

**FACT**

공식 가이드는 성장에 나이·출전시간·활약·건강·숨겨진 성장 특성이 영향을 주며, 잠재력은 상한이고 출전이 부족하면 도달하지 못한다고 설명한다.

출처: https://kickru.com/guide

**기술적 의미**

성장은 단일 XP 시스템이 아니라 multi-factor simulation이다.

---

## 16. 포지션별 평가

**FACT**

공식 포지션 공략은 OVR이 포지션별 중요 능력치에 가중치를 두고 계산된다고 설명한다.

출처: https://kickru.com/guides/positions

**RECOMMENDATION**

POSITION_WEIGHTS를 데이터 테이블로 분리한다.

---

## 17. 수상 평가

**FACT**

발롱도르는 시즌 활약, 큰 대회 우승, 개인상, 실제 기여도를 반영하며 포지션/성향에 따른 불리함을 줄이도록 지속적으로 조정됐다.

출처:
- https://kickru.com/guides/ballon-dor
- https://kickru.com/patch-notes

**기술적 교훈**

award score는 raw 공격포인트가 아니라 role-normalized score가 필요하다.

---

# 결론

공개 자료만으로도 Kickru는 단순한 랜덤 텍스트 게임보다 훨씬 복잡한 문제를 다루고 있음을 알 수 있다.

특히 개발적으로 중요한 축은 다음이다.

```text
Timeline
Save Compatibility
Deterministic Replay
Rule Versioning
Contextual Content
League Exceptions
Balance Simulation
Server Validation
```

내부 구현을 그대로 알 수는 없지만, 실제 발생한 버그와 공개된 수정 방향은 비슷한 장르를 새로 개발할 때 매우 가치 있는 engineering case study가 된다.
