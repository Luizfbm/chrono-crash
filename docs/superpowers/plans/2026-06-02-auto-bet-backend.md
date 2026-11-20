# Auto Bet Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the backend-only Auto bet bonus: one persistent fixed-value auto bet session per player, starting on the next betting round and stopping by configured limits or failures.

**Architecture:** Add an AutoBet application model and repository in the Game bounded context. Expose authenticated REST endpoints for start/status/stop, reuse `PlaceBetUseCase` for automated placement with a new source flag, run the executor inline when a new betting round opens, and apply session PnL when auto-bet executions resolve.

**Tech Stack:** NestJS, Bun test, Prisma/PostgreSQL, RabbitMQ wallet outbox, Socket.IO realtime publishing, Prometheus metrics, Kong/Keycloak E2E.

---

## Operating Rules

- Do not commit specs or design docs unless the user explicitly asks for that commit.
- Do not change frontend code in this backend-only slice.
- Use TDD one vertical behavior at a time.
- Never delete, disable, skip, or weaken existing tests.
- Run `git status --short --branch` before edits and before commits.

## File Map

Create:

- `services/games/src/application/auto-bet/auto-bet-session.ts`  
  Owns AutoBet types, validation, status/stop-reason constants, and result delta calculation.
- `services/games/src/application/ports/auto-bet-session.repository.ts`  
  Application port and injection token for AutoBet persistence.
- `services/games/src/application/use-cases/start-auto-bet-session.use-case.ts`  
  Starts one active session per player and records `startsAfterRoundId`.
- `services/games/src/application/use-cases/get-my-auto-bet-session.use-case.ts`  
  Returns active session or latest stopped session for the authenticated player.
- `services/games/src/application/use-cases/stop-auto-bet-session.use-case.ts`  
  Idempotently stops the active session or returns latest stopped session.
- `services/games/src/application/use-cases/execute-auto-bets-for-round.use-case.ts`  
  Places automated bets for active sessions when a new betting round opens.
- `services/games/src/application/use-cases/apply-auto-bet-result.use-case.ts`  
  Applies terminal bet result deltas idempotently and stops by PnL thresholds.
- `services/games/src/infrastructure/prisma/auto-bet-session.mapper.ts`  
  Converts Prisma AutoBet records to application types.
- `services/games/src/infrastructure/prisma/auto-bet-session-prisma.repository.ts`  
  PostgreSQL implementation of the AutoBet repository.
- `services/games/src/presentation/dtos/auto-bet-session-request.dto.ts`  
  Swagger request DTO.
- `services/games/src/presentation/dtos/auto-bet-session-response.dto.ts`  
  Swagger response DTO.
- `services/games/src/presentation/auto-bet-session-response.mapper.ts`  
  Converts AutoBet sessions to public response shape.
- `services/games/prisma/migrations/20260602130000_add_auto_bet_sessions/migration.sql`  
  Creates AutoBet tables and indexes.
- `services/games/tests/e2e/auto-bet.e2e.test.ts`  
  Kong/Keycloak E2E for backend auto bet.

Modify:

- `services/games/prisma/schema.prisma`  
  Add AutoBet Prisma models.
- `services/games/src/application/game.errors.ts`  
  Add AutoBet errors.
- `services/games/src/presentation/game-http-errors.ts`  
  Map AutoBet errors to HTTP exceptions.
- `services/games/src/application/use-cases/place-bet.use-case.ts`  
  Add source flag and manual-bet block.
- `services/games/src/application/use-cases/advance-round-lifecycle.use-case.ts`  
  Invoke executor after opening a new round and apply lost results after settlement.
- `services/games/src/application/use-cases/cash-out.use-case.ts`  
  Apply cashed-out result after manual cashout completes.
- `services/games/src/application/use-cases/advance-round-lifecycle.use-case.ts`  
  Apply cashed-out result after automatic cashout completes.
- `services/games/src/infrastructure/observability/game-metrics.ts`  
  Add AutoBet counters.
- `services/games/src/presentation/controllers/games.controller.ts`  
  Add endpoints and constructor dependencies.
- `services/games/src/app.module.ts`  
  Wire repository and use cases.
- `services/games/tests/unit/application/game-use-cases.test.ts`  
  Add application behavior tests and fake repository.
- `services/games/tests/unit/games-controller.test.ts`  
  Add controller contract tests.
- `services/games/tests/e2e/e2e-helpers.ts`  
  Add AutoBet response and helper functions.

---

### Task 1: AutoBet Model And Start/Status/Stop Use Cases

**Files:**

- Create: `services/games/src/application/auto-bet/auto-bet-session.ts`
- Create: `services/games/src/application/ports/auto-bet-session.repository.ts`
- Create: `services/games/src/application/use-cases/start-auto-bet-session.use-case.ts`
- Create: `services/games/src/application/use-cases/get-my-auto-bet-session.use-case.ts`
- Create: `services/games/src/application/use-cases/stop-auto-bet-session.use-case.ts`
- Modify: `services/games/src/application/game.errors.ts`
- Test: `services/games/tests/unit/application/game-use-cases.test.ts`

- [ ] **Step 1: Write the failing start-session tests**

Append tests to `services/games/tests/unit/application/game-use-cases.test.ts` after the existing use case setup helpers. Add imports for the new use cases and repository types.

Add these helpers near the existing `openRound` helper:

```ts
class SequenceIdGenerator implements IdGenerator {
  constructor(private readonly ids: string[]) {}

  generate(): string {
    const id = this.ids.shift();

    if (!id) {
      throw new Error("SequenceIdGenerator ran out of ids");
    }

    return id;
  }
}

function openRoundFixture(
  overrides: {
    chainIndex?: number;
    crashPointBp?: number;
    id?: string;
  } = {},
): Round {
  return Round.openBetting({
    id: overrides.id ?? "round-1",
    bettingStartsAt: new Date("2026-05-30T10:00:00.000Z"),
    bettingEndsAt: new Date("2026-05-30T10:00:10.000Z"),
    crashPointBp: overrides.crashPointBp ?? 30000,
    serverSeedHash: "seed-hash",
    clientSeed: "client-seed",
    nonce: overrides.chainIndex ?? 1,
    chainIndex: overrides.chainIndex ?? 1,
  });
}

function acceptedBetFixture(
  overrides: {
    autoCashoutMultiplierBp?: number | null;
    id?: string;
    playerId?: string;
    roundId?: string;
  } = {},
): Bet {
  return Bet.accepted({
    id: overrides.id ?? "bet-1",
    roundId: overrides.roundId ?? "round-1",
    playerId: overrides.playerId ?? "player-1",
    username: "player",
    amountCents: 1000n,
    autoCashoutMultiplierBp: overrides.autoCashoutMultiplierBp ?? null,
  });
}

function autoBetSessionFixture(
  overrides: Partial<AutoBetSession> = {},
): AutoBetSession {
  return {
    id: "auto-session-1",
    playerId: "player-1",
    username: "player",
    status: "ACTIVE",
    amountCents: 1000n,
    autoCashoutMultiplierBp: null,
    maxRounds: 3,
    roundsPlayed: 0,
    netProfitCents: 0n,
    stopLossCents: null,
    takeProfitCents: null,
    stopReason: null,
    startsAfterRoundId: null,
    createdAt: new Date("2026-05-30T10:00:00.000Z"),
    updatedAt: new Date("2026-05-30T10:00:00.000Z"),
    stoppedAt: null,
    ...overrides,
  };
}

function autoBetExecutionFixture(
  overrides: Partial<AutoBetRoundExecution> = {},
): AutoBetRoundExecution {
  return {
    id: "execution-1",
    sessionId: "auto-session-1",
    roundId: "round-1",
    betId: "bet-1",
    status: "BET_PLACED",
    reason: null,
    resultStatus: null,
    resultDeltaCents: null,
    resultAppliedAt: null,
    createdAt: new Date("2026-05-30T10:00:00.000Z"),
    ...overrides,
  };
}
```

