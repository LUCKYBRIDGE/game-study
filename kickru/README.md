# Kickru 분석 인덱스

대상: [kickru.com — 벤치에서 발롱까지](https://kickru.com/)

이 폴더는 Kickru를 단순히 "어떤 기능이 있는 게임인가" 수준에서 정리하지 않고, **비슷한 장르의 웹 게임을 실제로 설계·개발할 수 있도록 시스템·데이터·상태·알고리즘·저장·검증·운영 문제까지 분해한 개발 분석 자료**를 모은다.

## 문서 구성

1. [analysis.md](./analysis.md)
   - 서비스/게임 구조 전체 분석
   - 핵심 루프, 성장, 선택, 내러티브, 랭킹, 라이브옵스

2. [game-systems.md](./game-systems.md)
   - 실제 구현 가능한 수준으로 게임 시스템 분해
   - 선수 상태, 시즌 시뮬레이션, 성장, 이적, 계약, 수상, 노화, 이벤트, 내러티브

3. [technical-architecture.md](./technical-architecture.md)
   - 웹 게임 아키텍처
   - local-first 저장, 서버 랭킹, 재현 가능한 시뮬레이션, 저장 버전 관리, 성능, 배포

4. [problem-solving.md](./problem-solving.md)
   - Kickru 패치노트에서 확인되는 실제 개발 문제
   - 문제 원인 추정, 실제 패치 방향, 재구현 시 권장 해결 구조

5. [data-model.md](./data-model.md)
   - TypeScript 기준 데이터 모델 초안
   - Career / Player / Club / League / Season / Event / Offer / Award / Save / Ranking

6. [implementation-roadmap.md](./implementation-roadmap.md)
   - MVP부터 완성형까지 실제 개발 순서
   - 테스트 전략, 밸런스 검증, 확장 순서

## 분석 원칙

이 문서들은 다음을 구분한다.

- **확인된 사실**: Kickru 공개 가이드/공략/패치노트에서 직접 확인
- **구조적 추론**: 공개 동작을 구현하려면 필요할 것으로 보이는 내부 구조
- **개발 권장안**: 우리가 유사 장르를 새로 만들 때 가장 안전하고 유지보수하기 좋은 방식

특히 내부 소스코드나 비공개 서버 구조를 확인한 것이 아니므로, 기술 스택 자체를 Kickru의 실제 구현이라고 단정하지 않는다.

## 핵심 결론

Kickru를 구현 관점에서 보면 본질은 다음과 같다.

```text
Career State
  + World Rules
  + Season Simulator
  + Event/Choice Engine
  + Persistent History
  + Deterministic/Replayable Results
  + Local Save
  + Server Ranking Validation
```

그래픽보다 **시뮬레이션 상태와 규칙의 일관성**이 훨씬 중요한 게임이다.
