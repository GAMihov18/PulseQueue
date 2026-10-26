# PulseQueue

A durable background-job queue with visible retries and recovery.

> Planning repository — documentation and Git initialization only. No application, package, solution, database, tests, CI workflow, or runnable demo has been implemented yet. Everything below describes the proposed implementation, unless explicitly marked as verified.

## Goals

Make job delivery semantics understandable through a small persistent queue, worker leases, retry histories, and operational views. Demonstrate failure recovery and bounded work rather than claiming distributed-system guarantees that the storage model cannot provide.

## Target users

Backend developers adding asynchronous work, small teams running scheduled or deferred tasks, and learners studying worker coordination.

## MVP scope

- Persist typed job payloads with schema version, enqueue time, and optional deduplication key.
- Atomic claims with leases, heartbeats, cancellation, and attempt history.
- Exponential backoff with jitter, retry limits, and a dead-letter state.
- Separate producer API and worker host with registered handlers.
- Dashboard for queue depth, oldest-job age, failures, and controlled requeue.

## Stretch goals

- Recurring schedules with explicit timezone and missed-run policy.
- Priority lanes and per-tenant concurrency limits.
- PostgreSQL storage adapter and multi-host contention benchmarks.

## Proposed tech stack

C# / .NET 10, ASP.NET Core, .NET Worker Service, SQLite for a single-machine MVP, Blazor dashboard, xUnit. Adopt PostgreSQL only after a shared storage contract is tested.

Select compatible stable dependency versions during milestone 1, record them in project files, and pin the SDK/toolchain after the compatibility spike. No packages or development workloads are installed by this documentation task.

## Architecture

Queue core defines transitions and delivery contracts. Storage performs transactional claims and conditional lease updates. The worker pulls bounded batches, renews leases, and dispatches handlers through a registry. API and dashboard operate through application services. Inject time and randomness for deterministic lease/backoff tests; do not hold database transactions across handler execution.

## Proposed folder structure

Only README.md and .gitignore exist initially, alongside Git metadata. The following tree is a plan; create directories when their implementation begins.

```text
PulseQueue/
  README.md
  .gitignore
  PulseQueue.slnx
  src/PulseQueue.Core/
  src/PulseQueue.Storage.Sqlite/
  src/PulseQueue.Worker/
  src/PulseQueue.Api/
  src/PulseQueue.Dashboard/
  tests/PulseQueue.UnitTests/
  tests/PulseQueue.IntegrationTests/
  samples/PulseQueue.DemoHandlers/
  docs/semantics/
```

## Data models and invariants

Job(Id, Type, PayloadVersion, Payload, State, AvailableAtUtc, DedupKey); Lease(JobId, OwnerId, Token, ExpiresAtUtc); Attempt(Id, JobId, Number, StartedAtUtc, CompletedAtUtc, Error); DeadLetter(JobId, Reason, FailedAtUtc). Transition Enqueued -> Leased -> Succeeded, or retry/delay/dead-letter. A stale worker cannot complete a job without its current lease token.

## Proposed API and contracts

POST /api/jobs enqueues with an optional deduplication key; GET /api/jobs/{id} includes attempts; POST /api/jobs/{id}/cancel records cancellation intent; POST /api/deadletters/{id}/requeue requires an explicit reason. IJobHandler<T>.HandleAsync(payload, context, cancellationToken) must tolerate repeated delivery.

Version persisted/exported formats from their first implementation. Validate at boundaries and return structured errors rather than leaking internal exceptions. The contracts above remain proposals until backed by code and contract tests.

## Implementation milestones

1. Specify at-least-once delivery, deduplication scope, cancellation, and lease ownership.
2. Implement transactional enqueue/claim/complete and recovery from worker termination.
3. Add handlers, retries, dead letters, and producer API.
4. Build operations view; demonstrate worker kill/restart and duplicate-side-effect mitigation.

Each milestone should end with a small demonstrable result, a written acceptance check, and tests for its important failure modes. Work through a vertical slice before expanding the stretch scope.

## Testing and CI plan

Run concurrent claim tests against real temporary SQLite databases, lease expiry and fencing tests with fake time, retry/jitter bounds, poison payloads, and process-kill recovery. Verify handler idempotency in examples. CI runs unit/integration tests on Windows/Linux and a multi-worker smoke test with deterministic synthetic jobs.

CI is planned, not configured. Once a solution exists, add .github/workflows/ci.yml with read-only default permissions, explicit toolchain setup, dependency restore, build, and the checks above. Keep ordinary CI offline from real services and hardware. Store test reports as artifacts; do not publish packages, create releases, or deploy from the first workflow. Record hardware/manual results separately from automated passes.

## Setup and development

### Current checkout

Open this README in a Markdown viewer. Git is initialized locally; no build or run step exists yet. Check the working tree with `git status --short --branch`. Documentation files start untracked; no initial commit or remote is created.

### Prerequisites and first implementation

After scaffolding, create the database migration and a harmless demo handler that writes a local result keyed by job ID. Run the API and worker as separate processes pointing at the same absolute database path. Set concurrency to one initially and increase only after contention tests.

Install only the dependencies needed for the selected milestone. Add a checked-in sample configuration with harmless defaults; keep local databases, recordings, generated datasets, credentials, and device identifiers outside version control. Pick a license before public distribution; no license is implied by this plan.

### Future commands

These commands are examples for after the referenced projects/files have been created; they will not build this documentation-only repository.

```powershell
dotnet restore PulseQueue.slnx
dotnet test PulseQueue.slnx
dotnet run --project src/PulseQueue.Worker
```

## Supporting libraries and boundaries

ConfigLens for worker settings and DataSynth for load fixtures. FlowForge may eventually use PulseQueue for scheduling work, but neither MVP requires the other.

Consume stable library versions through local package artifacts initially. Avoid absolute ProjectReference paths into sibling repositories and avoid Git submodules for the MVP. Keep a consumer's first vertical slice runnable using a small local interface or fixture while a supporting package is under development.

## Realistic limitations

Delivery is at least once, not exactly once. A worker can execute a side effect and die before acknowledging it. SQLite limits write concurrency and this MVP is not an internet-scale broker. Cancellation is cooperative; it cannot undo an external side effect.

Scope and performance targets are hypotheses until measured. Document reproducible dataset sizes, hardware, dependency versions, and failure cases alongside any benchmark claim. Do not present planned capabilities as shipped features.

## Documentation provenance

Prepared 2026-10-23. The previous chat supplied the project names and the AutoDiag platform change, but its detailed assistant responses were unavailable content references. The user confirmed the seven proposed application themes in this session; individual designs here are newly proposed plans, not a recovered transcript.

