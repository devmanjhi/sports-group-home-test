# Jackpot Service

A Spring Boot backend that receives bets, contributes each one to a matching jackpot pool, and
evaluates bets for a jackpot reward.

- **Java 21**, **Spring Boot 3.5**, **Gradle** (wrapper included)
- **H2 in-memory** database for bets, jackpots, contributions and rewards
- **Kafka** via Spring Kafka, with a **mock publisher** so the service runs with no broker at all

---

## Quick start

Requires only a JDK 21+. The Gradle wrapper downloads Gradle itself on first run, so there is
nothing else to install — no database, no broker, no Docker.

```bash
./gradlew bootRun
```

The service starts on `http://localhost:8080` and seeds two jackpots. In another terminal:

```bash
./demo.sh
```

`demo.sh` walks through the whole flow: publishing bets, watching the fixed and variable
contribution rates, winning a jackpot, and seeing the pool reset. To run the tests:

```bash
./gradlew test
```

To build the runnable jar (`build/libs/jackpot-service-1.0.0.jar`):

```bash
./gradlew build
```

---

## What the service does

### Use case 1 and 2: publish a bet, then consume it

`POST /api/v1/bets` validates the bet, stores it, and publishes it to the **`jackpot-bets`**
topic. It returns `202 Accepted`, because the contribution happens on the consumer side. The
consumer reads the topic and hands each bet to the contribution logic.

### Use case 3: every bet contributes to a matching jackpot

The consumer looks up the jackpot by the bet's jackpot id. If one matches, it contributes to the
pool according to **that jackpot's own configuration** and writes a contribution record holding
the bet id, user id, jackpot id, stake amount, contribution amount, resulting pool amount and
creation time. If no jackpot matches, nothing is contributed and the bet is marked
`NO_MATCHING_JACKPOT`.

Two contribution models ship today:

| Model | Behaviour |
| --- | --- |
| `FIXED` | A constant percentage of the stake. |
| `VARIABLE` | Starts high and falls at a fixed rate as the pool grows, down to a floor. |

The variable rate is:

```
growth     = currentPool - initialPool
steps      = growth / poolStepAmount
percentage = max(minPercentage, startPercentage - decayPercentagePerStep * steps)
```

So a jackpot configured to start at 10% and lose 1 percentage point per 10,000 of growth
contributes 10% while the pool is fresh, 9% once it has grown by 10,000, and 7.5% at 25,000 of
growth. An empty jackpot fills quickly while a large one grows slowly.

### Use case 4: evaluate a bet for the jackpot reward

`POST /api/v1/bets/{betId}/evaluate` checks whether a **contributing** bet wins. It computes the
win chance from the jackpot's reward configuration and rolls against it. On a win the entire pool
is paid out, a reward record is stored, and **the pool is reset to its initial value**.

Two reward models ship today:

| Model | Behaviour |
| --- | --- |
| `FIXED` | A constant win chance. |
| `VARIABLE` | Starts low and grows as the pool grows; becomes 100% at a pool limit. |

The variable chance is:

```
if currentPool >= guaranteedPoolLimit -> 100%
growth = currentPool - initialPool
steps  = growth / poolStepAmount
chance = min(100, startChance + chanceGrowthPercentagePerStep * steps)
```

So a jackpot starting at a 1% chance and gaining 5 percentage points per 10,000 of growth wins
1% of the time when fresh, 13.5% at 25,000 of growth, and always once the pool hits its limit.
That ceiling guarantees a jackpot cannot grow forever.

---

## Configured jackpots

Jackpots are defined in `src/main/resources/application.yml` under `jackpot.definitions` and
seeded into H2 at startup. Adding or retuning a jackpot needs no code change.

| Jackpot | Initial pool | Contribution | Reward |
| --- | --- | --- | --- |
| `jackpot-fixed-1` | 10,000.00 | `FIXED`, flat 5% of the stake | `FIXED`, flat 10% chance |
| `jackpot-variable-1` | 50,000.00 | `VARIABLE`, 10% decaying by 1pp per 10,000, floor 1% | `VARIABLE`, 1% growing by 5pp per 10,000, guaranteed at 100,000 |

