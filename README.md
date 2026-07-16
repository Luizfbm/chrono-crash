# Chrono Crash

A multiplayer crash game. A multiplier climbs from `1.00x` and stops at a point that was already decided before the betting window opened, and that you can verify yourself once the round is over. You bet, then cash out before the crash or lose the bet.

Under the hood it is two NestJS services with separate databases. They settle money through RabbitMQ and push round state to every connected client over WebSocket.

## Quickstart

You need Bun 1.x and Docker with Compose.

```bash
bun install
bun run docker:up
```

Open `http://localhost:8000` and sign in as `player` / `player123`. The account starts with R$ 1.000,00.

There are no manual setup steps. Keycloak imports its realm on boot, Kong loads its config, and both services run `prisma migrate deploy` before serving traffic. `docker:up` stays in the foreground; add `-d` to the compose command if you want it detached.

```bash
bun run docker:down    # stop the stack
bun run docker:prune   # wipe containers, volumes and local images
```

## Stack

| Layer | Choice |
| --- | --- |
| Runtime | Bun |
| Backend | NestJS + TypeScript strict |
| Databases | PostgreSQL, separate `games` and `wallets` databases |
| ORM | Prisma |
| Messaging | RabbitMQ |
| API Gateway | Kong DB-less |
| IdP | Keycloak, realm `crash-game` |
| Realtime | Socket.IO via `@nestjs/websockets` |
| Frontend | Vite + React + Tailwind CSS v4 |
| Client state | TanStack Query + Zustand |
| Tests | Bun test, Vitest, Playwright |
| API docs | Swagger/OpenAPI |
| Observability | Prometheus, Grafana, Jaeger, OpenTelemetry |

## Features

- Round lifecycle: betting window, multiplier climb, crash, settlement.
- Provably fair crash points that any player can verify after the round.
- Realtime sync over WebSocket. Two tabs show the same round at the same time.
- Transactional outbox in the game and inbox in the wallet, so money commands survive redelivery without double-spending.
- Auto cashout at a target multiplier, configured per bet.
- Auto bet with fixed and Martingale strategies, plus stop-loss and take-profit.
- Leaderboard ranked by net profit.
- Rate limiting on bet, cashout and wallet routes at the gateway.
- Metrics, dashboards and traces for bet volume, payouts, RTP and event latency.
- Deterministic seeding for reproducible E2E runs.

## Architecture

```text
Frontend React/Vite
  | HTTP REST + WebSocket
Kong API Gateway
  | REST                         | REST
Game Service (NestJS)            Wallet Service (NestJS)
  | PostgreSQL games             | PostgreSQL wallets
  |                              |
  +------ RabbitMQ wallet.commands ------+

Keycloak issues JWTs; both services validate Bearer tokens via JWKS.
```

The game service owns rounds, bets, cashouts, the leaderboard, auto bet, provably fair and realtime events. The wallet service owns balances, the ledger and financial idempotency. Each service has its own database and neither reads the other's tables.

Both follow the same layering: `domain`, `application`, `infrastructure`, `presentation`.

## Design decisions

### The game never writes a balance

Every debit and credit goes through RabbitMQ as a `wallet.debit` or `wallet.credit` command. The game writes the command to an outbox table in the same transaction that records the bet; the wallet keeps an inbox so a redelivered command is processed once. This is what keeps the two services consistent without a distributed transaction or an HTTP call between them.

I used request/reply over the broker for interactive operations like placing a bet, since the player needs an answer. The alternative was exposing the wallet over HTTP to the game, which would have coupled them.

### Money is never a float

Balances and bets are `BigInt`/`BIGINT` in cents, and the API serializes them as strings so JSON cannot round them. `100` is R$ 1,00. Minimum bet is `100` cents, maximum is `100000`.

### Multipliers are basis points

`10000` is `1.00x` and `25000` is `2.50x`. Payout is `floor(amountCents * multiplierBp / 10000)`, integer math start to finish.

### Provably fair

Each round commits before it reveals. `serverSeedHash` is public before and during the round; `serverSeed` and `crashPointBp` only appear after the crash. The crash point comes from a hash chain and HMAC SHA-256 over the client seed, a nonce and the house edge. `GET /games/rounds/:roundId/verify` recalculates the result so a player does not have to trust the server.

Start here to read it:

- `services/games/src/domain/provably-fair.ts`
- `services/games/src/application/use-cases/verify-round.use-case.ts`
- `services/games/tests/unit/domain/game-domain.test.ts`

## API

Everything is reachable through Kong at `http://localhost:8000`.

### Wallet

| Method | Route | Auth | Description |
| --- | --- | --- | --- |
| `POST` | `/wallets` | yes | Create the authenticated player's wallet |
| `GET` | `/wallets/me` | yes | Balance of the authenticated player |
| `GET` | `/wallets/health` | no | Healthcheck |
| `GET` | `/wallets/metrics` | no | Prometheus metrics |