```ts
test("starts a persistent auto bet session after the current round", async () => {
  const round = openRoundFixture();
  const gameRepository = new InMemoryGameRepository(round);
  const autoBetRepository = new FakeAutoBetSessionRepository();
  const useCase = new StartAutoBetSessionUseCase(
    gameRepository,
    autoBetRepository,
    new SequenceIdGenerator(["auto-session-1"]),
  );

  await expect(
    useCase.execute({
      amountCents: "1000",
      autoCashoutMultiplierBp: 20000,
      maxRounds: 3,
      playerId: "player-1",
      stopLossCents: "3000",
      takeProfitCents: "5000",
      username: "player",
    }),
  ).resolves.toMatchObject({
    id: "auto-session-1",
    amountCents: 1000n,
    autoCashoutMultiplierBp: 20000,
    maxRounds: 3,
    netProfitCents: 0n,
    playerId: "player-1",
    roundsPlayed: 0,
    startsAfterRoundId: round.id,
    status: "ACTIVE",
    stopLossCents: 3000n,
    takeProfitCents: 5000n,
  });
});

test("rejects a second active auto bet session for the same player", async () => {
  const gameRepository = new InMemoryGameRepository(openRoundFixture());
  const autoBetRepository = new FakeAutoBetSessionRepository();
  const useCase = new StartAutoBetSessionUseCase(
    gameRepository,
    autoBetRepository,
    new SequenceIdGenerator(["auto-session-1", "auto-session-2"]),
  );

  await useCase.execute({
    amountCents: "1000",
    maxRounds: 2,
    playerId: "player-1",
    username: "player",
  });

  await expect(
    useCase.execute({
      amountCents: "1000",
      maxRounds: 2,
      playerId: "player-1",
      username: "player",
    }),
  ).rejects.toThrow(AutoBetSessionActiveError);
});

test("gets active auto bet session or latest stopped session", async () => {
  const autoBetRepository = new FakeAutoBetSessionRepository();
  const getUseCase = new GetMyAutoBetSessionUseCase(autoBetRepository);
  const stopUseCase = new StopAutoBetSessionUseCase(autoBetRepository);

  autoBetRepository.sessions.push(
    autoBetSessionFixture({
      id: "auto-session-1",
      playerId: "player-1",
      status: "ACTIVE",
    }),
  );

  await expect(
    getUseCase.execute({ playerId: "player-1" }),
  ).resolves.toMatchObject({ id: "auto-session-1", status: "ACTIVE" });

  await stopUseCase.execute({ playerId: "player-1" });

  await expect(
    getUseCase.execute({ playerId: "player-1" }),
  ).resolves.toMatchObject({
    id: "auto-session-1",
    status: "STOPPED",
    stopReason: "MANUAL",
  });
});
```

Add a `FakeAutoBetSessionRepository` near the other fake repositories:

```ts
class FakeAutoBetSessionRepository implements AutoBetSessionRepository {
  public sessions: AutoBetSession[] = [];
  public executions: AutoBetRoundExecution[] = [];

  async create(input: NewAutoBetSession): Promise<AutoBetSession> {
    const session: AutoBetSession = {
      ...input,
      createdAt: new Date("2026-05-30T10:00:00.000Z"),
      updatedAt: new Date("2026-05-30T10:00:00.000Z"),
      stoppedAt: null,
    };
    this.sessions.push(session);
    return session;
  }

  async findActiveByPlayer(playerId: string): Promise<AutoBetSession | null> {
    return (
      this.sessions.find(
        (session) => session.playerId === playerId && session.status === "ACTIVE",
      ) ?? null
    );
  }

  async findLatestByPlayer(playerId: string): Promise<AutoBetSession | null> {
    return (
      this.sessions
        .filter((session) => session.playerId === playerId)
        .sort((left, right) => right.createdAt.getTime() - left.createdAt.getTime())[0] ??
      null
    );
  }

  async listActive(): Promise<AutoBetSession[]> {
    return this.sessions.filter((session) => session.status === "ACTIVE");
  }

  async stop(input: StopAutoBetSessionInput): Promise<AutoBetSession> {
    const session = this.sessions.find((candidate) => candidate.id === input.sessionId);
    if (!session) {
      throw new Error(`Session not found: ${input.sessionId}`);
    }
    Object.assign(session, {
      status: "STOPPED" as const,
      stopReason: input.reason,
      stoppedAt: new Date("2026-05-30T10:00:00.000Z"),
      updatedAt: new Date("2026-05-30T10:00:00.000Z"),
    });
    return session;
  }

  async findExecution(
    sessionId: string,
    roundId: string,
  ): Promise<AutoBetRoundExecution | null> {
    return (
      this.executions.find(
        (execution) => execution.sessionId === sessionId && execution.roundId === roundId,
      ) ?? null
    );
  }

  async recordExecution(
    input: NewAutoBetRoundExecution,
  ): Promise<AutoBetRoundExecution> {
    const execution: AutoBetRoundExecution = {
      ...input,
      createdAt: new Date("2026-05-30T10:00:00.000Z"),
      resultAppliedAt: null,
      resultDeltaCents: null,
      resultStatus: null,
    };
    this.executions.push(execution);
    return execution;
  }

  async incrementRoundsPlayed(sessionId: string): Promise<AutoBetSession> {
    const session = this.sessions.find((candidate) => candidate.id === sessionId);
    if (!session) {
      throw new Error(`Session not found: ${sessionId}`);
    }
    session.roundsPlayed += 1;
    return session;
  }

  async applyBetResult(input: ApplyAutoBetResultInput): Promise<AutoBetSession | null> {
    const execution = this.executions.find(
      (candidate) => candidate.id === input.executionId,
    );
    if (!execution || execution.resultAppliedAt) {
      return null;
    }
    const session = this.sessions.find(
      (candidate) => candidate.id === execution.sessionId,
    );
    if (!session) {
      throw new Error(`Session not found: ${execution.sessionId}`);
    }
    execution.resultStatus = input.resultStatus;
    execution.resultDeltaCents = input.deltaCents;
    execution.resultAppliedAt = new Date("2026-05-30T10:00:00.000Z");
    session.netProfitCents += input.deltaCents;
    return session;
  }
}
```

- [ ] **Step 2: Run the focused test to verify it fails**

Run:

```bash
cd services/games && bun test tests/unit/application/game-use-cases.test.ts
```

Expected: fail with missing AutoBet imports/classes.

- [ ] **Step 3: Implement model, repository port, errors, and use cases**

Create `services/games/src/application/auto-bet/auto-bet-session.ts`:

