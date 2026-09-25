# Broker-Neutral Copy-Trading Platform Design

**Status:** Approved conversational design, revised after written review

**Date:** 2026-09-25

**Initial operating scale:** 100 followers, 10 concurrent strategies
**Initial environment:** Paper trading through a deterministic simulated broker

## 1. Purpose

Build a production-shaped copy-trading platform that converts administrator-managed strategy targets into individually sized, risk-controlled follower positions. The platform must recover safely from duplicate events, partial fills, rejected orders, process crashes, stale market data, manual follower interference, and ambiguous broker responses.

The platform is declarative: strategies publish desired positions, not imperative buy/sell instructions. PostgreSQL records the authoritative desired and actual state, and execution continuously converges eligible follower accounts toward approved targets.

The first release remains entirely broker-neutral and never places real-money orders. Its simulator and adapter contract must nevertheless model the behavior and failure modes of real Indian broker APIs closely enough that a future broker adapter does not require redesigning the trading core.

## 2. Feasibility and external constraints

The system is technically feasible. Target-position replication, durable event processing, per-account execution serialization, and independent reconciliation are established patterns for safely converging distributed trading state.

Commercial deployment is a separate regulatory and broker-approval project. The current Indian retail-algo framework has applied to all stockbrokers since 1 April 2026. NSE implementation standards include mapped static IPs, daily API-session logout, broker RMS controls, and exchange registration rules based on order rates. SEBI has also cautioned investors about unregulated algorithmic-trading platforms. A future multi-customer live service therefore requires broker partnership and qualified Indian legal/compliance review before activation; this specification is not a legal opinion.

Primary references:

- [SEBI: Safer participation of retail investors in Algorithmic trading](https://www.sebi.gov.in/legal/circulars/feb-2025/safer-participation-of-retail-investors-in-algorithmic-trading_91614.html)
- [SEBI: implementation glide path and 1 April 2026 applicability](https://www.sebi.gov.in/sebi_data/attachdocs/sep-2025/1759232056254.pdf)
- [NSE: implementation standards for retail algorithmic trading](https://nsearchives.nseindia.com/content/circulars/INVG67858.pdf)
- [SEBI: caution concerning unregulated algorithmic-trading platforms](https://www.sebi.gov.in/sebi_data/attachdocs/jun-2022/1654859706888.pdf)
- [Kite order lifecycle and asynchronous outcomes](https://kite.trade/docs/connect/v3/orders/)
- [Kite authentication and daily token expiry](https://www.kite.trade/docs/connect/v3/user/)
- [Kite API and order-rate limits](https://kite.trade/docs/connect/v3/exceptions/)
- [Upstox OAuth authentication](https://upstox.com/developer/api-documentation/authentication/)
- [Upstox API rate limits](https://upstox.com/developer/api-documentation/rate-limiting/)
- [Digital Personal Data Protection Rules, 2025](https://www.meity.gov.in/static/uploads/2025/11/53450e6e5dc0bfa85ebd78686cadad39.pdf)
- [CERT-In directions, including ICT log retention](https://cert-in.org.in/PDF/CERT-In_Directions_70B_28.04.2022.pdf)

## 3. Goals and success criteria

The MVP must:

1. Accept manual administrator targets from the dashboard and machine targets from an authenticated strategy API.
2. Support cash equities, intraday long and short equities, futures, and independent single-leg options.
3. Represent multi-leg group and leg metadata in the schema without executing multi-leg strategies.
4. Allocate follower targets proportionally by allocated capital, then constrain them through deterministic risk policies.
5. Maintain virtual strategy-subscription ledgers while netting execution at broker-account and instrument level.
6. Submit eligible follower orders within two seconds at p95 from the committed master target under the defined 100-follower, 10-strategy load profile.
7. Remain correct under at-least-once event delivery, concurrency, restarts, delayed updates, and ambiguous order outcomes.
8. Detect manual follower interference and pause the affected account and instrument until explicitly acknowledged.
9. Provide append-only, correlated evidence for every target, allocation, risk decision, safety action, order, fill, reconciliation result, and override.
10. Prevent tenant-to-tenant data access and prevent secrets from entering logs or traces.

The latency SLO measures accepted target commit through initial simulated-broker submission. Exchange or simulator fill latency is excluded. Followers that are paused, already at target, risk-rejected, or rounded to zero are separately classified and never counted as successful latency samples.

## 4. Non-goals

The MVP does not include:

- A real broker adapter or live-money activation
- User-published strategies
- Multi-leg option execution or atomic multi-leg guarantees
- Unprotected market-order conversion
- Automatic liquidation as part of an ordinary pause or quarantine
- Automatic resumption after a safety or drift quarantine
- AI-generated trades or AI overrides of deterministic safety limits
- High-frequency or sub-second execution guarantees
- Custody of follower funds

## 5. Non-negotiable invariants

1. No order is dispatched without a current risk approval, margin reservation, and execution fencing token.
2. No exposure-increasing order is dispatched with stale required market or account data.
3. An ambiguous broker submission is never blindly retried.
4. A stale worker cannot dispatch after an execution epoch changes.
5. A target version cannot move backwards or overwrite a newer version.
6. Losing Redis cannot lose or corrupt trading state.
7. Platform pause, cancel, and quarantine actions do not implicitly flatten positions.
8. A reversal cannot use its closing risk exemption to open exposure on the opposite side.
9. Fractional allocation residuals cannot be transferred between followers, strategies, or instruments.
10. No authenticated principal can read or mutate resources outside the tenant and follower boundaries encoded in, and revalidated for, the session.

## 6. Architecture

The MVP is a modular monolith: one versioned codebase with isolated domain modules and independently scalable runtime roles.

```text
React dashboard / strategy clients
                 |
          API and ingress
                 |
      target application service
                 |
        PostgreSQL transaction
       state + audit + outbox
                 |
       durable worker consumers
       /       |       |       \
allocation   risk   execution   reconciliation
                         |
                simulated broker
                         |
                stream + snapshots
```

### 6.1 Runtime roles

- **API/control plane:** identity, authorization, strategy administration, subscriptions, target intake, risk configuration, acknowledgements, safety controls, and reporting.
- **Trading workers:** allocation, virtual targets, account aggregation, deterministic risk evaluation, and execution-command creation.
- **Execution workers:** per-account/instrument orchestration, price-protected orders, cancellation, replacement, and broker-update processing.
- **Reconciliation workers:** streamed-update ingestion, broker snapshots, drift detection, manual-trade detection, and quarantine.
- **Startup coordinator and scheduler:** deployment-generation fencing, boot reconciliation, periodic snapshots, session transitions, and retention jobs.
- **Simulated broker:** deterministic order, fill, position, funds, margin, rate-limit, and failure behavior behind the broker contract.

### 6.2 Storage responsibilities

PostgreSQL is the source of truth for business state, event outbox/inbox records, leases, fencing epochs, policy versions, and audit records. Redis is limited to replaceable cache entries, latest-version pointers, leaky-bucket counters, short-lived coordination hints, and read acceleration. Correctness must survive Redis flush, unavailability, or failover.

Audit batches are periodically exported to retention-locked object storage. Metrics, logs, and traces are operational telemetry and do not replace the business ledger.

## 7. Core domain model

### 7.1 Canonical instruments

Broker symbols never serve as primary identities.

- **Equity:** exchange, security identifier/ISIN, series, currency, tick size, tradable unit.
- **Future:** underlying identifier, exchange, expiry, currency, tick size, lot size, contract multiplier, settlement type.
- **Option:** underlying identifier, exchange, expiry, strike, call/put, currency, tick size, lot size, contract multiplier, settlement type.

Instrument master versions are effective-dated. An intent retains the instrument-master version used for its validation. Expired or disabled contracts cannot accept exposure-increasing targets.

### 7.2 Primary entities

- `tenants`, `users`, `roles`, `memberships`
- `followers`, `broker_accounts`
- `strategies`, `strategy_credentials`, `strategy_model_capital`
- `copy_subscriptions`, `allocated_capital`
- `instrument_versions`, `strategy_groups`, `strategy_legs`
- `target_intents`, `target_versions`
- `subscription_virtual_positions`, `account_targets`
- `risk_policies`, `risk_decisions`, `margin_reservations`
- `execution_epochs`, `execution_leases`, `execution_commands`
- `orders`, `order_attempts`, `fills`, `platform_receipts`
- `broker_events`, `broker_snapshots`, `positions`, `funds_snapshots`
- `reconciliation_runs`, `drift_cases`, `quarantines`, `acknowledgements`
- `outbox_events`, `inbox_receipts`, `audit_events`

All trading records use fixed-precision decimal values and signed integer quantities. Floating-point arithmetic is prohibited for money, price, quantity, margin, and P&L.

## 8. Target ingestion and version control

Both the administrator dashboard and external strategies invoke the same target application service. No ingress path calls the execution layer directly.

The primary command is:

```http
POST /strategies/{strategy_id}/targets
```

It contains:

- Canonical instrument key
- Signed target quantity
- Strategy model capital
- Reference price and timestamp
- Strictly increasing target version within the strategy/instrument stream
- Client idempotency key
- Optional expiration timestamp
- Optional `reduce_only` flag
- Optional future-facing strategy-group and leg identifiers

### 8.1 Stateless ingress controls

Each strategy has a distributed leaky bucket with a default drain rate of 20 target requests per second and burst capacity of 40. The values are administrator-configurable without changing application code.

The ingress layer reads a cache pointer for the latest committed version. Requests at or below that pointer are rejected before database access. The pointer is written only after the authoritative transaction commits. Cache absence falls back to the database; cache state never authorizes a transition. Database uniqueness and monotonic-version constraints remain the final replay defense.

External requests use scoped credentials, signed payloads, a nonce, and a maximum default clock skew of 30 seconds. Reused nonces and idempotency keys return the original result when safe or a conflict when the payload differs.

## 9. Allocation, rounding, and netting

The raw follower target is:

```text
raw target =
    strategy target quantity
    * follower allocated capital
    / strategy model capital
```

The result is rounded toward zero to the instrument's tradable unit or derivative lot size. The exact fractional residual is stored with the allocation decision for explanation and testing. It is not accumulated into a separate executable entitlement, moved between users, or reassigned across instruments. A future target naturally absorbs the residual only when its newly calculated quantity crosses the next tradable unit.

The allocation record stores the raw target, rounded target, risk-approved target, residual, input prices, input timestamps, instrument-master version, and policy version.

Each subscription maintains a virtual target and virtual position. These positions aggregate into one account/instrument target. Opposing strategy targets net before external execution. Internal offsets are recorded at the canonical mark used by the allocation run so attribution remains explainable; they are not represented as broker fills.

## 10. Position reversals

Quantities are signed. The core delta is:

```text
required delta =
    approved aggregate target
    - confirmed broker position
    - remaining signed quantity of confirmed live platform orders
```

A move from `-5` to `+10` is a required delta of `+15`. It executes as two phases whenever product, venue, or risk rules require it:

1. Buy 5 with `reduce_only=true` to reach zero.
2. Obtain a fresh market-data, margin, and risk approval.
3. Buy 10 as new long exposure.

The second phase cannot inherit the first phase's risk-reducing permission. If new exposure is paused or denied, the workflow stops at zero.

## 11. Risk and margin

Risk policies exist at global, tenant, account, strategy, subscription, asset-class, underlying, and instrument scopes. The most restrictive applicable value wins. Followers may tighten their policies but cannot loosen administrator hard limits.

Initial default hard limits are:

- Maximum risk budget per target: 1% of applicable NAV
- Maximum daily loss: 3% of start-of-day account equity
- Maximum drawdown: 10% from persisted high-water equity
- Maximum gross capital or notional concentration per position: 20% of applicable allocation
- Maximum simultaneously open instrument positions per subscription: 5
- Naked short options: disabled

Administrators may lower these defaults. Raising them requires privileged policy change, step-up authentication, an audit reason, and a new policy version.

Every exposure increase checks:

- Platform, account, strategy, subscription, and instrument state
- Trading session and instrument eligibility
- Quote, volatility, position, funds, and margin freshness
- Quantity, lots, notional, concentration, and position-count limits
- Account, strategy, sector, underlying, and correlated exposure
- Available funds and projected post-order margin
- Margin and exposure already reserved for pending commands
- Daily realized plus marked-to-market loss
- Account and subscription drawdown
- Exchange price bands, configured liquidity minimums, and order price protection

Market data is stale by default after five seconds for tradable quotes and fifteen seconds for IV inputs during an active session. These thresholds are versioned by asset class. Stale data blocks new exposure. A risk-reducing order may proceed only with a conservative price bound and fresh enough position state.

### 11.1 Dynamic margin buffer

Margin is reserved atomically before an execution command is released. The default required reservation is the simulated or broker-provided estimate plus a 15% baseline buffer.

For options and futures, the IV ratio is current relevant IV divided by the trailing 20-session median IV for the same underlying and comparable tenor. The default IV surcharge is:

- Ratio below 1.25: 0 percentage points
- Ratio from 1.25 to below 1.50: 5 percentage points
- Ratio from 1.50 to below 2.00: 10 percentage points
- Ratio at or above 2.00: 20 percentage points

For physically settled derivatives, the default expiry surcharge is:

- More than 48 hours to expiry: 0 percentage points
- From 48 hours to more than 24 hours: 10 percentage points
- 24 hours or less: 20 percentage points

Surcharges are additive to the 15% baseline and capped at a total 60% buffer. Exchange calendars and instrument settlement metadata determine time to expiry. Missing or stale required IV blocks exposure increases rather than silently using the baseline. Every multiplier and threshold is a versioned policy input recorded with the decision.

## 12. Safety states and epoch fencing

Safety commands are distinct:

- `PAUSE_NEW_EXPOSURE`
- `CANCEL_PLATFORM_ORDERS`
- `QUARANTINE`
- `RESUME_WITH_ACKNOWLEDGEMENT`
- `FLATTEN_WITH_EXPLICIT_CONFIRMATION`

The ordinary emergency action pauses new exposure and cancels platform-owned open orders. It does not liquidate existing positions. Flattening is a separate permission-controlled command with step-up authentication, explicit confirmation, and complete audit context.

Invoking `PAUSE_NEW_EXPOSURE`, `CANCEL_PLATFORM_ORDERS`, or `QUARANTINE` atomically increments the database execution epoch for the targeted scope in the same transaction as the state change and outbox event. The scope may be global, tenant, account, strategy, subscription, broker, or instrument. Every queued command and active lease must revalidate all applicable epoch values immediately before an external dispatch.

An API request already transmitted cannot be recalled by epoch advancement. Such attempts are marked for cancellation and immediate reconciliation.

### 12.1 Execution leases

Only one workflow may control a broker-account/instrument pair at once. The lease includes the account ID, instrument ID, deployment generation, account execution epoch, instrument/scope epochs, owner, expiry, and a monotonically increasing fencing token. The database rejects state changes from stale fencing tokens. Redis is not the lease authority.

Account-wide safety operations take precedence by advancing the account epoch. They do not wait for narrower leases to expire.

Lease acquisition locks the account/instrument ownership row, increments its fencing token, and records the current deployment generation and every applicable execution epoch. The default lease lasts five seconds and is renewed every second while its worker remains healthy. A failed renewal immediately stops further dispatch from that worker. After expiry, another worker may claim ownership only by atomically incrementing the fencing token; the prior owner can no longer persist execution state or obtain a dispatch permit.

Immediately before each broker call, the worker uses a database transaction to verify the lease expiry, fencing token, deployment generation, safety state, and applicable epochs, then records a uniquely identified dispatch attempt. The network request follows that committed dispatch intent. A safety transaction that linearizes after the permit cannot retract an already-started network call, so the resulting attempt is cancelled and reconciled as described above.

## 13. Execution policy and state machine

Default follower orders are marketable limit orders. Initial price-protection defaults are 25 basis points for equities, 20 basis points for futures, and 100 basis points for options, further constrained by tick size and exchange price bands. Orders have a default working TTL of two seconds and at most two cancel-and-reprice attempts. Exhaustion enters `RECONCILIATION_REQUIRED` and quarantines the account/instrument; an order is never automatically converted to an unrestricted market order.

Execution states include:

```text
CREATED -> RISK_PENDING -> APPROVED -> RESERVED -> DISPATCH_PENDING
-> SUBMITTED -> BROKER_ACCEPTED -> PARTIALLY_FILLED -> FILLED
```

Alternate states include:

```text
SUPERSEDED, REJECTED, CANCEL_PENDING, CANCELLED, EXPIRED,
FAILED, OUTCOME_UNKNOWN, RECONCILIATION_REQUIRED, QUARANTINED
```

A newer target supersedes older unsubmitted commands. Confirmed live orders are cancelled or modified only through a new fenced execution decision.

### 13.1 Deterministic order identity

The platform creates a canonical deterministic execution identity from tenant, broker account, instrument, aggregate target version, phase, and attempt. Adapters encode this identity into a broker `ClientOrderId`, tag, or supported correlation field. The simulator preserves the full identifier.

If a future broker cannot preserve an adequate client identifier, the adapter must produce a signed local submission receipt and correlate responses through adapter-specific immutable fields and order fingerprints. Lack of correlation is an unsupported adapter state, not permission to retry.

## 14. Events and transactional delivery

Business mutations and their outgoing events commit in one PostgreSQL transaction through an outbox. Consumers apply an inbox receipt and business mutation atomically. Delivery is at least once; effectively-once business effects come from idempotency constraints, monotonic versions, fencing, and reconciliation.

Events contain an event ID, type, schema version, tenant, aggregate identity, aggregate version, correlation and causation IDs, occurred-at time, committed-at time, and typed payload. Consumers reject unknown incompatible schema versions.

Outbox publishers claim bounded batches with `FOR UPDATE SKIP LOCKED` and a 30-second database claim lease. Publishing and marking a row as published cannot be atomic with an external transport: a crash after publish but before acknowledgement deliberately causes redelivery. Consumers enforce a unique `(consumer_name, event_id)` inbox key and commit that receipt with their state mutation, making redelivery harmless.

A publisher crash mid-batch leaves unpublished rows pending and claimed rows reclaimable after lease expiry. Retries begin at 250 milliseconds, double to a maximum 30 seconds, and enter an operator-visible dead-letter state after 20 failed attempts without deleting the event. Restart tests must crash publishers and consumers before and after every transaction boundary.

No component claims end-to-end exactly-once broker execution.

## 15. Broker adapter and simulator

The broker interface exposes:

- Capabilities and instrument mapping
- Session/authentication state
- Order submission, modification, and cancellation
- Order, trade, position, funds, and margin snapshots
- Streaming order, fill, position, and connection events
- Pre-order margin estimation
- Rate-limit and session metadata
- Correlation and sequence-domain rules

The deterministic simulator implements:

- Seeded fills and a controllable clock
- Marketable-limit price behavior
- Full, partial, delayed, and absent fills
- Rejections, price-band failures, and insufficient margin
- Rate limiting and connection failures
- Duplicate, delayed, missing, and out-of-order events
- Ambiguous submissions and delayed snapshot visibility
- Manual untagged orders and positions
- Funds, margin, IV, and expiry-window shocks
- Process restart and session-boundary scenarios

## 16. Reconciliation and manual-trade detection

Reconciliation compares:

1. Strategy-subscription virtual targets
2. Approved aggregate account targets
3. Confirmed live platform orders and reserved exposure
4. Broker orders, fills, positions, funds, and margin snapshots

The projected broker position is:

```text
projected position =
    confirmed broker position
    + remaining signed quantity of confirmed live platform orders
```

For derivatives, any non-zero difference between the approved aggregate target and projected position immediately quarantines the account/instrument. When no live order remains, this becomes an exact target-versus-position comparison.

For equities, an unexplained difference of at least one tradable unit that persists for five seconds quarantines the account/instrument. Detection of any untagged broker order or fill triggers quarantine immediately, regardless of position difference.

A manual trade signature exists when either:

- An order or fill lacks a recognized platform correlation identity or valid platform-signed receipt, or
- Broker position changes without correlated platform fills that explain the change.

Quarantine advances the applicable epoch, blocks automatic correction, requests cancellation of identifiable platform orders, and requires an explicit audited acknowledgement before resume.

### 16.1 Sequence and snapshot handling

Each adapter declares its broker sequence domains and whether a snapshot exposes a comparable high-water marker. A streaming event is discarded when its sequence is less than or equal to the authoritative snapshot watermark only when the broker guarantees comparability in that domain.

Where only timestamps exist, the engine does not assume global monotonicity. It deduplicates using stable event identity, broker order identity, fill identity, payload fingerprint, and monotonic state-transition validation. Concurrent events around a snapshot are retained unless conclusively covered by its watermark.

### 16.2 Ambiguous outcomes

`OUTCOME_UNKNOWN` begins when a submission may have reached the broker without an authoritative response. The engine queries order, trade, and position snapshots and sends no replacement order. If unresolved after exactly 30 seconds, it advances the applicable epoch, quarantines the account/instrument, cancels any identifiable platform orders, and alerts an operator.

## 17. Startup, shutdown, and recovery

A deployment starts execution-closed. One database-elected startup coordinator establishes a new deployment generation and atomically advances the execution epoch for every managed account. Worker replicas join that generation; they do not independently advance all epochs.

Before opening new exposure, the system:

1. Rebuilds projections from committed PostgreSQL state.
2. Loads fresh broker orders, trades, positions, funds, and margin snapshots.
3. Resolves or quarantines pending and unknown executions.
4. Runs reconciliation and current risk checks.
5. Opens only accounts that pass every gate.

Graceful shutdown closes dispatch, allows bounded in-flight persistence, advances affected epochs when ownership is uncertain, and leaves unresolved external work for startup reconciliation. Crash recovery never blindly replays an external request.

## 18. Identity, authorization, and tenant isolation

Authentication uses an OIDC-compatible identity layer, short-lived access sessions, refresh controls, and MFA for privileged roles. Authorization combines RBAC with attribute/resource matching.

Signed session claims include subject, tenant boundaries, follower boundary where applicable, role set, authentication strength, token ID, issued-at time, and expiry. Every resource query and command requires matching tenant and follower attributes. The server also verifies current membership and revocation state so a previously valid token cannot preserve removed access indefinitely.

Database access helpers require an explicit tenant context. Missing context fails closed. Integration tests attempt cross-tenant reads and writes through every API class. PostgreSQL row-level security may provide defense in depth, but application authorization remains mandatory.

Privileged operations such as global resume, flattening, enabling naked option shorts, and loosening risk limits require step-up authentication, explicit reason text, and a separate command record. The command model supports later four-eyes approval.

## 19. Secrets, privacy, and log safety

Strategy credentials and future broker tokens use envelope encryption through a KMS-compatible interface. Secrets remain server-side and never enter browser storage, query strings, analytics payloads, or ordinary application tables in plaintext.

All logs pass through an automated redaction interceptor. Raw object dumping, arbitrary dictionary serialization, and unrestricted exception serialization are prohibited. Approved schemas whitelist printable fields. External request/response and traceback filters scrub authorization headers, bearer values, cookies, API keys, connection strings, tokens, and configured sensitive field names before standard output or trace export.

CI includes canary-secret tests that fail if representative secrets appear in captured logs or traces. Production logging fails closed to a minimal safe event when a payload lacks an approved schema.

As of this specification date, the notified Digital Personal Data Protection Rules, 2025 put Rules 1, 2, and 17–21 into force on publication; Rule 4 takes effect on 13 November 2026, and Rules 3, 5–16, 22, and 23 take effect on 13 May 2027. The release checklist must revalidate these dates and any amendments against the official Gazette.

Any paper pilot that onboards external people or processes real personal data is blocked until a privacy readiness review covers notice, consent or other lawful basis, purpose limitation, access, correction, erasure, retention, processor contracts, breach response, and user complaints. An internal demonstration using only synthetic identities may proceed without representing itself as an externally available pilot. Any live-money or public release additionally requires written legal confirmation of the platform's regulatory role.

As of this specification date, applicable CERT-In directions require relevant ICT logs to be retained securely for a rolling 180 days within India and specified cyber incidents to be reported within six hours of noticing them. The operations milestone must implement an India-resident log archive, time synchronization, named incident contacts, a six-hour reporting runbook, and an evidence-producing incident exercise. Release review must revalidate these obligations against the current official CERT-In directions.

## 20. Auditability

Audit events are append-only and include actor, tenant, action, resource, prior and resulting version references, policy version, reason, correlation ID, causation ID, request origin, authentication strength, timestamp, and outcome. Sensitive values are referenced or redacted rather than copied.

Application roles receive insert and select privileges on audit rows but no update or delete privileges; database triggers reject mutation by ordinary runtime roles. Each row receives a database-generated global sequence. PostgreSQL superuser access remains a trusted administrative boundary, so PostgreSQL alone is not described as immutable.

One database-elected audit exporter constructs deterministic batches from contiguous, previously unexported sequence ranges. Each signed manifest contains the first and last sequence, row count, payload Merkle root, previous manifest hash, creation time, and signing-key identifier. The exporter writes the encrypted batch and manifest to retention-locked object storage before advancing the database checkpoint. A crash retries the same deterministic batch key, so multiple workers or restarts cannot fork the chain.

Committed audit events have an export-lag SLO of 60 seconds at p95 and a hard operational limit of 120 seconds. Crossing 120 seconds pages an operator; crossing five minutes automatically pauses new exposure while leaving risk-reducing actions available. An hourly incremental verifier checks sequence continuity, signatures, manifest links, and object retention state; a daily full verifier rechecks all manifests created that day. Export and verification evidence is a mandatory release artifact.

## 21. Dashboard and operator workflows

- **Administrator:** strategy lifecycle, instruments, hard policies, margin rules, simulator scenarios, and follower entitlements.
- **Operator:** live workflows, backlog, drift, unknown outcomes, quarantines, safety controls, cancellation, acknowledgement, and reconciliation evidence.
- **Follower:** subscriptions, allocated capital, stricter personal policies, virtual and broker positions, P&L, pauses, drift explanations, and acknowledgement requests.
- **Auditor:** read-only target, decision, epoch, command, order, fill, policy, acknowledgement, and override histories.

The UI clearly distinguishes desired target, risk-approved target, projected broker position, confirmed broker position, and unexplained drift. Destructive or exposure-changing controls display their exact scope and effect before confirmation.

## 22. Observability

Structured logs, traces, metrics, and alerts use correlation IDs shared with business records. Primary measurements are:

- Target-commit to first-submission p50, p95, and p99
- Outbox, inbox, and worker backlog age
- Allocation exclusions and rounding-to-zero counts
- Target/projected/confirmed position drift
- Snapshot and reconciliation freshness
- `OUTCOME_UNKNOWN` age
- Margin utilization and applied buffer components
- Rejection, cancellation, repricing, and partial-fill rates
- Execution-epoch changes and stale-fence rejections
- Quarantines and acknowledgement age
- Stale market data, sequence gaps, duplicates, and discarded events
- Ingress throttling and replay rejection counts

Alerts are actionable and identify tenant-safe resource references, severity, current safety state, and runbook link. They exclude secrets and unnecessary personal data.

The initial operational objectives are:

- Eligible target commit to first broker submission: below two seconds at p95
- Active-account reconciliation freshness while an execution is open: below one second at p95 and never intentionally scheduled beyond two seconds
- Inactive-account snapshot freshness during a trading session: no more than 30 seconds
- Audit export lag: below 60 seconds at p95 and no more than 120 seconds without alerting
- Non-streaming dashboard read endpoints: below 500 milliseconds at p95 under the acceptance load
- Safety-state visibility in the operator dashboard: below one second at p95 after database commit

SLO calculations exclude planned maintenance but never exclude failures, retries, or overload. Breaches produce evidence and a remediation decision rather than silently resetting the measurement window.

## 23. Deployment and operations

The initial deployment uses containers in one region with separately scalable API, trading-worker, execution-worker, reconciliation-worker, scheduler/coordinator, dashboard, PostgreSQL, Redis, and observability processes. All application processes run the same versioned release.

Migrations are backward-compatible across a rolling deployment. New deployments remain execution-closed until generation fencing and boot reconciliation complete. Readiness does not imply trading readiness; the system exposes separate service-health and execution-readiness states.

For the single-region paper MVP, the database recovery objectives are RPO no greater than five minutes and RTO no greater than four hours. PostgreSQL uses encrypted daily full backups plus continuous write-ahead-log archiving. A clean-room restore test runs monthly and must demonstrate both objectives and a complete audit-sequence verification. Any live-money release requires a new threat and business-impact assessment and a multi-availability-zone design; the paper-MVP objectives are not inherited automatically.

Redis contains only replaceable latest-version pointers, leaky-bucket state, response caches, and coordination hints. Durable targets, policies, rate-limit configuration, epochs, leases, commands, and events remain in PostgreSQL. After total Redis loss, version pointers and caches rebuild from PostgreSQL. Machine strategy ingress fails closed until the distributed leaky bucket is available; safety commands and reconciliation remain available through PostgreSQL. Redis backups are therefore not required for trading correctness.

Configuration and risk-policy changes are versioned, validated, and auditable.

## 24. Verification strategy

### 24.1 Unit and property testing

- Signed target arithmetic and reversal decomposition
- Lot/tradable-unit rounding and residual isolation
- Proportional allocation and netting invariants
- State-machine transition legality
- Risk limit precedence and policy versioning
- IV and physical-settlement margin surcharges
- Daily loss and high-water drawdown
- Deterministic execution and receipt identities

### 24.2 Integration testing

- PostgreSQL outbox/inbox atomicity
- Publisher crash before publish, after publish, and before the published checkpoint
- Consumer crash before and after committing its inbox receipt
- Competing execution workers and stale fencing tokens
- Lease renewal failure, expiry, takeover, and stale-owner persistence rejection
- Atomic epoch changes for every safety scope
- Startup-generation fencing with multiple replicas
- Redis loss, stale cache pointers, and cache recovery
- Sequence watermarks and concurrent snapshot events
- RBAC plus tenant/follower attribute enforcement
- Approved logging schemas and canary-secret interception

### 24.3 Scenario testing

- Partial fills followed by cancel or rejection
- Duplicate and out-of-order updates
- Ambiguous submission resolved before and after 30 seconds
- Untagged manual trades and unexplained position changes
- Stale quotes and missing IV
- IV spikes and 48/24-hour settlement windows
- Margin deterioration after order acceptance
- Restart during every execution state
- Pause, cancel, quarantine, resume, and explicit flatten commands
- Opposing strategies and account-level netting

### 24.4 Load and endurance testing

The acceptance profile uses 100 followers and 10 concurrent strategies with burst traffic at the ingress limits. The platform must maintain target-to-submission p95 below two seconds for eligible deltas, preserve every invariant, exhibit no cross-tenant access, and return to a reconciled state after injected failures. Endurance tests cover full simulated trading sessions and session-boundary restart/reconciliation.

After passing the acceptance profile, a non-blocking capacity-characterization run uses 1,000 followers and 100 strategies. It does not impose the MVP SLO at that scale; it records saturation points, queue growth, database contention, and the partitioning or service-extraction work required before claiming support. No 10,000-follower capability is advertised without a separately approved scale plan and measured test evidence.

## 25. Delivery boundaries

The design decomposes into these testable capability milestones:

1. **Secure foundation:** core types, state machines, tenant-context enforcement, durable ledger, append-only audit ingestion, approved logging schemas, and secret interception.
2. **Instrument and simulation contract:** canonical equities, futures, and single-leg options plus the deterministic simulated broker and asset-specific failure scripting. This milestone exposes no path from a target to broker submission.
3. **Durable target processing:** authenticated ingress, outbox/inbox delivery, virtual ledgers, allocation, residuals, and account netting under tenant isolation.
4. **Risk and safety:** equity and derivative risk, IV and expiry margin behavior, drawdown, reservations, execution epochs, and safety controls. No execution command can exist without this gate.
5. **Execution:** orchestration, marketable-limit protection, reversal phases, leases, fencing, correlation receipts, and order state machines.
6. **Reconciliation and recovery:** streamed events, snapshots, manual-trade detection, unknown outcomes, crash recovery, and boot fencing.
7. **Product and privacy readiness:** administrator, operator, follower, and auditor APIs and dashboards; identity lifecycle; personal-data workflows; and external-pilot privacy gate.
8. **Audit, incident, and disaster operations:** locked audit export and verification, CERT-In-aligned retention and incident exercise, backup/restore evidence, and operator runbooks.
9. **System acceptance:** baseline load, endurance, fault injection, security testing, SLO evidence, and 1,000-follower capacity characterization.
10. **Paper-pilot release:** deployment rehearsal, execution-readiness review, release evidence bundle, and approval limited to the reviewed internal or external pilot scope.

### 25.1 Milestone exit gates

| Milestone | Required evidence before exit |
|---|---|
| 1 | Clean migrations; tenant-context tests deny missing and mismatched scopes; audit mutation is denied; canary secrets never reach captured logs or traces. |
| 2 | One broker contract suite passes for every supported asset class and every scripted failure is reproducible from its seed; no executable target endpoint exists. |
| 3 | Monotonic target and replay tests pass; publisher and consumer crash-boundary tests show no lost state; allocation/netting property tests preserve signed quantities and residual isolation. |
| 4 | No execution command can be created without current risk approval and atomic reservation; IV/expiry, drawdown, stale-data, epoch, and concurrent-reservation scenarios pass. |
| 5 | State-machine, partial-fill, cancellation, repricing, reversal, lease-expiry, and stale-fence tests pass; a small concurrent fan-out meets the two-second SLO before scale testing. |
| 6 | Restart from every execution state converges or quarantines without duplicate exposure; manual trades and 30-second unknown outcomes enforce the specified quarantine behavior. |
| 7 | Cross-tenant API attempts fail; step-up controls work; accessibility and role workflows pass; privacy review approves the exact pilot data scope. |
| 8 | Audit export stays within its lag limits and verifies end to end; a clean-room restore meets RPO/RTO; incident exercise produces the required contacts, timeline, and evidence. |
| 9 | All invariant, security, fault, baseline-load, endurance, and SLO suites pass; the capacity report records the 1,000/100 saturation behavior without making an unsupported SLA claim. |
| 10 | Deployment and rollback rehearsals pass; boot reconciliation reaches execution-ready; release evidence is signed off by the platform safety owner and, for any external pilot, the privacy/compliance owner. |

The detailed implementation plan will define exact files, interfaces, test commands, commit boundaries, and reviewer gates. The specification defines behavioral exit evidence without substituting an arbitrary global line-coverage percentage for safety-critical scenario coverage.

## 26. Future evolution

The boundaries intentionally permit later additions without committing the MVP to them:

- Real broker adapters with capability declarations
- Broker and exchange approval workflows
- Multi-leg option execution and portfolio Greeks
- User-published strategies and disclosure workflows
- Service extraction where measured load justifies it
- Commercial compliance controls and four-eyes operations
- AI-based analysis that can recommend risk reduction but cannot override hard limits

None of these future capabilities may weaken the invariants in Section 5.

Every future capability begins with an architecture decision record containing an invariant-impact matrix, threat-model delta, data/privacy impact, migration and rollback plan, and tests that prove unchanged invariants. The platform safety owner must approve the record before implementation. Any externally facing, personal-data, broker, exchange, or live-money change also requires privacy/compliance approval. CI keeps the invariant suite mandatory for every subsequent release; governance is a release gate rather than a reminder in documentation.
