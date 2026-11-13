# game-study

웹 게임, 앱 게임, 게임형 웹서비스를 심층 분석하고 재구성 가능한 설계 지식으로 축적하는 저장소입니다.

## 분석 원칙

- 단순 기능 목록이 아니라 **Core Loop → Progression → Choice → Economy → Feedback → Retention** 흐름으로 분석합니다.
- UI/UX, 게임 규칙, 데이터 구조, 저장 방식, 서버 의존성, 운영 방식까지 함께 봅니다.
- 직접 확인된 사실과 기술적 추정을 구분합니다.
- 원본을 복제하기보다 **재사용 가능한 디자인 패턴과 시스템 설계 원리**를 추출합니다.
- 분석 방법론은 [storytold/craft-anything](https://github.com/storytold/craft-anything)의 Study → Build → Check 관점을 참고합니다.

## 분석 목록

| 서비스 | 유형 | 핵심 장르 | 분석 |
|---|---|---|---|
| Kickru - 벤치에서 발롱까지 | Web Game | 축구선수 커리어 시뮬레이션 / 육성 / 선택형 RPG | [심층 분석](studies/kickru/analysis.md) |

## 디렉터리

```
studies/
  kickru/
    analysis.md
```

각 서비스별 폴더에는 향후 필요에 따라 `analysis.md`, `screens.md`, `mechanics.md`, `technical-notes.md` 등을 추가합니다.
