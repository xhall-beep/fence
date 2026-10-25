# fence

**One cached verdict for the question every HA script gets wrong: may this node
serve right now?**

Two machines can both hold the same role - hub, coordinator, writer, job
runner. If the only rule is "start if the peer is down", the node that reboots
first wins, and for a while you have two writers. `fence` gives every component
the same answer, so you only fix the split-brain window once.

```
active    (exit 0)  this node may serve
fenced    (exit 1)  the peer holds a higher epoch - stand down
unprobed  (exit 2)  the peer did not answer - you choose fail-open or closed
```

The mechanism is deliberately boring: each node publishes `{"epoch": N}`
somewhere the other can read it (a file server, an HTTP endpoint, a KV key).
Higher epoch wins. Equal or lower means the other node is standby. Epochs are
never raised automatically - promotion is a deliberate act.

## Install

Two scripts, no dependencies beyond `bash` and `curl`:

```sh
install -m 755 bin/fence bin/fence-retry ~/.local/bin/
```

## Use

```sh
export FENCE_PEER_URL="http://peer:8471/kv/get?key=ops/epoch"
echo 1 > ~/.agent-ops/local-epoch

fence                  # human-readable
fence -q; echo $?      # 0 active / 1 fenced / 2 unprobed
fence --json           # one-line verdict
fence --no-cache       # force a probe
fence --history        # last 20 probes
fence --self-test      # four checks, local stub, touches nothing
```

Verdicts are cached for `FENCE_TTL` seconds (default 20), so six components
make one probe per TTL instead of six. Only fresh probes are recorded in
`fence-history.jsonl`.

In a service runner:

```sh
fence -q
case $? in
  0) exec my-server ;;                    # active
  1) exec fence-retry ;;                  # fenced: capped backoff, not a hot loop
  *) exec fence-retry ;;                  # unprobed: fail closed here
esac
```

`fence-retry` retries ten times at 60 s, then backs off to 15 minutes and logs
one `ESCALATION` line. A repeating restart is the incident; do not amplify it.

## Why free

The fence is the part that prevents data corruption, and it is small enough to
read in one sitting. The rest of the kit - the service watchdog, the orphan
reaper, the restart cap and the lesson ledger - is the [Agent Ops
Kit](https://montsvontz.gumroad.com/l/agent-ops-kit), a paid product extracted from the same
deployment.

## License

MIT. See [LICENSE](LICENSE).
