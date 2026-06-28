# Coordination-kernel architecture hardening plan

## Decision

Adopt a coordination-kernel architecture rather than a heavyweight event-sourced/domain-driven rewrite.

Orchlink v1 remains local-first and single-operator. It should not paint itself into a corner for future multi-operator use. Cancellation remains a documented best-effort contract: Orchlink revokes leases, sends steering, and aborts if Pi exposes an abort hook, but does not guarantee already-running tool calls stop immediately.

## Target architecture

Three planes:

1. CLI plane: thin HTTP client, no live-state business logic for task/talk/session coordination.
2. Broker/control plane: mailbox, jobs, leases, sessions, status transitions, audit journal, storage backends.
3. Pi/agent plane: poll loop, heartbeat, cancel-check, reply delivery, recoverable-error handling, compaction hooks.

Hard contract:

- The versioned message envelope is the stable boundary between broker, CLI, and Pi extension.
- Live task/talk/session state goes through broker APIs.
- Goal Mode can keep `.orch/goals/*` as v1 live state, but goal transitions must be journaled from v1 so v1.1 can project goal state without a painful backfill.

## V1 milestones

### M1 — Audit journal

Add an append-only audit journal written on every broker state transition and every Goal Mode transition.

Deliverables:

- `src/orchlink/broker/journal.py`
- journal integration in broker mutations
- `GET /v1/journal?project_id=&since=&limit=`
- goal transition journaling from `goal/cli.py` and `goal/runner.py` while keeping current goal file storage
- tests for append-only behavior, project scoping, cursor/since behavior, and transition coverage

Journal record shape:

```json
{
  "time": "...",
  "project_id": "orchlink",
  "actor": "orchlink.work",
  "action": "job.replied",
  "target_type": "job",
  "target_id": "TASK-001",
  "before": "RUNNING",
  "after": "DONE",
  "meta": {}
}
```

Action vocabulary:

- `job.created`, `job.dispatched`, `job.heartbeat`, `job.reclaimed`, `job.replied`, `job.cancelled`, `job.terminal`
- `lease.acquired`, `lease.renewed`, `lease.expired`
- `session.registered`, `session.released`
- `goal.started`, `goal.gated`, `goal.worked`, `goal.blocked`, `goal.done`, `goal.cancelled`, `goal.signedoff`

### M2 — Envelope contract hardening

Create one typed, versioned envelope contract.

Deliverables:

- `src/orchlink/core/envelope.py`
- `docs/envelope.md`
- `tests/test_contract.py`
- `x-orchlink-envelope: 1` on `/v1` responses
- no bare dict envelope construction in `bridge/` or `cli/` where the core contract should be used

### M3 — Job lease + epoch reliability

Make stale replies and stale heartbeats rejectable.

Deliverables:

- job lease model: `{holder, expires_at, epoch, heartbeat_ms}`
- `RECLAIMABLE` lifecycle state
- `POST /v1/jobs/{id}/heartbeat`
- `POST /v1/jobs/{id}/reclaim`
- reply endpoint rejects wrong holder/epoch with 409
- terminal transitions clear leases
- extension heartbeat renews leases and stops on 409
- tests for acquire, renew, expire, reclaim, stale reply rejection, terminal lease clearing, and epoch monotonicity

Lease invariants:

- non-terminal active job has a lease
- holder is explicit agent ID
- heartbeat only renews matching holder and epoch
- expired lease can be reclaimed
- epoch only increases
- terminal state clears lease
- only holder at current epoch may reply
- reclaim is idempotent within one logical tick

### M4 — Pi extension pure-function extraction

Make the generated TypeScript extension more testable.

Deliverables:

- extract pure logic into `src/orchlink/connector/pi_extension_pure.ts` or equivalent generated artifact
- keep `pi_extension.py` focused on Pi event wiring
- tests for reconciliation detection, recoverable-error detection, compaction summary, reply envelope construction, and lease math

### M5 — CLI filesystem-coupling guard

Prevent task/talk live state from depending on direct `.orch` file reads.

Deliverables:

- `tests/test_cli_no_live_fs_coupling.py` (static guard: task/talk CLI must not reference `.orch/goals/*`, must use `.orch/run/*` only for the store-path default, must not import the Goal Mode live-state layer, and must route state through `BrokerClient`)
- `docs/v1.1-roadmap.md` documenting that Goal Mode file coupling is a v1.1 target (M6), not a v1 blocker

## V1.1 milestone

### M6 — Goal Mode projection

Make `.orch/goals/*` authored artifacts/views rather than a parallel live-state store.

Deliverables:

- journal projection for goal state
- `goal.yaml`, evidence, blockers, deferred state derived from journal
- authored artifacts remain file-based: `source.md`, `acceptance.md`, `plan.md`, `coverage.md`
- goal CLI reads live goal state through broker APIs
- `tests/test_goal_projection.py`

## Production-readiness criteria

V1 is not production-ready until:

- M1-M5 are implemented and tested
- full Python test suite passes
- compile checks pass
- manual smoke test passes end-to-end, including compaction disable-path and broker crash/stale-session recovery
- best-effort cancellation is documented in user-facing docs
- `doctor` reports relevant journal/lease/storage health
- version sync across `pyproject.toml`, `src/orchlink/__init__.py`, and `src/orchlink/broker/main.py` is enforced by tests

## Not in scope for v1

- guaranteed interruption of running tool calls
- HA broker or distributed storage
- multi-tenant authorization
- replacing JSONL with SQLite purely for perceived production maturity
- renaming existing CLI commands
- replacing the current manual smoke strategy
