# Orchlink envelope contract

The Orchlink envelope is the stable JSON contract between the CLI, broker, and Pi extension.

Current envelope header:

```text
x-orchlink-envelope: 1
```

Current protocol field:

```text
orch-a2a-v1
```

## Versioning policy

Envelope version `1` allows additive optional fields. Renaming, removing, or changing the meaning of existing fields requires a future envelope version and explicit compatibility handling.

Unknown future protocol values are rejected instead of silently coerced.

## Message envelope

Required fields:

```text
protocol
message_id
correlation_id
project_id
conversation_id
from_agent
to_agent
type
status
turn
max_turns
requires_reply
timeout_seconds
delivery
payload
```

Optional fields:

```text
task_id
```

Chat messages use `type` values beginning with `CHAT_`, `delivery: conversation`, and `payload.mode: TALK`.

Task messages use `type: TASK` and non-conversation delivery.

## Source of truth

The canonical Python model lives in:

```text
src/orchlink/core/envelope.py
```

`src/orchlink/broker/protocol.py` re-exports the core contract for backward-compatible imports.
