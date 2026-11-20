# Crash Game Auto Bet Backend Design

Date: 2026-06-02
Status: Pending written spec review

## Summary

Implement the README bonus `Auto bet` as a backend-only first slice. A player
can create one persistent auto bet session that automatically places fixed-size
bets on future betting rounds until a configured safety limit stops the
session.

This slice deliberately avoids frontend work and Martingale. It establishes a
stable REST contract, persistence model, execution flow, idempotency rules, and
test coverage so the frontend can be added in a later PR without moving game
rules into UI code.

## Goals

- Add authenticated REST endpoints to start, inspect, and stop auto bet
  sessions.
- Persist auto bet sessions in the Game Service PostgreSQL database.
- Allow at most one active auto bet session per player.
- Start automation only from the next opened `BETTING` round, not the current
  round.
- Support fixed bet amount, optional per-bet auto cashout target, maximum
  number of rounds, optional stop-loss, and optional take-profit.
- Reuse the existing bet placement, wallet debit outbox, realtime event, and
  auto cashout behavior.
- Stop sessions automatically on configured limits, wallet rejection, wallet
  unavailability, or manual stop.
- Block manual `POST /games/bet` while the player has an active auto bet
  session.
- Keep manual cashout available for bets that were placed by auto bet.
- Preserve provably fair behavior, wallet consistency, observability, quality
  gates, and existing tests.

## Non-Goals

- No frontend controls in this slice.
- No Martingale, progressive betting, or multiple betting strategies.
- No infinite auto bet sessions.
- No multiple active sessions per player.
- No auto restart after a session stops.
- No websocket-specific auto bet events in this slice.
- No wallet schema changes.
- No deletion, disabling, skipping, or reduction of existing tests.

## Product Contract

A player starts a session with fixed configuration:

```json
{
  "amountCents": "1000",
  "autoCashoutMultiplierBp": 20000,
  "maxRounds": 20,
  "stopLossCents": "10000",
  "takeProfitCents": "15000"
}
```

Rules:

- `amountCents` follows the same range as manual bets: `100` to `100000`.
- `autoCashoutMultiplierBp` is optional and follows the existing auto cashout
  range: `10100` to `10000000`.
- `maxRounds` is required and must be an integer from `1` to `100`.
- `stopLossCents` is optional and must be a positive integer when provided.
- `takeProfitCents` is optional and must be a positive integer when provided.
- At least `maxRounds` is always present, so the first slice never creates an
  unbounded session.
- If a current round exists when the session is created, the session stores
  `startsAfterRoundId` and skips that round.
- If no current round exists, `startsAfterRoundId` is `null` and the session may
  run on the next opened betting round.

Manual bet behavior:

- `POST /games/bet` with `source = "MANUAL"` rejects while the player has an
  active auto bet session.
- Internal auto bet placement uses `source = "AUTO_BET"` and can reuse the same
  `PlaceBetUseCase` without being blocked by the manual guard.
- Manual cashout remains allowed while an auto bet placed by the session is
  still `ACCEPTED`.

## API Contract

All endpoints live under the existing Game Service route and are protected by
the Keycloak JWT guard.

### Start Session

```http
POST /games/auto-bet/sessions
Authorization: Bearer <token>
Content-Type: application/json

{
  "amountCents": "1000",
  "autoCashoutMultiplierBp": 20000,
  "maxRounds": 20,
  "stopLossCents": "10000",
  "takeProfitCents": "15000"
}
```

Response:

```ts
type AutoBetSessionResponse = {
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
  stopReason: AutoBetStopReason | null;
  startsAfterRoundId: string | null;
  createdAt: string;
  updatedAt: string;
  stoppedAt: string | null;
};
```

Invalid configuration returns `400 Bad Request`.

Starting a second active session for the same player returns `400 Bad Request`
with an application error code such as `AUTO_BET_SESSION_ACTIVE`.

### Inspect Session

```http
GET /games/auto-bet/sessions/me
Authorization: Bearer <token>
```

Returns the active session when one exists. Otherwise returns the player's most
recent stopped session. If the player has no auto bet history, returns `null`.

This gives the later frontend enough state to recover after a browser reload.

### Stop Session

```http
POST /games/auto-bet/sessions/me/stop
Authorization: Bearer <token>
```

Stopping is idempotent. If an active session exists, it is stopped with
`stopReason = "MANUAL"` and returned. If no active session exists, the endpoint
returns the most recent stopped session or `null`.

## Domain Model

Auto bet introduces one aggregate-like application model owned by the Game
bounded context:

```ts
type AutoBetSessionStatus = "ACTIVE" | "STOPPED";

type AutoBetStopReason =
  | "MANUAL"
  | "MAX_ROUNDS_REACHED"
  | "STOP_LOSS_REACHED"
  | "TAKE_PROFIT_REACHED"
  | "WALLET_REJECTED"
  | "WALLET_UNAVAILABLE"
  | "ROUND_NOT_AVAILABLE";
```

Session state:

- `roundsPlayed` increments only after an auto bet is accepted for a round.
- `netProfitCents` starts at `0`.
- A lost auto bet contributes `-amountCents`.
- A cashed-out auto bet contributes `payoutCents - amountCents`.
- Profit and loss limits are evaluated after the bet result is known.
- `maxRounds` is evaluated after a bet is accepted and again after the bet
  result is applied. This lets a session stop immediately after placing its
  final configured bet while still allowing that bet to settle normally.

## Persistence Design

Add `AutoBetSession` to the `games` Prisma schema:

```prisma
model AutoBetSession {
  id                       String    @id
  playerId                 String
  username                 String
  status                   String
  amountCents              BigInt
  autoCashoutMultiplierBp  Int?
  maxRounds                Int
  roundsPlayed             Int       @default(0)
  netProfitCents           BigInt    @default(0)
  stopLossCents            BigInt?
  takeProfitCents          BigInt?
  stopReason               String?
  startsAfterRoundId       String?
  createdAt                DateTime  @default(now())
  updatedAt                DateTime  @updatedAt
  stoppedAt                DateTime?

  executions               AutoBetRoundExecution[]

  @@index([playerId, createdAt])
  @@index([status])
}
```

Add `AutoBetRoundExecution`:

```prisma
model AutoBetRoundExecution {
  id                String    @id
  sessionId         String
  roundId           String
  betId             String?
  status            String
  reason            String?
  resultStatus      String?
  resultDeltaCents  BigInt?
  resultAppliedAt   DateTime?
  createdAt         DateTime  @default(now())

  session           AutoBetSession @relation(fields: [sessionId], references: [id])

  @@unique([sessionId, roundId])
  @@index([roundId])
}
```

Create a PostgreSQL partial unique index in the migration:

```sql
CREATE UNIQUE INDEX "AutoBetSession_one_active_per_player"
ON "AutoBetSession" ("playerId")
WHERE "status" = 'ACTIVE';
```

The partial index is the database-level backstop for the "one active session per
player" rule.

## Backend Design

Add an application port:

```ts
interface AutoBetSessionRepository {
  create(input: CreateAutoBetSessionInput): Promise<AutoBetSession>;
  findActiveByPlayer(playerId: string): Promise<AutoBetSession | null>;
  findLatestByPlayer(playerId: string): Promise<AutoBetSession | null>;
  listActive(): Promise<AutoBetSession[]>;
  stop(input: StopAutoBetSessionInput): Promise<AutoBetSession>;
  recordExecution(input: RecordAutoBetExecutionInput): Promise<void>;
  findExecution(sessionId: string, roundId: string): Promise<AutoBetRoundExecution | null>;
  incrementRoundsPlayed(sessionId: string): Promise<AutoBetSession>;
  applyBetResult(input: ApplyAutoBetResultInput): Promise<AutoBetSession>;
}
```

Use cases:

- `StartAutoBetSessionUseCase`
- `GetMyAutoBetSessionUseCase`
- `StopAutoBetSessionUseCase`
- `ExecuteAutoBetsForRoundUseCase`
- `ApplyAutoBetResultUseCase`

`PlaceBetUseCase` gains an optional source:

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

Default source is `MANUAL`. The manual path checks the auto bet repository and
rejects when an active session exists. The auto bet executor calls the same use
case with `source: "AUTO_BET"`.

## Execution Flow

Inline executor:

1. `AdvanceRoundLifecycleUseCase` opens a new `BETTING` round.
2. It saves the round.
3. It calls `ExecuteAutoBetsForRoundUseCase.execute({ round })`.
4. The executor lists active sessions.
5. It skips sessions where `startsAfterRoundId === round.id`.
6. It skips sessions with an existing `(sessionId, roundId)` execution.
7. It calls `PlaceBetUseCase` with `source: "AUTO_BET"`.
8. It records `BET_PLACED` execution with `betId`.
9. It increments `roundsPlayed`.
10. If `roundsPlayed >= maxRounds`, it stops the session with
    `MAX_ROUNDS_REACHED`.

If wallet debit is rejected, the executor records a failed execution and stops
the session:

- `WALLET_REJECTED` for known business rejections such as insufficient funds.
- `WALLET_UNAVAILABLE` for timeout, outbox dispatch failure, or unexpected
  wallet infrastructure failure.