```ts
import {
  AutoBetSessionConfigInvalidError,
} from "../game.errors";
import {
  MAX_BET_AMOUNT_CENTS,
  MIN_BET_AMOUNT_CENTS,
} from "../game.constants";
import { parseAutoCashoutMultiplierBp } from "../auto-cashout";
import { toCents } from "../cents";

export type AutoBetSessionStatus = "ACTIVE" | "STOPPED";

export type AutoBetStopReason =
  | "MANUAL"
  | "MAX_ROUNDS_REACHED"
  | "STOP_LOSS_REACHED"
  | "TAKE_PROFIT_REACHED"
  | "WALLET_REJECTED"
  | "WALLET_UNAVAILABLE"
  | "ROUND_NOT_AVAILABLE";

export type AutoBetSession = {
  id: string;
  playerId: string;
  username: string;
  status: AutoBetSessionStatus;
  amountCents: bigint;
  autoCashoutMultiplierBp: number | null;
  maxRounds: number;
  roundsPlayed: number;
  netProfitCents: bigint;
  stopLossCents: bigint | null;
  takeProfitCents: bigint | null;
  stopReason: AutoBetStopReason | null;
  startsAfterRoundId: string | null;
  createdAt: Date;
  updatedAt: Date;
  stoppedAt: Date | null;
};

export type AutoBetRoundExecutionStatus = "BET_PLACED" | "SKIPPED" | "FAILED";
export type AutoBetResultStatus = "LOST" | "CASHED_OUT";

export type AutoBetRoundExecution = {
  id: string;
  sessionId: string;
  roundId: string;
  betId: string | null;
  status: AutoBetRoundExecutionStatus;
  reason: string | null;
  resultStatus: AutoBetResultStatus | null;
  resultDeltaCents: bigint | null;
  resultAppliedAt: Date | null;
  createdAt: Date;
};

export type StartAutoBetConfig = {
  amountCents: bigint | number | string;
  autoCashoutMultiplierBp?: number | null;
  maxRounds: number;
  stopLossCents?: bigint | number | string | null;
  takeProfitCents?: bigint | number | string | null;
};

export type ParsedAutoBetConfig = {
  amountCents: bigint;
  autoCashoutMultiplierBp: number | null;
  maxRounds: number;
  stopLossCents: bigint | null;
  takeProfitCents: bigint | null;
};

export function parseAutoBetConfig(input: StartAutoBetConfig): ParsedAutoBetConfig {
  const amountCents = toCents(input.amountCents);

  if (
    amountCents < MIN_BET_AMOUNT_CENTS ||
    amountCents > MAX_BET_AMOUNT_CENTS
  ) {
    throw new AutoBetSessionConfigInvalidError(
      "Auto bet amount must be between 1.00 and 1000.00",
    );
  }

  if (!Number.isInteger(input.maxRounds) || input.maxRounds < 1 || input.maxRounds > 100) {
    throw new AutoBetSessionConfigInvalidError(
      "Auto bet maxRounds must be between 1 and 100",
    );
  }

  return {
    amountCents,
    autoCashoutMultiplierBp: parseAutoCashoutMultiplierBp(
      input.autoCashoutMultiplierBp,
    ),
    maxRounds: input.maxRounds,
    stopLossCents: parseOptionalPositiveCents(
      input.stopLossCents,
      "stopLossCents",
    ),
    takeProfitCents: parseOptionalPositiveCents(
      input.takeProfitCents,
      "takeProfitCents",
    ),
  };
}

export function shouldStopForProfitLimits(
  session: AutoBetSession,
): AutoBetStopReason | null {
  if (
    session.stopLossCents !== null &&
    session.netProfitCents <= -session.stopLossCents
  ) {
    return "STOP_LOSS_REACHED";
  }

  if (
    session.takeProfitCents !== null &&
    session.netProfitCents >= session.takeProfitCents
  ) {
    return "TAKE_PROFIT_REACHED";
  }

  return null;
}

function parseOptionalPositiveCents(
  value: bigint | number | string | null | undefined,
  fieldName: string,
): bigint | null {
  if (value === null || value === undefined) {
    return null;
  }

  const cents = toCents(value);

  if (cents <= 0n) {
    throw new AutoBetSessionConfigInvalidError(
      `Auto bet ${fieldName} must be a positive integer`,
    );
  }

  return cents;
}
```

Create `services/games/src/application/ports/auto-bet-session.repository.ts`:

```ts
import type {
  AutoBetResultStatus,
  AutoBetRoundExecution,
  AutoBetRoundExecutionStatus,
  AutoBetSession,
  AutoBetStopReason,
} from "../auto-bet/auto-bet-session";

export const AUTO_BET_SESSION_REPOSITORY = Symbol("AUTO_BET_SESSION_REPOSITORY");

export type NewAutoBetSession = Omit<
  AutoBetSession,
  "createdAt" | "updatedAt" | "stoppedAt"
> & {
  stoppedAt: null;
};

export type StopAutoBetSessionInput = {
  sessionId: string;
  reason: AutoBetStopReason;
};

export type NewAutoBetRoundExecution = {
  id: string;
  sessionId: string;
  roundId: string;
  betId: string | null;
  status: AutoBetRoundExecutionStatus;
  reason: string | null;
};

export type ApplyAutoBetResultInput = {
  executionId: string;
  resultStatus: AutoBetResultStatus;
  deltaCents: bigint;
};

export interface AutoBetSessionRepository {
  create(input: NewAutoBetSession): Promise<AutoBetSession>;
  findActiveByPlayer(playerId: string): Promise<AutoBetSession | null>;
  findLatestByPlayer(playerId: string): Promise<AutoBetSession | null>;
  listActive(): Promise<AutoBetSession[]>;
  stop(input: StopAutoBetSessionInput): Promise<AutoBetSession>;
  findExecution(
    sessionId: string,
    roundId: string,
  ): Promise<AutoBetRoundExecution | null>;
  recordExecution(
    input: NewAutoBetRoundExecution,
  ): Promise<AutoBetRoundExecution>;
  incrementRoundsPlayed(sessionId: string): Promise<AutoBetSession>;
  applyBetResult(
    input: ApplyAutoBetResultInput,
  ): Promise<AutoBetSession | null>;
}
```

Add to `services/games/src/application/game.errors.ts`:

```ts
export class AutoBetSessionActiveError extends Error {
  constructor() {
    super("An auto bet session is already active");
    this.name = "AutoBetSessionActiveError";
  }
}

export class AutoBetSessionConfigInvalidError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "AutoBetSessionConfigInvalidError";
  }
}

export class ManualBetBlockedByAutoBetError extends Error {
  constructor() {
    super("Manual betting is blocked while auto bet is active");
    this.name = "ManualBetBlockedByAutoBetError";
  }
}
```

Create the three use cases with these constructor shapes:

```ts
export class StartAutoBetSessionUseCase {
  constructor(
    private readonly gameRepository: GameRepository,
    private readonly autoBetSessionRepository: AutoBetSessionRepository,
    private readonly idGenerator: IdGenerator,
  ) {}
}

export class GetMyAutoBetSessionUseCase {
  constructor(
    private readonly autoBetSessionRepository: AutoBetSessionRepository,
  ) {}
}

export class StopAutoBetSessionUseCase {
  constructor(
    private readonly autoBetSessionRepository: AutoBetSessionRepository,
  ) {}
}
```

Implement `StartAutoBetSessionUseCase.execute` to parse config, reject an
existing active session, read `gameRepository.findCurrentRound()`, and call
repository `create`.

- [ ] **Step 4: Run the focused test to verify it passes**

Run:

```bash
cd services/games && bun test tests/unit/application/game-use-cases.test.ts
```

Expected: pass.

- [ ] **Step 5: Commit**

