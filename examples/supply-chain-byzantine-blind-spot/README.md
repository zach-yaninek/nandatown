# Supply chain: byzantine corruption that both validators call clean

**One setting changed:** `failures.byzantine_agents`, `0.0` → `0.25`, on the bundled
`supply_chain` scenario. Seed, agent count, round count, layer stack and drop rate
are all left at their shipped values.

**Result:** every message sent by the byzantine agent arrives as random bytes, and
both `supply_chain` validators report `PASS`. Every metric in `nest report` is
identical to the baseline. Three separate tools report a clean run.

---

## Why I chose this setting

`message_drop` is the knob the quickstart itself suggests, so I wanted something
less travelled. `byzantine_agents` is the only failure mode in `FailureConfig` that
attacks **message integrity** rather than message **delivery**, which made it the
more interesting test of what the validators actually check.

The other four settings are partition controls that need a nested `groups` dict, so
`byzantine_agents` was the only single scalar left that changes agent behaviour
rather than the volume of the run.

## Hypothesis, recorded before running

From the `description` field of `my.yaml`, committed before the experiment was run:

> Sets byzantine_agents = 0.25 to verify the validator will flag it.

I expected corrupted messages to break the supply chain's `material: → product: →
shipment: → delivered:` grammar, and `supply_chain_pipeline` to fail on the
malformed hops.

The commit order in this branch shows the prediction was recorded first: the
scenario lands in one commit, the trace that contradicts it in the next.

## What actually happened

```
PASS supply_chain_pipeline - all hops present
PASS supply_chain_no_lost  - 15/15 materials delivered
```

Both checks pass. Message volume is unchanged from the baseline: 128 events, 60
sends, 60 receives, nothing dropped.

But the trace holds the same message twice, once as sent and once as delivered:

```jsonc
{"agent":"retailer-0","kind":"send","to":"supplier-0",
 "msg":"delivered:1:good-manufacturer-0-1","size":33}

{"agent":"supplier-0","kind":"receive","from":"retailer-0",
 "msg":"�����o�P�c�N\ba��l����d�3Xg","size":33}
```

All 15 messages from `retailer-0` are destroyed this way. The other three senders
are untouched:

| sender | garbled / total received |
|---|---|
| `supplier-0` | 0 / 15 |
| `manufacturer-0` | 0 / 15 |
| `distributor-0` | 0 / 15 |
| `retailer-0` | **15 / 15** |

`report_baseline.html` and `report_byzantine.html` are the `nest report` output for
the control and the experiment. Every metric matches exactly — `success_rate`
1.0000, `delivery_rate` 1.0000, `message_count` 120, `dropped_count` 0.

## Investigation: why nothing sees it

Two implementation details combine, neither wrong on its own.

**Corruption is applied at delivery.** The simulator records the send from the
original payload, then scrambles a separate copy for the receiver
(`packages/nest-core/nest_core/sim/simulator.py:384`):

```python
delivered_payload = event.payload
if event.target_id in self._byzantine_agents:
    delivered_payload = bytes(
        (b ^ self._failure_rng.randint(0, 255)) for b in event.payload
    )
```

**The supply chain validators only read sends.** Both functions registered for
`supply_chain` — `validate_supply_chain_pipeline` (`validators.py:720`) and
`validate_supply_chain_no_lost` (`validators.py:775`) — open with the same guard:

```python
for ev in events:
    if ev.get("kind") != "send":
        continue
