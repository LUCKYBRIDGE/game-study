# Kickru류 커리어 게임 데이터 모델 초안

TypeScript 기반으로 실제 구현 가능한 형태를 제안한다.

---

# 1. ID 원칙

이름을 ID로 쓰지 않는다.

```ts
type CareerId = string;
type PlayerId = string;
type ClubId = string;
type LeagueId = string;
type CompetitionId = string;
type SeasonId = string;
type EventId = string;
```

모든 관계는 stable ID로 연결한다.

---

# 2. Player

```ts
type Position =
  | "ST" | "LW" | "RW"
  | "CAM" | "CM" | "CDM"
  | "CB" | "LB" | "RB"
  | "GK";

type OutfieldStats = {
  speed: number;
  shooting: number;
  passing: number;
  dribbling: number;
  defense: number;
  physical: number;
};

type DevelopmentProfile = {
  potential: number;
  growthRate: number;
  breakoutAge: number | null;
  agingStartAge: number;
  agingRate: number;
  durability: number;
  consistency: number;
};

type PlayerState = {
  id: PlayerId;
  name: string;
  nationality: string;

  birthDate: string;
  age: number;

  position: Position;
  preferredFoot: "LEFT" | "RIGHT" | "BOTH";

  stats: OutfieldStats;
  ovr: number;

  development: DevelopmentProfile;

  reputation: number;
  morale: number;
  fitness: number;

  currentClubId: ClubId | null;
  squadRole: SquadRole;

  trainingFocus: StatKey[];

  injury: InjuryState | null;
};
```

---

# 3. Club

```ts
type ClubState = {
  id: ClubId;
  name: string;

  countryCode: string;
  leagueId: LeagueId;

  reputation: number;
  squadStrength: number;
  wageLevel: number;

  rivalClubIds: ClubId[];

  isReserveTeam: boolean;
  parentClubId?: ClubId;

  eligibilityTags: string[];
};
```

---

# 4. League / Competition

```ts
type CompetitionSeasonDefinition = {
  id: SeasonId;
  competitionId: CompetitionId;

  start: GameDate;
  end: GameDate;

  formatId: string;
  registrationWindows: DateRange[];
};

type LeagueFormat = {
  rounds: number;

  split?: {
    afterRound: number;
    groups: number;
    pointMultiplier?: number;
  };

  promotionSlots: number;
  relegationSlots: number;

  playoffs: PlayoffRule[];
};
```

---

# 5. GameDate

```ts
type GameDate = {
  year: number;
  month: number;
  week?: number;
};
```

"2031 시즌"만 저장하면 춘추제/추춘제 이동에서 문제가 생긴다.

---

# 6. Contract

```ts
type ContractState = {
  clubId: ClubId;

  startDate: GameDate;
  endDate: GameDate;

  annualSalary: number;
  squadRole: SquadRole;

  loan?: {
    parentClubId: ClubId;
    returnDate: GameDate;
  };
};
```

---

# 7. Season Stats

```ts
type CompetitionStats = {
  competitionId: CompetitionId;
  seasonId: SeasonId;

  appearances: number;
  starts: number;
  minutes: number;

  goals: number;
  assists: number;
  cleanSheets: number;

  averageRating: number;
};

type SeasonSummary = {
  seasonId: SeasonId;

  playerAge: number;
  clubId: ClubId;

  stats: CompetitionStats[];

  trophies: TrophyResult[];
  awards: AwardResult[];

  ovrStart: number;
  ovrEnd: number;
};
```

---

# 8. Match

```ts
type MatchResult = {
  id: string;
  date: GameDate;

  competitionId: CompetitionId;
  phase:
    | "REGULAR"
    | "CUP"
    | "SPLIT"
    | "PLAYOFF"
    | "FINAL";

  homeClubId: ClubId;
  awayClubId: ClubId;

  homeScore: number;
  awayScore: number;

  penaltyScore?: [number, number];

  playerResult?: PlayerMatchResult;
};
```

---

# 9. Career Event

```ts
type CareerEventDefinition = {
  id: EventId;

  category:
    | "TRAINING"
    | "TRANSFER"
    | "CONTRACT"
    | "MEDIA"
    | "CAPTAIN"
    | "POSITION"
    | "RETIREMENT"
    | "MILITARY";

  priority: number;
  conflictGroup?: string;
  cooldownSeasons?: number;

  conditions: Condition[];
  exclusions: Condition[];

  choices: EventChoice[];
};
```