```bash
git status --short --branch
git add services/games/src/application/auto-bet/auto-bet-session.ts \
  services/games/src/application/ports/auto-bet-session.repository.ts \
  services/games/src/application/use-cases/start-auto-bet-session.use-case.ts \
  services/games/src/application/use-cases/get-my-auto-bet-session.use-case.ts \
  services/games/src/application/use-cases/stop-auto-bet-session.use-case.ts \
  services/games/src/application/game.errors.ts \
  services/games/tests/unit/application/game-use-cases.test.ts
git commit -m "feat(games): add auto bet session use cases"
```

---

### Task 2: Controller Contract For AutoBet Sessions

**Files:**

- Create: `services/games/src/presentation/dtos/auto-bet-session-request.dto.ts`
- Create: `services/games/src/presentation/dtos/auto-bet-session-response.dto.ts`
- Create: `services/games/src/presentation/auto-bet-session-response.mapper.ts`
- Modify: `services/games/src/presentation/controllers/games.controller.ts`
- Modify: `services/games/src/presentation/game-http-errors.ts`
- Test: `services/games/tests/unit/games-controller.test.ts`

- [ ] **Step 1: Write the failing controller tests**

In `services/games/tests/unit/games-controller.test.ts`, extend `createController` with three new overrides and constructor arguments:

```ts
startAutoBetSessionUseCase: overrides.startAutoBetSessionUseCase ?? {
  execute: async () => autoBetSessionFixture(),
},
getMyAutoBetSessionUseCase: overrides.getMyAutoBetSessionUseCase ?? {
  execute: async () => null,
},
stopAutoBetSessionUseCase: overrides.stopAutoBetSessionUseCase ?? {
  execute: async () => null,
},
```

Add tests:

```ts
test("starts an auto bet session for the authenticated user", async () => {
  const controller = createController({
    startAutoBetSessionUseCase: {
      execute: async (input: unknown) => {
        expect(input).toEqual({
          amountCents: "1000",
          autoCashoutMultiplierBp: 20000,
          maxRounds: 3,
          playerId: "player-1",
          stopLossCents: "3000",
          takeProfitCents: "5000",
          username: "player",
        });

        return autoBetSessionFixture();
      },
    },
  });

  await expect(
    controller.startAutoBetSession(
      { playerId: "player-1", username: "player" },
      {
        amountCents: "1000",
        autoCashoutMultiplierBp: 20000,
        maxRounds: 3,
        stopLossCents: "3000",
        takeProfitCents: "5000",
      },
    ),
  ).resolves.toMatchObject({
    id: "auto-session-1",
    amountCents: "1000",
    status: "ACTIVE",
  });
});

test("gets and idempotently stops the authenticated user's auto bet session", async () => {
  const controller = createController({
    getMyAutoBetSessionUseCase: {
      execute: async () => autoBetSessionFixture(),
    },
    stopAutoBetSessionUseCase: {
      execute: async () =>
        autoBetSessionFixture({ status: "STOPPED", stopReason: "MANUAL" }),
    },
  });

  await expect(
    controller.myAutoBetSession({ playerId: "player-1", username: "player" }),
  ).resolves.toMatchObject({ id: "auto-session-1", status: "ACTIVE" });
  await expect(
    controller.stopAutoBetSession({ playerId: "player-1", username: "player" }),
  ).resolves.toMatchObject({ status: "STOPPED", stopReason: "MANUAL" });
});

test("maps auto bet errors to bad request responses", async () => {
  const controller = createController({
    startAutoBetSessionUseCase: {
      execute: async () => {
        throw new AutoBetSessionActiveError();
      },
    },
  });

  await expect(
    controller.startAutoBetSession(
      { playerId: "player-1", username: "player" },
      { amountCents: "1000", maxRounds: 3 },
    ),
  ).rejects.toThrow(BadRequestException);
});
```

- [ ] **Step 2: Run the controller test to verify it fails**

Run:

```bash
cd services/games && bun test tests/unit/games-controller.test.ts
```

Expected: fail with missing controller methods/constructor dependencies.

- [ ] **Step 3: Implement DTOs, mapper, HTTP error mapping, and controller routes**

Create request DTO:

```ts
import { ApiProperty, ApiPropertyOptional } from "@nestjs/swagger";

export class StartAutoBetSessionRequestDto {
  @ApiProperty({ example: "1000" })
  amountCents!: string;

  @ApiPropertyOptional({ example: 20000, nullable: true })
  autoCashoutMultiplierBp?: number | null;

  @ApiProperty({ example: 20, minimum: 1, maximum: 100 })
  maxRounds!: number;

  @ApiPropertyOptional({ example: "10000", nullable: true })
  stopLossCents?: string | null;

  @ApiPropertyOptional({ example: "15000", nullable: true })
  takeProfitCents?: string | null;
}
```

Create response DTO and mapper. The mapper must convert all bigint fields to
strings and dates to ISO strings.

Add to `GamesController` constructor:

```ts
private readonly startAutoBetSessionUseCase: StartAutoBetSessionUseCase,
private readonly getMyAutoBetSessionUseCase: GetMyAutoBetSessionUseCase,
private readonly stopAutoBetSessionUseCase: StopAutoBetSessionUseCase,
```

Add routes:

```ts
@Post("auto-bet/sessions")
@UseGuards(KeycloakJwtGuard)
@ApiBearerAuth()
async startAutoBetSession(
  @CurrentUser() user: AuthenticatedUser,
  @Body() body: StartAutoBetSessionRequestDto,
): Promise<AutoBetSessionResponseDto> {
  try {
    const session = await this.startAutoBetSessionUseCase.execute({
      playerId: user.playerId,
      username: user.username,
      amountCents: body.amountCents,
      autoCashoutMultiplierBp: body.autoCashoutMultiplierBp ?? null,
      maxRounds: body.maxRounds,
      stopLossCents: body.stopLossCents ?? null,
      takeProfitCents: body.takeProfitCents ?? null,
    });

    return toAutoBetSessionResponse(session);
  } catch (error) {
    throw toGameHttpError(error);
  }
}
```

Add corresponding `GET auto-bet/sessions/me` and `POST auto-bet/sessions/me/stop`.

Map `AutoBetSessionActiveError`, `AutoBetSessionConfigInvalidError`, and
`ManualBetBlockedByAutoBetError` to `BadRequestException` in
`toGameHttpError`.

- [ ] **Step 4: Run the focused controller test**

Run:

```bash
cd services/games && bun test tests/unit/games-controller.test.ts
```

Expected: pass.

- [ ] **Step 5: Commit**

```bash
git status --short --branch
git add services/games/src/presentation/dtos/auto-bet-session-request.dto.ts \
  services/games/src/presentation/dtos/auto-bet-session-response.dto.ts \
  services/games/src/presentation/auto-bet-session-response.mapper.ts \
  services/games/src/presentation/controllers/games.controller.ts \
  services/games/src/presentation/game-http-errors.ts \
  services/games/tests/unit/games-controller.test.ts
git commit -m "feat(games): expose auto bet session API"
```

---

### Task 3: Prisma Persistence For AutoBet

**Files:**

- Modify: `services/games/prisma/schema.prisma`
- Create: `services/games/prisma/migrations/20260602130000_add_auto_bet_sessions/migration.sql`
- Create: `services/games/src/infrastructure/prisma/auto-bet-session.mapper.ts`
- Create: `services/games/src/infrastructure/prisma/auto-bet-session-prisma.repository.ts`
- Modify: `services/games/src/app.module.ts`