```

So they reconstruct the protocol from send records alone. A payload destroyed
between sender and receiver leaves that reconstruction perfectly intact. The
corruption is invisible **by construction**, not by bad luck.

Worth being precise about the scope, because it makes the gap sharper rather than
softer: this is *not* a limitation of the validator framework. Five validators
elsewhere in the same file do inspect `receive` events (lines 2483, 2711, 2833,
4453, 4786). The capability exists and is used for other scenarios. It simply is
not applied to `supply_chain`, where a multi-hop chain of custody is exactly the
setting in which delivered-payload integrity matters most.

Put precisely: the validators do not check that a message *arrived* as sent. They
check that the sender *claimed* to send it. Under byzantine corruption those are
different facts, and only the weaker one is verified.

## What the validators do catch

Raising the same setting shows where the blind spot ends. The simulator marks
`max(1, int(agents * fraction))` agents, so a four-agent chain moves in whole
agents:

| `byzantine_agents` | agents | sends | pipeline | no_lost |
|---|---|---|---|---|
| 0.01 | 1 | 60 | PASS | PASS |
| **0.25** | **1 — retailer-0** | **60** | **PASS** | **PASS** |
| 0.50 | 2 | 45 | PASS | **FAIL** — 15 sent, 0 delivered |
| 0.75 | 3 | 15 | PASS | **FAIL** — 15 sent, 0 delivered |

`0.01` and `0.25` produce byte-identical traces, since `max(1, ...)` means any
nonzero value corrupts at least one whole agent. There is no gentle setting here.

At `0.50` and above a mid-chain agent is necessarily included. It receives bytes it
cannot parse, fails to recognise them as a work order, and stops forwarding, so the
pipeline truncates from 60 sends to 45 and then 15.

Note what that failure actually is. The validator did not detect corruption. It
detected **silence** — messages never sent because a broken agent went quiet. The
integrity violation stays invisible at every fraction; only its second-order
consequence is ever caught. At `0.25` the corrupted agent happens to sit at the
terminal hop, where nothing downstream depends on its output, so there is no
second-order consequence to leak and the run reports clean.

## Hypothesis for the next experiment: this should be seed-sensitive

This experiment held `seed: 42` fixed, since the brief asks for exactly one changed
setting and `seed` is neither in `failures` nor `task.config`. But the mechanism
above suggests the seed is load-bearing rather than incidental.

At `0.25` exactly one of four agents is marked byzantine, chosen by shuffling the
agent list with the seeded failure RNG (`simulator.py:252`):

```python
n_byzantine = max(1, int(len(all_ids) * self._byzantine_fraction))
shuffled = list(all_ids)
self._failure_rng.shuffle(shuffled)
self._byzantine_agents = set(shuffled[:n_byzantine])
```

A different seed shuffles differently and should corrupt a different agent. Combined
with the magnitude sweep above, which showed that corrupting a mid-chain agent
truncates the pipeline, the prediction is:

> The clean PASS/PASS result should reproduce only on seeds that happen to select
> `retailer-0`. Seeds selecting `supplier-0`, `manufacturer-0` or `distributor-0`
> should truncate the chain and fail `no_lost` — roughly one seed in four preserving
> the headline, if selection is uniform across the four agents.

If that holds, the finding is sharper than a single run suggests: **the same failure
mode, at the same magnitude, reports pass or fail depending on which agent the RNG
picks.** Reproducibility would be intact, since the seed guarantees it, while the
verdict itself would be fixed by a coin flip at scenario-authoring time. Twelve runs
varying only `seed` would settle it.

## Suggested fix

Narrow, and with precedent already in the file. Add a third `supply_chain` validator
that reads `receive` events and compares the delivered payload against the sent
payload for the same `corr` ID. The trace records both, and five validators for
other scenarios already iterate receives, so neither new instrumentation nor a new
pattern is needed.

## Reproducing

Run from the repository root. Output lands in the gitignored root `traces/`, so it
will not overwrite the committed evidence:

```bash
nest run ./examples/supply-chain-byzantine-blind-spot/my.yaml
```

That reproduces `traces/my_supply_chain_scenario.jsonl` byte-for-byte identically to
the copy committed here (sha256 `59db3cc7…`).

Then the validators. Two things to watch: use `python3.13` rather than `python3` if
`nest` was installed under the python.org framework build, and pass `supply_chain`
as the scenario key, **not** this scenario's own name. `validate_events` resolves
checks via `VALIDATORS.get(scenario_type, [])`, so a mismatched name runs zero
validators, prints nothing, and exits 0 — which looks exactly like a pass.

```bash
python3.13 -c "
from pathlib import Path
from nest_core.validators import validate_trace
p = 'examples/supply-chain-byzantine-blind-spot/traces/my_supply_chain_scenario.jsonl'
for r in validate_trace(Path(p), 'supply_chain'):
    print(('PASS' if r.passed else 'FAIL'), r.name, '-', r.detail)
"
```

And to see what those two green lines are hiding:

```bash
python3.13 -c "
import json
from collections import Counter
g=Counter(); t=Counter()
p='examples/supply-chain-byzantine-blind-spot/traces/my_supply_chain_scenario.jsonl'
for l in open(p):
    e=json.loads(l)
    if e['kind']=='receive':
        t[e['from']]+=1
        if chr(0xFFFD) in e.get('msg',''): g[e['from']]+=1
for k in sorted(t): print(f'{k:<18}{g[k]:>3} garbled / {t[k]}')
"
```

Note that `.gitignore` excludes `traces/` at any depth, so the traces here were
added with `git add -f`. Regenerating and re-committing them needs the same flag.

**Versions.** The run above used `nest-core` 0.1.4 from PyPI, which is older than
this branch's source — `validators.py` has grown since. The two `supply_chain`
validators are byte-identical between the two, so the results reproduce either way,
but line numbers cited above refer to this branch's source rather than the wheel.

## Files

| file | what it is |
|---|---|
| `my.yaml` | the scenario, one setting changed, hypothesis in the description |
| `traces/supply_chain.jsonl` | control — the unmodified baseline run |
| `traces/my_supply_chain_scenario.jsonl` | experiment — the byzantine run |
| `report_baseline.html` | `nest report` over the control |
| `report_byzantine.html` | `nest report` over the experiment, metrics identical |

## Relation to other submissions

Two open PRs overlap in part, and this is not a claim to be first:

- **#89** runs a `byzantine_agents` experiment on `marketplace` rather than
  `supply_chain`.
- **#87** reports a validator blind spot on `consensus`, but a liveness one.

What is distinct here is the class of failure: an **integrity** violation that no
validator can observe at any magnitude, where the multi-hop chain makes the
difference between a caught and an uncaught failure depend on the corrupted agent's
position in the pipeline.

## Use of AI

I used Claude (Anthropic) heavily throughout, and the specifics matter for judging
what is mine:

- **Mine:** choosing `byzantine_agents` as the setting and the reasoning for it,
  writing the hypothesis in `my.yaml` before running, and deciding the result was
  worth investigating rather than filing as a pass.
- **Claude's:** reading `simulator.py` and `validators.py` to locate the mechanism —
  the delivery-time corruption and the send-only guard, which is what turned "the
  validator didn't fire" into an explained finding rather than a shrug. It also ran
  the magnitude sweep, generated the comparison report, and drafted this README.
- **Worth stating plainly:** the finding did not come from reading the run output.
  The run output looks completely normal, which is the entire point. It came from
  reading the validator source to find out *why* nothing fired.

Claude also diagnosed two unrelated packaging problems while setting up: `nest
dashboard` cannot find `apps/dashboard/index.html` from a pip-only install because
the wheel does not ship it, and the quickstart's `python -c` snippet fails on macOS
where only `python3` exists. Neither is part of this experiment.