Debit and credit are not REST routes. They only happen over RabbitMQ.

### Game

| Method | Route | Auth | Description |
| --- | --- | --- | --- |
| `GET` | `/games/rounds/current` | no | Current round |
| `GET` | `/games/rounds/history` | no | Round history |
| `GET` | `/games/rounds/:roundId/verify` | no | Provably fair verification |
| `GET` | `/games/bets/me` | yes | Player's bet history |
| `POST` | `/games/bet` | yes | Place a bet |
| `POST` | `/games/bet/cashout` | yes | Cash out |
| `GET` | `/games/leaderboard` | no | Ranking by net profit |
| `POST` | `/games/auto-bet/sessions` | yes | Start auto bet |
| `GET` | `/games/auto-bet/sessions/me` | yes | Read the player's auto bet |
| `POST` | `/games/auto-bet/sessions/me/stop` | yes | Stop the active auto bet |
| `GET` | `/games/health` | no | Healthcheck |
| `GET` | `/games/metrics` | no | Prometheus metrics |

### WebSocket

The socket is server-to-client only. Bets and cashouts stay on REST, and a bet command sent over the socket does not create a bet.

Namespace `/games`, path `/games/socket.io` through Kong. Emitted events:

`round.snapshot`, `round.betting_started`, `round.started`, `round.tick`, `round.crashed`, `round.settled`, `bet.placed`, `bet.cashed_out`.

## Frontend

The app opens straight into the game, no landing page.

- Keycloak login with authorization code flow and PKCE.
- A 3D stage with the car, the portal, the multiplier curve and the round states.
- Bet controls, cashout, auto cashout and Martingale auto bet.
- Round history, leaderboard, live bet table and a provably fair tab.
- Dark arcade casino look in felt green and gold.
- Layouts checked by Playwright on mobile, tablet, laptop and desktop.

Entry points:

- `frontend/src/components/game/game-dashboard-shell.tsx`
- `frontend/src/components/game/chrono-stage.tsx`
- `frontend/src/hooks/use-game-realtime.ts`
- `frontend/src/stores/game-realtime-store.ts`

## Local dev shortcuts

With the stack running, the app has a side rail that collects the links you actually need while developing. The same list, if you prefer the URL bar:

| What | URL | Credentials |
| --- | --- | --- |
| App via Kong | `http://localhost:8000/` | `player` / `player123` |
| Game Swagger | `http://localhost:4001/docs` | public |
| Wallet Swagger | `http://localhost:4002/docs` | public |
| Game metrics | `http://localhost:8000/games/metrics` | public |
| Wallet metrics | `http://localhost:8000/wallets/metrics` | public |
| Grafana | `http://localhost:3001` | `admin` / `admin` |
| Prometheus | `http://localhost:9090` | public |
| Jaeger | `http://localhost:16686` | public |
| RabbitMQ Management | `http://localhost:15672` | `admin` / `admin` |
| Keycloak admin | `http://localhost:8080/admin/master/console/` | `admin` / `admin` |

## Testing

```bash
bun run ci:local         # lint, typecheck, unit, coverage, quality gate, compose config
bun run ci:e2e           # boots the stack, then API E2E against it
bun run test:e2e:browser # Playwright against http://localhost:8000
```

Focused runs:

```bash
cd services/games && bun test tests/unit
cd services/wallets && bun test tests/unit
cd services/games && bun test tests/e2e    # needs the stack up
cd frontend && bun run test
```

The quality gate compares coverage and duplication against `quality/baseline.json` and fails if either regresses. Playwright runs with `workers: 1` because every test shares one Docker stack, one Keycloak and one `player` user.

## Code map

- Compose: `docker-compose.yml`
- Kong: `docker/kong/kong.yml`
- Keycloak realm: `docker/keycloak/realm-export.json`
- Schemas: `services/games/prisma/schema.prisma`, `services/wallets/prisma/schema.prisma`
- Controllers: `services/games/src/presentation/controllers/games.controller.ts`, `services/wallets/src/presentation/controllers/wallets.controller.ts`
- Realtime gateway: `services/games/src/presentation/realtime/rounds.gateway.ts`
- Wallet messaging: `services/wallets/src/infrastructure/messaging/wallet-command-handler.ts`
- Browser E2E: `tests/browser/player-flow.spec.ts`
- Quality gate: `quality/baseline.json`, `scripts/quality`

## Trade-offs and what is missing

The frontend container runs the Vite dev server. That is fine for running this locally and wrong for production.

The `/metrics` routes are open. They expose no tokens, secrets or database URLs, but a real deployment would put them behind auth.

Auth is Keycloak rather than something hand-rolled, so there is no password handling in this codebase at all.

Not built: Storybook, and sound effects for bet, cashout and crash.