- [ ] **Step 1: Update Prisma schema and migration**

Add models from the spec to `services/games/prisma/schema.prisma` with
`@@map("auto_bet_sessions")` and `@@map("auto_bet_round_executions")`.

Create migration SQL:

```sql
CREATE TABLE "auto_bet_sessions" (
  "id" TEXT NOT NULL,
  "playerId" TEXT NOT NULL,
  "username" TEXT NOT NULL,
  "status" TEXT NOT NULL,
  "amountCents" BIGINT NOT NULL,
  "autoCashoutMultiplierBp" INTEGER,
  "maxRounds" INTEGER NOT NULL,
  "roundsPlayed" INTEGER NOT NULL DEFAULT 0,
  "netProfitCents" BIGINT NOT NULL DEFAULT 0,
  "stopLossCents" BIGINT,
  "takeProfitCents" BIGINT,
  "stopReason" TEXT,
  "startsAfterRoundId" TEXT,
  "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  "updatedAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  "stoppedAt" TIMESTAMP(3),
  CONSTRAINT "auto_bet_sessions_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "auto_bet_round_executions" (
  "id" TEXT NOT NULL,
  "sessionId" TEXT NOT NULL,
  "roundId" TEXT NOT NULL,
  "betId" TEXT,
  "status" TEXT NOT NULL,
  "reason" TEXT,
  "resultStatus" TEXT,
  "resultDeltaCents" BIGINT,
  "resultAppliedAt" TIMESTAMP(3),
  "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT "auto_bet_round_executions_pkey" PRIMARY KEY ("id")
);

CREATE INDEX "auto_bet_sessions_playerId_createdAt_idx"
ON "auto_bet_sessions" ("playerId", "createdAt");

CREATE INDEX "auto_bet_sessions_status_idx"
ON "auto_bet_sessions" ("status");

CREATE UNIQUE INDEX "AutoBetSession_one_active_per_player"
ON "auto_bet_sessions" ("playerId")
WHERE "status" = 'ACTIVE';

CREATE UNIQUE INDEX "auto_bet_round_executions_sessionId_roundId_key"
ON "auto_bet_round_executions" ("sessionId", "roundId");

CREATE INDEX "auto_bet_round_executions_roundId_idx"
ON "auto_bet_round_executions" ("roundId");

ALTER TABLE "auto_bet_round_executions"
ADD CONSTRAINT "auto_bet_round_executions_sessionId_fkey"
FOREIGN KEY ("sessionId") REFERENCES "auto_bet_sessions"("id")
ON DELETE RESTRICT ON UPDATE CASCADE;
```

- [ ] **Step 2: Run Prisma generate to verify schema**

Run:

```bash
cd services/games && bun run db:generate
```

Expected: Prisma Client generated successfully.

- [ ] **Step 3: Implement mapper and repository**

Create mapper functions:

```ts
export function toAutoBetSession(record: AutoBetSessionRecord): AutoBetSession {
  return {
    id: record.id,
    playerId: record.playerId,
    username: record.username,
    status: record.status as AutoBetSessionStatus,
    amountCents: record.amountCents,
    autoCashoutMultiplierBp: record.autoCashoutMultiplierBp,
    maxRounds: record.maxRounds,
    roundsPlayed: record.roundsPlayed,
    netProfitCents: record.netProfitCents,
    stopLossCents: record.stopLossCents,
    takeProfitCents: record.takeProfitCents,
    stopReason: record.stopReason as AutoBetStopReason | null,
    startsAfterRoundId: record.startsAfterRoundId,
    createdAt: record.createdAt,
    updatedAt: record.updatedAt,
    stoppedAt: record.stoppedAt,
  };
}
```

Repository methods should use Prisma models:

- `autoBetSession.create`
- `autoBetSession.findFirst`
- `autoBetSession.findMany`
- `autoBetRoundExecution.create`
- `autoBetRoundExecution.findUnique`
- `autoBetSession.update`
- `autoBetRoundExecution.updateMany` for idempotent result application with
  `where: { id, resultAppliedAt: null }`.

- [ ] **Step 4: Wire repository in `AppModule`**

Add provider:

```ts
{
  provide: AUTO_BET_SESSION_REPOSITORY,
  useFactory: (): AutoBetSessionRepository =>
    new AutoBetSessionPrismaRepository(prismaClient),
}
```

Do not wire the executor yet.

- [ ] **Step 5: Run typecheck**

Run:

```bash
bun run check:types
```

Expected: pass.

- [ ] **Step 6: Commit**

```bash
git status --short --branch
git add services/games/prisma/schema.prisma \
  services/games/prisma/migrations/20260602130000_add_auto_bet_sessions/migration.sql \
  services/games/src/infrastructure/prisma/auto-bet-session.mapper.ts \
  services/games/src/infrastructure/prisma/auto-bet-session-prisma.repository.ts \
  services/games/src/app.module.ts
git commit -m "feat(games): persist auto bet sessions"
```

---

### Task 4: Manual Bet Blocking And AUTO_BET Source

**Files:**

- Modify: `services/games/src/application/use-cases/place-bet.use-case.ts`
- Modify: `services/games/src/app.module.ts`
- Test: `services/games/tests/unit/application/game-use-cases.test.ts`

- [ ] **Step 1: Write failing tests for manual block and auto source**

Add tests:

```ts
test("blocks manual bet placement while an auto bet session is active", async () => {
  const round = openRoundFixture();
  const gameRepository = new InMemoryGameRepository(round);
  const autoBetRepository = new FakeAutoBetSessionRepository();
  autoBetRepository.sessions.push(
    autoBetSessionFixture({ playerId: "player-1", status: "ACTIVE" }),
  );
  const useCase = createPlaceBetUseCase({
    gameRepository,
    autoBetSessionRepository: autoBetRepository,
  });

  await expect(
    useCase.execute({
      amountCents: "1000",
      playerId: "player-1",
      username: "player",
    }),
  ).rejects.toThrow(ManualBetBlockedByAutoBetError);
});

test("allows auto bet source to reuse place bet while the session is active", async () => {
  const round = openRoundFixture();
  const gameRepository = new InMemoryGameRepository(round);
  const autoBetRepository = new FakeAutoBetSessionRepository();
  autoBetRepository.sessions.push(
    autoBetSessionFixture({ playerId: "player-1", status: "ACTIVE" }),
  );
  const useCase = createPlaceBetUseCase({
    gameRepository,
    autoBetSessionRepository: autoBetRepository,
  });

  await expect(
    useCase.execute({
      amountCents: "1000",
      playerId: "player-1",
      source: "AUTO_BET",
      username: "player",
    }),
  ).resolves.toMatchObject({
    bet: { status: "ACCEPTED" },
  });
});
```

- [ ] **Step 2: Run focused tests to verify they fail**

Run:

```bash
cd services/games && bun test tests/unit/application/game-use-cases.test.ts
```

Expected: fail because `PlaceBetUseCase` has no source or auto bet repository.

- [ ] **Step 3: Modify `PlaceBetUseCase`**

Add:

```ts
type PlaceBetSource = "MANUAL" | "AUTO_BET";

type PlaceBetInput = {
  playerId: string;
  username: string;
  amountCents: bigint | number | string;
  autoCashoutMultiplierBp?: number | null;
  source?: PlaceBetSource;
};
```