---

## API

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/api/v1/bets` | Publish a bet to the `jackpot-bets` topic (use case 1) |
| `POST` | `/api/v1/bets/{betId}/evaluate` | Evaluate the bet for a jackpot reward (use case 4) |
| `GET` | `/api/v1/bets/{betId}` | What the service did with a bet |
| `GET` | `/api/v1/jackpots` | All jackpots with their current rate and chance |
| `GET` | `/api/v1/jackpots/{jackpotId}` | One jackpot |
| `GET` | `/api/v1/jackpots/{jackpotId}/contributions` | That jackpot's contribution records |

The H2 console is available at `http://localhost:8080/h2-console`
(JDBC URL `jdbc:h2:mem:jackpot`, user `sa`, empty password) to browse the tables directly.

### Publish a bet

```bash
curl -X POST http://localhost:8080/api/v1/bets \
  -H 'Content-Type: application/json' \
  -d '{"betId":"bet-001","userId":"user-42","jackpotId":"jackpot-fixed-1","betAmount":200.00}'
```

```json
{
  "betId": "bet-001",
  "userId": "user-42",
  "jackpotId": "jackpot-fixed-1",
  "betAmount": 200.00,
  "status": "CONTRIBUTED",
  "createdAt": "2026-09-13T10:36:46.832654Z",
  "evaluatedAt": null
}
```

The jackpot pool has now grown by 5% of 200.00:

```bash
curl http://localhost:8080/api/v1/jackpots/jackpot-fixed-1
```

```json
{
  "jackpotId": "jackpot-fixed-1",
  "name": "Daily Fixed Jackpot",
  "initialPoolAmount": 10000.00,
  "currentPoolAmount": 10010.00,
  "contributionType": "FIXED",
  "currentContributionPercentage": 5.0000,
  "rewardType": "FIXED",
  "currentWinChancePercentage": 10.0000
}
```

### Evaluate a bet for the reward

```bash
curl -X POST http://localhost:8080/api/v1/bets/bet-001/evaluate
```

A win pays out the whole pool and resets it, which `jackpotPoolAmount` shows:

```json
{
  "betId": "bet-001",
  "userId": "user-42",
  "jackpotId": "jackpot-fixed-1",
  "won": true,
  "rewardAmount": 10010.00,
  "winChancePercentage": 10.0000,
  "jackpotPoolAmount": 10000.00,
  "alreadyEvaluated": false,
  "evaluatedAt": "2026-09-13T10:36:58.772649Z"
}
```

A loss returns the same shape with `"won": false` and a `rewardAmount` of `0.00`.

### Error responses

| Status | When |
| --- | --- |
| `400` | The bet failed validation; the failing fields are listed in `details` |
| `404` | The bet or jackpot does not exist |
| `409` | That bet id has already been published |
| `422` | The bet exists but never contributed, so it cannot win a jackpot |
| `503` | The bet could not be published to Kafka |

---

## Running with real Kafka

The service defaults to the mock publisher, which the assignment allows: it logs the payload and
hands the bet to the same consumer logic a real listener would call.

```
[MOCK KAFKA] topic=jackpot-bets key=bet-001 payload=BetMessage[betId=bet-001, ...]
```

To run against a real broker instead, start one and flip a single flag:

```bash
docker compose up -d
./gradlew bootRun --args='--jackpot.kafka.enabled=true'
```

`jackpot.kafka.enabled` swaps `MockBetPublisher` for `KafkaBetPublisher` and activates
`BetKafkaListener`, which consumes `jackpot-bets`. Neither the controller nor the service layer
changes, because both publishers implement the same `BetPublisher` interface and both paths end
in the same `BetProcessingService`.

