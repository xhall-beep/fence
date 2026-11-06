# Fence — Runbook

Fence is the estate's split-brain guard for two-node runit services. It answers
one question — "am I the active node, or am I fenced off?" — as a single cached
verdict that services and scripts can check before acting.

## Components

- `bin/fence` — probes the peer node's published epoch (via `FENCE_PEER_URL`)
  and prints one verdict. Verdicts are cached for 20 seconds (TTL) and each
  probe is recorded in a history log.
- `bin/fence-retry` — restart guard for a fenced service: retries up to
  10 times at 60-second intervals, then emits one ESCALATION line and backs
  off for 900 seconds instead of looping forever.

## Verdicts (exit codes from `bin/fence`)

| Code | Meaning |
|------|---------|
| 0 | Active — this node holds the current epoch; proceed. |
| 1 | Fenced — the peer holds the epoch; do not act as primary. |
| 2 | Unprobed — no verdict yet; probe before trusting either way. |
| 3 | Config error — fence itself is misconfigured (e.g. no peer URL). |

## Configuration

- `FENCE_PEER_URL` — the URL where the peer node publishes its epoch.
  Without it, `fence` returns verdict 3 (config error).

## Standing up a fenced node (`--standup-ok`)

Promotion is deliberately hard. `fence --standup-ok` allows a fenced node to
stand up as active only when BOTH conditions hold:

1. The peer has been unreachable for at least 4 hours.
2. The operator confirms — twice.

If either condition is missing, stay fenced. A node that promotes itself
early is how split-brain happens; the 4-hour wait is the price of being sure
the peer is dead rather than merely quiet.

## When you see an ESCALATION line

`fence-retry` has burned its 10 retries and gone quiet for 900 seconds.
That means the service is down AND fence refused to keep restarting it —
a human (or the primary agent) needs to look: check the peer's epoch,
check why the service will not stay up, then restart it deliberately.
Do not just clear the backoff and walk away.