Add optional constructor dependency:

```ts
private readonly autoBetSessionRepository?: Pick<
  AutoBetSessionRepository,
  "findActiveByPlayer"
>,
```

Before reading the current round:

```ts
if ((input.source ?? "MANUAL") === "MANUAL") {
  const activeAutoBetSession =
    await this.autoBetSessionRepository?.findActiveByPlayer(input.playerId);

  if (activeAutoBetSession) {
    throw new ManualBetBlockedByAutoBetError();
  }
}
```

Keep controller calls unchanged; they default to manual.

- [ ] **Step 4: Wire dependency in `AppModule`**

Inject `AUTO_BET_SESSION_REPOSITORY` into the `PlaceBetUseCase` provider and
pass it as the final constructor argument.

- [ ] **Step 5: Run focused tests**

Run:

```bash
cd services/games && bun test tests/unit/application/game-use-cases.test.ts
```

Expected: pass.

- [ ] **Step 6: Commit**

```bash
git status --short --branch
git add services/games/src/application/use-cases/place-bet.use-case.ts \
  services/games/src/app.module.ts \
  services/games/tests/unit/application/game-use-cases.test.ts
git commit -m "feat(games): block manual bets during auto bet"
```

---

### Task 5: Inline Executor For Newly Opened Rounds

**Files:**

- Create: `services/games/src/application/use-cases/execute-auto-bets-for-round.use-case.ts`
- Modify: `services/games/src/application/use-cases/advance-round-lifecycle.use-case.ts`
- Modify: `services/games/src/app.module.ts`
- Test: `services/games/tests/unit/application/game-use-cases.test.ts`

- [ ] **Step 1: Write failing executor tests**

Add tests:

```ts
test("auto bet executor skips the activation round and bets once on the next round", async () => {
  const autoBetRepository = new FakeAutoBetSessionRepository();
  const firstRound = openRoundFixture({ id: "round-1", chainIndex: 1 });
  autoBetRepository.sessions.push(
    autoBetSessionFixture({
      id: "auto-session-1",
      playerId: "player-1",
      startsAfterRoundId: "round-1",
      status: "ACTIVE",
    }),
  );
  const placeBetUseCase = {
    calls: [] as unknown[],
    execute: async (input: unknown) => {
      placeBetUseCase.calls.push(input);
      return {
        balanceCents: 99000n,
        bet: acceptedBetFixture({ id: "bet-1", roundId: "round-2" }),
      };
    },
  };
  const executor = new ExecuteAutoBetsForRoundUseCase(
    autoBetRepository,
    placeBetUseCase as never,
      new SequenceIdGenerator(["execution-1"]),
  );

  await executor.execute({ round: firstRound });
  expect(placeBetUseCase.calls).toHaveLength(0);

  await executor.execute({
    round: openRoundFixture({ id: "round-2", chainIndex: 2 }),
  });
  await executor.execute({
    round: openRoundFixture({ id: "round-2", chainIndex: 2 }),
  });

  expect(placeBetUseCase.calls).toHaveLength(1);
  expect(autoBetRepository.executions).toMatchObject([
    {
      betId: "bet-1",
      roundId: "round-2",
      sessionId: "auto-session-1",
      status: "BET_PLACED",
    },
  ]);
  expect(autoBetRepository.sessions[0].roundsPlayed).toBe(1);
});
```

- [ ] **Step 2: Run focused tests to verify they fail**

Run:

```bash
cd services/games && bun test tests/unit/application/game-use-cases.test.ts
```

Expected: fail because executor does not exist.

- [ ] **Step 3: Implement executor**

Create `ExecuteAutoBetsForRoundUseCase`:

```ts
export class ExecuteAutoBetsForRoundUseCase {
  constructor(
    private readonly autoBetSessionRepository: AutoBetSessionRepository,
    private readonly placeBetUseCase: Pick<PlaceBetUseCase, "execute">,
    private readonly idGenerator: IdGenerator,
    private readonly gameMetrics?: Pick<
      GameMetrics,
      "recordAutoBetPlaced" | "recordAutoBetFailure" | "recordAutoBetSessionStopped"
    >,
  ) {}

  async execute(input: { round: Round }): Promise<void> {
    const sessions = await this.autoBetSessionRepository.listActive();

    for (const session of sessions) {
      await this.executeSessionForRound(session, input.round);
    }
  }
}
```

`executeSessionForRound` must:

- skip `startsAfterRoundId === round.id`;
- skip if `findExecution(session.id, round.id)` exists;
- call `placeBetUseCase.execute({ source: "AUTO_BET", ... })`;
- record `BET_PLACED`;
- increment rounds;
- stop with `MAX_ROUNDS_REACHED` when reached;
- catch wallet/business failures, record `FAILED`, stop with wallet reason, and continue.

- [ ] **Step 4: Call executor after opening a round**

Modify `AdvanceRoundLifecycleUseCase` constructor with optional
`executeAutoBetsForRoundUseCase?: Pick<ExecuteAutoBetsForRoundUseCase, "execute">`.

In `openNextRound` after `saveRound(round)`:

```ts
await this.executeAutoBetsForRoundUseCase?.execute({ round });
```

- [ ] **Step 5: Wire executor in `AppModule`**

Add provider for `ExecuteAutoBetsForRoundUseCase` and inject it into
`AdvanceRoundLifecycleUseCase`.

- [ ] **Step 6: Run focused tests**

Run:

```bash
cd services/games && bun test tests/unit/application/game-use-cases.test.ts
```

Expected: pass.

- [ ] **Step 7: Commit**

```bash
git status --short --branch
git add services/games/src/application/use-cases/execute-auto-bets-for-round.use-case.ts \
  services/games/src/application/use-cases/advance-round-lifecycle.use-case.ts \
  services/games/src/app.module.ts \
  services/games/tests/unit/application/game-use-cases.test.ts
git commit -m "feat(games): execute auto bets on new rounds"
```

---

### Task 6: Apply AutoBet Results On Cashout And Settlement

**Files:**

- Create: `services/games/src/application/use-cases/apply-auto-bet-result.use-case.ts`
- Modify: `services/games/src/application/use-cases/cash-out.use-case.ts`
- Modify: `services/games/src/application/use-cases/advance-round-lifecycle.use-case.ts`
- Modify: `services/games/src/app.module.ts`
- Test: `services/games/tests/unit/application/game-use-cases.test.ts`

- [ ] **Step 1: Write failing result-application tests**

Add tests:

```ts
test("applies lost auto bet result once and stops on stop loss", async () => {
  const autoBetRepository = new FakeAutoBetSessionRepository();
  autoBetRepository.sessions.push(
    autoBetSessionFixture({
      id: "auto-session-1",
      amountCents: 1000n,
      netProfitCents: 0n,
      stopLossCents: 1000n,
      status: "ACTIVE",
    }),
  );
  autoBetRepository.executions.push(
    autoBetExecutionFixture({
      id: "execution-1",
      betId: "bet-1",
      sessionId: "auto-session-1",
    }),
  );
  const useCase = new ApplyAutoBetResultUseCase(autoBetRepository);

  await useCase.execute({
    amountCents: 1000n,
    betId: "bet-1",
    payoutCents: null,
    resultStatus: "LOST",
  });
  await useCase.execute({
    amountCents: 1000n,
    betId: "bet-1",
    payoutCents: null,
    resultStatus: "LOST",
  });

  expect(autoBetRepository.sessions[0].netProfitCents).toBe(-1000n);
  expect(autoBetRepository.sessions[0].status).toBe("STOPPED");
  expect(autoBetRepository.sessions[0].stopReason).toBe("STOP_LOSS_REACHED");
});

test("applies cashed out auto bet result and stops on take profit", async () => {
  const autoBetRepository = new FakeAutoBetSessionRepository();
  autoBetRepository.sessions.push(
    autoBetSessionFixture({
      id: "auto-session-1",
      amountCents: 1000n,
      netProfitCents: 0n,
      takeProfitCents: 500n,
      status: "ACTIVE",
    }),
  );
  autoBetRepository.executions.push(
    autoBetExecutionFixture({
      id: "execution-1",
      betId: "bet-1",
      sessionId: "auto-session-1",
    }),
  );
  const useCase = new ApplyAutoBetResultUseCase(autoBetRepository);

  await useCase.execute({
    amountCents: 1000n,
    betId: "bet-1",
    payoutCents: 1500n,
    resultStatus: "CASHED_OUT",
  });

  expect(autoBetRepository.sessions[0].netProfitCents).toBe(500n);
  expect(autoBetRepository.sessions[0].status).toBe("STOPPED");
  expect(autoBetRepository.sessions[0].stopReason).toBe("TAKE_PROFIT_REACHED");
});
```

- [ ] **Step 2: Run focused tests to verify they fail**

Run:

```bash
cd services/games && bun test tests/unit/application/game-use-cases.test.ts
```

Expected: fail because `ApplyAutoBetResultUseCase` does not exist.

- [ ] **Step 3: Implement result use case**

Create:

```ts
type ApplyAutoBetResultInput = {
  betId: string;
  amountCents: bigint;
  payoutCents: bigint | null;
  resultStatus: "LOST" | "CASHED_OUT";
};
```

Repository needs one additional method:

```ts
findExecutionByBetId(betId: string): Promise<AutoBetRoundExecution | null>;
```

Use case flow:

1. Find execution by `betId`.
2. Return if no execution.
3. Return if `resultAppliedAt` is already set.
4. Calculate delta:
   - lost: `-amountCents`
   - cashed out: `payoutCents - amountCents`
5. Call repository `applyBetResult`.
6. If updated session is active and `shouldStopForProfitLimits` returns a reason,
   stop the session.

- [ ] **Step 4: Integrate cashout and settlement**

In `CashOutUseCase`, after `round.completeCashOut` and `saveRound`, call:

```ts
await this.applyAutoBetResultUseCase?.execute({
  betId: bet.id,
  amountCents: bet.amountCents,
  payoutCents,
  resultStatus: "CASHED_OUT",
});
```

In `AdvanceRoundLifecycleUseCase.applyAutoCashouts`, after completing auto
cashout and saving, call the same use case with `CASHED_OUT`.

In `settleCrashedRound`, after `round.settle()` and `saveRound`, loop over lost
bets and call the use case with `LOST`.

- [ ] **Step 5: Wire use case in `AppModule`**

Create provider for `ApplyAutoBetResultUseCase`, inject it into `CashOutUseCase`
and `AdvanceRoundLifecycleUseCase`.

- [ ] **Step 6: Run focused tests**

Run:

```bash
cd services/games && bun test tests/unit/application/game-use-cases.test.ts
```

Expected: pass.

- [ ] **Step 7: Commit**

```bash
git status --short --branch
git add services/games/src/application/use-cases/apply-auto-bet-result.use-case.ts \
  services/games/src/application/ports/auto-bet-session.repository.ts \
  services/games/src/application/use-cases/cash-out.use-case.ts \
  services/games/src/application/use-cases/advance-round-lifecycle.use-case.ts \
  services/games/src/infrastructure/prisma/auto-bet-session-prisma.repository.ts \
  services/games/src/app.module.ts \
  services/games/tests/unit/application/game-use-cases.test.ts
git commit -m "feat(games): apply auto bet session results"
```

---

### Task 7: AutoBet Metrics

**Files:**

- Modify: `services/games/src/infrastructure/observability/game-metrics.ts`
- Modify: `services/games/src/application/use-cases/start-auto-bet-session.use-case.ts`
- Modify: `services/games/src/application/use-cases/execute-auto-bets-for-round.use-case.ts`
- Modify: `services/games/src/application/use-cases/stop-auto-bet-session.use-case.ts`
- Test: `services/games/tests/unit/infrastructure/game-metrics.test.ts`

- [ ] **Step 1: Write failing metrics tests**

Append to `game-metrics.test.ts`:

```ts
test("records auto bet metrics", async () => {
  const metrics = new GameMetrics();

  metrics.recordAutoBetSessionStarted();
  metrics.recordAutoBetPlaced();
  metrics.recordAutoBetFailure("WALLET_REJECTED");
  metrics.recordAutoBetSessionStopped("MAX_ROUNDS_REACHED");

  await expect(metrics.metricsText()).resolves.toContain(
    'crash_auto_bet_sessions_started_total 1',
  );
  await expect(metrics.metricsText()).resolves.toContain(
    'crash_auto_bets_placed_total 1',
  );
  await expect(metrics.metricsText()).resolves.toContain(
    'crash_auto_bet_failures_total{reason="WALLET_REJECTED"} 1',
  );
  await expect(metrics.metricsText()).resolves.toContain(
    'crash_auto_bet_sessions_stopped_total{reason="MAX_ROUNDS_REACHED"} 1',
  );
});
```

- [ ] **Step 2: Run focused metrics test to verify it fails**

Run:

```bash
cd services/games && bun test tests/unit/infrastructure/game-metrics.test.ts
```

Expected: fail because methods do not exist.

- [ ] **Step 3: Implement metrics**

Add counters with names from the spec:

```ts
private readonly autoBetSessionsStartedTotal = new Counter({
  name: "crash_auto_bet_sessions_started_total",
  help: "Total auto bet sessions started.",
  registers: [this.registry],
});
```

Add methods:

```ts
recordAutoBetSessionStarted(): void {
  this.autoBetSessionsStartedTotal.inc();
}

recordAutoBetSessionStopped(reason: string): void {
  this.autoBetSessionsStoppedTotal.inc({ reason });
}

recordAutoBetPlaced(): void {
  this.autoBetsPlacedTotal.inc();
}

recordAutoBetFailure(reason: string): void {
  this.autoBetFailuresTotal.inc({ reason });
}
```

Call them best-effort from start, stop, and executor use cases.

- [ ] **Step 4: Run focused metrics test**

Run:

```bash
cd services/games && bun test tests/unit/infrastructure/game-metrics.test.ts
```

Expected: pass.

- [ ] **Step 5: Commit**

```bash
git status --short --branch
git add services/games/src/infrastructure/observability/game-metrics.ts \
  services/games/src/application/use-cases/start-auto-bet-session.use-case.ts \
  services/games/src/application/use-cases/execute-auto-bets-for-round.use-case.ts \
  services/games/src/application/use-cases/stop-auto-bet-session.use-case.ts \
  services/games/tests/unit/infrastructure/game-metrics.test.ts
git commit -m "feat(observability): record auto bet metrics"
```

---

### Task 8: AutoBet E2E Through Kong And Keycloak

**Files:**

- Modify: `services/games/tests/e2e/e2e-helpers.ts`
- Create: `services/games/tests/e2e/auto-bet.e2e.test.ts`

- [ ] **Step 1: Add E2E helper functions**