```ts
type EventChoice = {
  id: string;
  label: string;

  probability?: number;

  successEffects?: Effect[];
  failureEffects?: Effect[];
  deterministicEffects?: Effect[];
};
```

---

# 10. Career Moment

```ts
type CareerMoment = {
  id: string;
  seasonId: SeasonId;

  type: string;
  importance: number;

  actorClubId?: ClubId;
  relatedClubId?: ClubId;
  competitionId?: CompetitionId;

  payload: Record<string, string | number | boolean>;
};
```

이 데이터가 은퇴 후 인생 이야기의 원료가 된다.

---

# 11. Offer

```ts
type TransferOffer = {
  id: string;
  fromClubId: ClubId;

  createdAt: GameDate;

  squadRole: SquadRole;
  salary: number;
  contractYears: number;

  transferType: "PERMANENT" | "LOAN" | "FREE";
};
```

제안서를 저장해야 "거절했던 이적 제안"도 커리어 회고에 사용할 수 있다.

---

# 12. Award

```ts
type AwardResult = {
  awardId: string;
  seasonId: SeasonId;

  rank?: number;

  evaluation: {
    performanceScore: number;
    trophyScore: number;
    competitionScore: number;
    roleAdjustedScore: number;
  };
};
```

디버그 모드에서는 evaluation detail을 저장하면 밸런싱이 쉬워진다.

---

# 13. Finance / Shop

```ts
type FinanceState = {
  cash: number;
  careerIncome: number;
};

type ItemDefinition = {
  id: string;
  price: number;

  effects: Effect[];
  stackLimit?: number;
};
```

아이템 효과를 코드에 직접 쓰지 않고 Effect로 통일한다.

---

# 14. Save File

```ts
type SaveFile = {
  schemaVersion: number;
  rulesVersion: string;

  careerId: CareerId;
  createdAt: string;
  updatedAt: string;

  seed: string;

  state: CareerState;
  replay: ReplayLog;
};
```

---

# 15. Replay

```ts
type ReplayCommand =
  | { type: "CREATE_PLAYER"; payload: CreatePlayerCommand }
  | { type: "REROLL_CANDIDATES" }
  | { type: "ADVANCE_SEASON" }
  | { type: "SELECT_EVENT_CHOICE"; eventId: string; choiceId: string }
  | { type: "ACCEPT_TRANSFER"; offerId: string }
  | { type: "REJECT_TRANSFER"; offerId: string }
  | { type: "BUY_ITEM"; itemId: string }
  | { type: "RETIRE" };

type ReplayLog = {
  initialSeed: string;
  commands: ReplayCommand[];
};
```

---

# 16. Career Flags

이벤트 조건을 위해 boolean을 수백 개 만들면 관리가 어려워진다.

```ts
type CareerFlags = {
  values: Record<string, boolean | number | string>;
  occurrenceCount: Record<string, number>;
  lastTriggeredSeason: Record<string, SeasonId>;
};
```

다만 핵심 도메인 상태는 flags에 숨기지 않는다.

예: captain 여부는 별도 필드가 낫다.

---

# 17. Static Data vs Save Data

Static:

- 구단
- 리그
- 대회
- 이벤트 정의
- 아이템
- 수상 규칙

Save:

- 내 선수
- 현재 커리어 상태
- 발생한 사건
- 선택
- 기록

static data 전체를 save에 넣지 않는다.

---

# 18. 데이터 검증

Zod 예:

```ts
const SaveSchema = z.object({
  schemaVersion: z.number().int(),
  rulesVersion: z.string(),
  careerId: z.string(),
  seed: z.string(),
  state: CareerStateSchema,
  replay: ReplayLogSchema
});
```

단, 오래된 파일은 최신 schema로 바로 파싱하지 말고 migration 전에 최소 envelope만 읽는다.

---

# 19. Migration

```ts
function migrateSave(raw: unknown): SaveFile {
  const version = readEnvelopeVersion(raw);

  let value = raw;

  if (version < 5) value = migrate4to5(value);
  if (version < 6) value = migrate5to6(value);
  if (version < 7) value = migrate6to7(value);

  return SaveSchema.parse(value);
}
```

---

# 20. 데이터 모델 핵심

이 장르의 데이터 모델은 "현재 선수 능력치"보다 **시간에 따른 기록**이 더 중요하다.

따라서 핵심은:

```
Current State
+ Immutable History
+ Replay Commands
```

세 축이다.

현재 state만 저장하면:

- 과거 회고
- 랭킹 검증
- 버그 재현
- 통계
- 서사 생성

이 모두 어려워진다.