The one behavioural difference: with a real broker the flow is genuinely asynchronous, so a bet
is briefly `PUBLISHED` before becoming `CONTRIBUTED`. With the mock it is contributed in-process
before the API responds, which keeps the service runnable and deterministically testable with no
infrastructure.

---

## Design notes

### Structure

```
api/          controllers, request/response DTOs, exception handling
service/      use case orchestration
  contribution/  contribution strategies + registry
  reward/        reward chance strategies, registry, randomness
messaging/    bet payload, publisher abstraction, Kafka and mock implementations
domain/       JPA entities and per-jackpot configuration
repository/   Spring Data repositories
config/       externalised configuration and startup seeding
```

### Adding a third contribution or reward model

This is the main extension point the requirements ask for, so it costs two steps and touches no
existing logic:

1. Add a constant to `ContributionType` (or `RewardType`).
2. Add a `@Component` implementing `ContributionCalculator` (or `RewardChanceCalculator`) that
   returns that constant from `type()`.

`ContributionCalculators` and `RewardChanceCalculators` receive every implementation Spring finds
and index them by type, so nothing else needs editing. A type with no calculator, or two
calculators claiming the same type, fails loudly at startup rather than silently at runtime.

### Correctness decisions

- **Concurrency.** Contributing to a pool and paying one out are both read-modify-write cycles,
  so the jackpot row is loaded with a `PESSIMISTIC_WRITE` lock. Without it, concurrent bets on
  the same jackpot would lose contributions.
- **Transactions.** A contribution's pool increase and its contribution record commit together.
  A payout's reward record and pool reset likewise.
- **Idempotent consumer.** Kafka delivers at least once, so `bet_id` is unique on the
  contribution table and a redelivered bet is detected and skipped. A bet can never contribute
  twice.
- **One evaluation per bet.** A bet is evaluated at most once; repeat calls replay the stored
  outcome and report `alreadyEvaluated`. Otherwise a caller could retry a losing bet until it won.
- **No publishing from inside a transaction.** The bet is committed by `BetRecorder` before it is
  published. If publishing then fails, the bet row is discarded so the client can retry with the
  same bet id instead of hitting a duplicate error.
- **Money.** All amounts are `BigDecimal`, stored as `DECIMAL(19,2)` and rounded half up to 2
  decimals; percentages are held at 4 decimals. No floating point anywhere in the money path.
- **Randomness behind an interface.** `ChanceEvaluator` isolates the roll, so tests substitute a
  deterministic implementation and assert the exact reward and the resulting pool.

### The `bet` table

The requirements prescribe the contribution and reward tables and ask for bets to be stored, but
leave the bet's own shape open. It holds a status (`PUBLISHED`, `CONTRIBUTED`,
`NO_MATCHING_JACKPOT`) and an evaluation timestamp, which is what lets the API reject a duplicate
bet id up front, makes the outcome of asynchronous processing observable through
`GET /api/v1/bets/{betId}`, and keeps reward evaluation one-shot.

---

## Tests

`./gradlew test` runs 35 tests:

- **Calculator unit tests** pin the arithmetic of both strategies, including the decay floor, the
  chance ceiling, the guaranteed-win pool limit, and rounding.
- **`JackpotFlowIntegrationTest`** drives the real HTTP endpoints against the real persistence
  layer and covers the flow end to end: a bet contributing the right amount, a variable jackpot
  contributing less as it grows, a win paying the whole pool and resetting it, a loss leaving the
  pool alone, a redelivered bet contributing only once, duplicate and invalid bets, and bets that
  match no jackpot.

---

## Assumptions

- A jackpot pays out its **entire** current pool, then returns to its initial value.
- Only a bet that actually contributed can win; the bet and reward are for the same jackpot.
- A bet that names an unknown jackpot is still accepted and recorded, since a publisher cannot be
  expected to know the jackpot catalogue; it simply contributes nothing.
- `ddl-auto=create-drop` on H2 is deliberate for an in-memory database: state is rebuilt from
  configuration on every start. A durable database would use versioned migrations instead.