Add types:

```ts
export type AutoBetSessionResponse = {
  id: string;
  playerId: string;
  username: string;
  status: "ACTIVE" | "STOPPED";
  amountCents: string;
  autoCashoutMultiplierBp: number | null;
  maxRounds: number;
  roundsPlayed: number;
  netProfitCents: string;
  stopLossCents: string | null;
  takeProfitCents: string | null;
  stopReason:
    | "MANUAL"
    | "MAX_ROUNDS_REACHED"
    | "STOP_LOSS_REACHED"
    | "TAKE_PROFIT_REACHED"
    | "WALLET_REJECTED"
    | "WALLET_UNAVAILABLE"
    | "ROUND_NOT_AVAILABLE"
    | null;
  startsAfterRoundId: string | null;
  createdAt: string;
  updatedAt: string;
  stoppedAt: string | null;
};
```

Add helpers:

```ts
export async function startAutoBetSession(
  token: string,
  body: {
    amountCents: string;
    autoCashoutMultiplierBp?: number | null;
    maxRounds: number;
    stopLossCents?: string | null;
    takeProfitCents?: string | null;
  },
): Promise<AutoBetSessionResponse> {
  return apiJson<AutoBetSessionResponse>("/games/auto-bet/sessions", {
    method: "POST",
    token,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
}

export async function getMyAutoBetSession(
  token: string,
): Promise<AutoBetSessionResponse | null> {
  return apiJson<AutoBetSessionResponse | null>(
    "/games/auto-bet/sessions/me",
    { token },
  );
}

export async function stopAutoBetSession(
  token: string,
): Promise<AutoBetSessionResponse | null> {
  return apiJson<AutoBetSessionResponse | null>(
    "/games/auto-bet/sessions/me/stop",
    { method: "POST", token },
  );
}
```

- [ ] **Step 2: Write failing E2E test**

Create `services/games/tests/e2e/auto-bet.e2e.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import {
  ensureStackIsHealthy,
  forceBettingRoundToStart,
  getAccessToken,
  getCurrentRound,
  getMyAutoBetSession,
  listMyBets,
  prepareDeterministicRound,
  startAutoBetSession,
  waitFor,
  waitForCurrentRound,
  waitForCurrentStatus,
  waitForRoundStatus,
  withE2ELock,
} from "./e2e-helpers";

describe("auto bet E2E", () => {
  test("starts next round only and stops after max rounds", async () => {
    await withE2ELock(async () => {
      await ensureStackIsHealthy();
      const token = await getAccessToken();
      const firstRound = await prepareDeterministicRound("clean-betting");

      const session = await startAutoBetSession(token, {
        amountCents: "1000",
        autoCashoutMultiplierBp: 20000,
        maxRounds: 1,
        takeProfitCents: "1000",
      });

      expect(session.startsAfterRoundId).toBe(firstRound.id);
      expect(session.status).toBe("ACTIVE");
      expect(firstRound.bets).toHaveLength(0);

      await forceBettingRoundToStart(firstRound.id);
      const runningRound = await waitForCurrentStatus("RUNNING");
      await waitForRoundStatus(runningRound.id, "SETTLED");

      const nextRound = await waitForCurrentRound();
      await waitFor(async () => {
        const round = await getCurrentRound();
        const bets = round?.bets ?? [];
        return bets.some((bet) => bet.playerId === session.playerId) ? bets : null;
      }, "auto bet placement");

      const bets = await listMyBets(token, 5);
      expect(bets.some((bet) => bet.roundId === nextRound.id)).toBe(true);

      const stopped = await waitFor(async () => {
        const current = await getMyAutoBetSession(token);
        return current?.status === "STOPPED" ? current : null;
      }, "auto bet session stopped");

      expect(stopped.stopReason).toBe("MAX_ROUNDS_REACHED");
      expect(stopped.roundsPlayed).toBe(1);
    });
  });
});
```

If `waitFor` is not exported from helpers, export it.

- [ ] **Step 3: Run E2E to verify it fails before full wiring**

Run:

```bash
cd services/games && bun test tests/e2e/auto-bet.e2e.test.ts
```

Expected: fail until migration, routes, and executor are fully wired in Docker.

- [ ] **Step 4: Rebuild stack and run E2E**

Run:

```bash
docker compose up -d --build
bun scripts/ci/check-kong-health.ts
cd services/games && bun test tests/e2e/auto-bet.e2e.test.ts
```

Expected: pass.

- [ ] **Step 5: Commit**

```bash
git status --short --branch
git add services/games/tests/e2e/e2e-helpers.ts \
  services/games/tests/e2e/auto-bet.e2e.test.ts
git commit -m "test(games): cover auto bet e2e flow"
```

---

### Task 9: Final Regression Gates And PR Preparation

**Files:**

- Modify if needed: `docs/superpowers/specs/2026-05-31-crash-game-implementation-issue-log.md` for any real issues encountered.

- [ ] **Step 1: Run full local gates**

Run:

```bash
bun run lint
bun run check:types
bun run test:unit
cd frontend && bun run test
cd ..
cd frontend && bun run build
cd ..
bun run test:coverage && bun run quality:gate
docker compose config
docker compose up -d --build
bun scripts/ci/check-kong-health.ts
bun scripts/ci/check-observability-health.ts
cd services/games && bun run test:e2e
cd ../..
bun run test:e2e:browser
git diff --check
```

Expected: all pass. If any gate fails, diagnose root cause, add the smallest
fix, and rerun the relevant focused test before rerunning the failed gate.

- [ ] **Step 2: Review quality metrics**

Run:

```bash
bun run quality:gate
```

Expected: `security.high` remains `0`, no regressions or warnings. Do not update
`quality/baseline.json` unless the user explicitly approves.

- [ ] **Step 3: Commit any issue-log-only fixes if needed**

Only if a real implementation issue was encountered and documented:

```bash
git status --short --branch
git add docs/superpowers/specs/2026-05-31-crash-game-implementation-issue-log.md
git commit -m "docs: log auto bet implementation issues"
```

Do not commit `docs/superpowers/specs/2026-06-02-auto-bet-backend-design.md`
unless the user explicitly asks.

- [ ] **Step 4: Push and create PR only after user asks**

When the user asks for PR:

```bash
git push origin crash-game-implementation
gh pr create \
  --base main \
  --head crash-game-implementation \
  --title "feat(games): add backend auto bet sessions" \
  --body "Adds backend-only auto bet sessions with fixed bet amount, next-round start, safety stops, metrics, and E2E coverage."
```

Then use `gh-pr-babysit` to monitor Actions until green.

---

## Self-Review Checklist

- Spec coverage:
  - Persistent sessions: Tasks 1 and 3.
  - One active session per player: Tasks 1 and 3.
  - Start next round only: Tasks 1, 5, and 8.
  - Fixed amount, auto cashout, max rounds, stop-loss, take-profit: Tasks 1 and 6.
  - Manual bet block: Task 4.
  - Manual cashout allowed: Task 6 keeps cashout path and only applies result.
  - Metrics: Task 7.
  - E2E: Task 8.
- Incomplete-instruction scan: no unfinished instructions are intended.
- Type consistency:
  - `AutoBetSession`, `AutoBetRoundExecution`, and repository method names are
    consistent across tasks.
  - `source: "MANUAL" | "AUTO_BET"` is introduced before executor usage.
  - Result idempotency uses `resultAppliedAt` in both schema and use case.