The lifecycle should continue processing other sessions when one session fails.
One player's auto bet failure must not prevent the round from opening or block
other players.

## Applying Results

When a bet reaches a terminal result, the Game Service updates the owning auto
bet session if the bet came from a session execution.

Terminal results:

- `LOST`: delta is `-amountCents`.
- `CASHED_OUT`: delta is `payoutCents - amountCents`.

Implementation points:

- After settlement marks accepted bets as lost, call
  `ApplyAutoBetResultUseCase` for lost auto bet executions.
- After manual or automatic cashout completes wallet credit, call
  `ApplyAutoBetResultUseCase` for cashed-out auto bet executions.
- The result application must be idempotent. Re-running settlement or retrying
  pending cashout credit must not apply the same profit/loss twice.
- `ApplyAutoBetResultUseCase` no-ops when `resultAppliedAt` is already set on
  the execution row.

After applying a result:

- Stop with `STOP_LOSS_REACHED` when
  `stopLossCents !== null && netProfitCents <= -stopLossCents`.
- Stop with `TAKE_PROFIT_REACHED` when
  `takeProfitCents !== null && netProfitCents >= takeProfitCents`.
- Do not reopen a stopped session.

## Error Handling

New game errors:

- `AutoBetSessionActiveError`
- `AutoBetSessionConfigInvalidError`
- `ManualBetBlockedByAutoBetError`

HTTP mapping:

- Invalid config: `400 Bad Request`
- Duplicate active session: `400 Bad Request`
- Manual bet blocked by active auto bet: `400 Bad Request`
- Missing or invalid auth: existing `401 Unauthorized`

Executor failures are recorded on the session and execution rows. They should
not be returned to a browser request unless the user later inspects the session.

## Observability

Add Game Service metrics:

- `crash_auto_bet_sessions_started_total`
- `crash_auto_bet_sessions_stopped_total{reason}`
- `crash_auto_bets_placed_total`
- `crash_auto_bet_failures_total{reason}`

Metrics are best-effort and must not affect gameplay, matching the existing
metrics style.

The existing Grafana dashboard does not need to be changed in this backend-only
slice unless the implementation plan explicitly scopes it in later.

## Testing Strategy

Use TDD in vertical slices. Tests should verify behavior through public
interfaces, stable application use cases, or E2E flows.

Backend unit/application tests:

- Start use case creates an active persisted session.
- Start use case rejects a second active session for the same player.
- Start use case validates amount, auto cashout target, max rounds, stop-loss,
  and take-profit.
- Manual place bet rejects when the player has an active auto bet session.
- Auto bet source can reuse `PlaceBetUseCase` while a session is active.
- Executor skips the current round saved in `startsAfterRoundId`.
- Executor places exactly one bet in the next opened betting round.
- Executor does not duplicate a bet when called twice for the same round.
- Executor stops after reaching `maxRounds`.
- Executor stops on wallet rejection or wallet unavailability.
- Result application updates `netProfitCents` for lost bets.
- Result application updates `netProfitCents` for cashed-out bets.
- Result application is idempotent.
- Session stops on stop-loss.
- Session stops on take-profit.

Controller tests:

- `POST /games/auto-bet/sessions` returns the response DTO.
- `GET /games/auto-bet/sessions/me` returns active, latest stopped, or `null`.
- `POST /games/auto-bet/sessions/me/stop` is idempotent.
- Invalid request payloads map to `400`.

E2E tests:

- Create a session through Kong with a real Keycloak token.
- Confirm it does not bet in the current betting round.
- Advance to the next deterministic round and confirm an automatic bet is
  placed.
- Confirm repeated lifecycle calls do not create duplicate bets.
- Confirm the session stops by `maxRounds` or `takeProfit`.

No frontend tests are required in this slice because frontend code is out of
scope.

## Required Gates Before Commit

The project rule is that all applicable tests and gates must pass before any
implementation commit:

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

If a gate is not applicable to a docs-only commit or a focused intermediate
checkpoint, call that out explicitly before committing.

## Rollout

Deliver backend auto bet as one bonus slice after this spec and the
implementation plan are approved:

1. Persistence and repository.
2. Start, inspect, and stop use cases plus controller contract.
3. Manual bet blocking and `PlaceBetUseCase` source.
4. Inline executor on newly opened betting rounds.
5. Result application and stop reasons.
6. Observability metrics.
7. E2E and full gate verification.

The follow-up frontend slice will consume the REST contract and add controls to
configure, monitor, and stop auto bet sessions. Martingale remains a separate
future slice after the fixed-value backend and frontend flows are stable.
