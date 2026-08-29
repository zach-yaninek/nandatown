# Supply chain: byzantine corruption that both validators call clean

**One setting changed:** `failures.byzantine_agents`, `0.0` → `0.25`, on the bundled
`supply_chain` scenario. I left seed, agent count, round count, layer stack, and drop rate
at their shipped values.

**Result:** every message sent by the byzantine agent arrives as random bytes, and
both `supply_chain` validators report `PASS`. Every metric in `nest report` is
identical to the baseline. Three separate tools report a clean run.

---

## Why this setting?

`message_drop` is the knob the quickstart itself suggests, so I wanted to try
something different. `byzantine_agents` is the only failure mode in `FailureConfig` that
attacks message integrity instead of message delivery. I found this interesting
because I wanted to verify that the validators would catch this.

The other four settings are partition controls that need a nested `groups` dict, so
`byzantine_agents` was the only single scalar left that changes agent behavior
rather than the volume of the run.

## Hypothesis, recorded before running

From the `description` field of `my.yaml`, committed before I ran the experiment:

> Sets byzantine_agents = 0.25 to verify the validator will flag it.

I expected corrupted messages to break the supply chain's `material: → product: →
shipment: → delivered:` grammar, and `supply_chain_pipeline` to fail on the
malformed hops.

The commit order in this branch shows the prediction was recorded first: the
scenario followed by the trace that contradicts it.

## Results

```
PASS supply_chain_pipeline - all hops present
PASS supply_chain_no_lost  - 15/15 materials delivered
```

Both checks passed. The message volume did not change from the baseline: 128 events, 60
sends, 60 receives, nothing dropped.

However, the trace holds the same message twice, once as sent and once as delivered:

```jsonc
{"agent":"retailer-0","kind":"send","to":"supplier-0",
 "msg":"delivered:1:good-manufacturer-0-1","size":33}

{"agent":"supplier-0","kind":"receive","from":"retailer-0",
 "msg":"�����o�P�c�N\ba��l����d�3Xg","size":33}
```

`retailer-0` destroyed all 15 messages this way. The other three senders
are untouched (see the screenshot in the form submission and further below 
for the script I used):

| sender | garbled / total received |
|---|---|
| `supplier-0` | 0 / 15 |
| `manufacturer-0` | 0 / 15 |
| `distributor-0` | 0 / 15 |
| `retailer-0` | **15 / 15** |

`report_baseline.html` and `report_byzantine.html` are the `nest report` output for
the control and the experiment. Every metric matches exactly — `success_rate`
1.0000, `delivery_rate` 1.0000, `message_count` 120, `dropped_count` 0.

## Investigation

Two implementation details explain what I found.

**Corruption is applied at delivery.** The simulator records the send from the
original payload. It then scrambles a separate copy for the receiver
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

The validators reconstruct the protocol from the send records alone. A payload destroyed
between sender and receiver leaves that reconstruction perfectly intact. The
corruption is therefore invisible by construction.

This is not a limitation of the validator framework. Five validators
elsewhere in the same file inspect `receive` events (lines 2483, 2711, 2833,
4453, 4786). The capability exists and is used for other scenarios. But it currently
is not applied to `supply_chain`. That being said, a multi-hop chain of custody is 
an environment in which delivered-payload integrity matters a lot.

In summary, the validators do not check that a message arrived as sent. They
check that the sender claimed to send it. This byzantine corruption experiment 
demonstrated that only the latter (weaker) condition is verified.

## What the validators catch

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
nonzero value corrupts at least one whole agent.

At `0.50` and above, a mid-chain agent is necessarily included. It receives bytes it
cannot parse, fails to recognize them as a work order, and stops forwarding, so the
pipeline truncates from 60 sends to 45 and then 15.

Note the details of this failure. The validator did not detect corruption; it
detected the silence. It caught messages that were never sent because
a broken agent went quiet. The integrity violation stayed invisible at every fraction, though. 
Only its second-order consequence is ever caught. At `0.25`, the corrupted agent happens to sit at the terminal hop, where nothing downstream depends on its output. In that case, there is no
second-order consequence to leak, and the run reports clean.

## Hypothesis for the next experiment: likely seed sensitivity

This experiment held `seed: 42` fixed, since the brief asks for exactly one changed
setting. Also, `seed` is neither in `failures` nor `task.config`. But the mechanism
above suggests that the seed matters for this result.

At `0.25`, one of four agents is chosen as byzantine by shuffling the
agent list with the seeded failure RNG (`simulator.py:252`):

```python
n_byzantine = max(1, int(len(all_ids) * self._byzantine_fraction))
shuffled = list(all_ids)
self._failure_rng.shuffle(shuffled)
self._byzantine_agents = set(shuffled[:n_byzantine])
```

Consequently, a different seed is likely to corrupt a different agent. Combined
with the magnitude sweep above, which showed that corrupting a mid-chain agent
truncates the pipeline, I predict the following:

> The clean PASS/PASS result should reproduce only on seeds that happen to select
> `retailer-0`. Seeds selecting `supplier-0`, `manufacturer-0` or `distributor-0`
> should truncate the chain and fail `no_lost`, roughly one seed in four will preserve
> the finding, assuming selection is uniform across the four agents.

If so, the finding sharpens further. The same failure mode, at the same magnitude, 
reports pass or fail depending on which agent the RNG
picks. Reproducibility would be intact because of the seed, but the verdict itself 
relies upon the seed choice at scenario-authoring time. Twelve runs varying only
`seed` would confirm this.

## Suggested fix

Add a third `supply_chain` validator that reads `receive` events and compares 
the delivered and sent payloads with same `corr` ID. The trace records both, 
and five validators for other scenarios already iterate receives. Neither a new 
instrumentation nor a new pattern is required.

## Reproducing

Run from the repository root. Output goes to the gitignored root `traces/`, so it
will not overwrite the committed evidence:

```bash
nest run ./examples/supply-chain-byzantine-blind-spot/my.yaml
```

This reproduces `traces/my_supply_chain_scenario.jsonl` byte-for-byte identically to
the copy committed here (sha256 `59db3cc7…`).

Also, two things to watch regarding the validators. Use `python3.13` rather than 
`python3` if `nest` was installed under the python.org framework build. 
Pass `supply_chain` as the scenario key (not this scenario's own name). 
`validate_events` resolves checks via `VALIDATORS.get(scenario_type, [])`, so a 
mismatched name runs zero validators, prints nothing, and exits 0,
which looks like a pass.

```bash
python3.13 -c "
from pathlib import Path
from nest_core.validators import validate_trace
p = 'examples/supply-chain-byzantine-blind-spot/traces/my_supply_chain_scenario.jsonl'
for r in validate_trace(Path(p), 'supply_chain'):
    print(('PASS' if r.passed else 'FAIL'), r.name, '-', r.detail)
"
```

And to see what that misses:

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
this branch's source (`validators.py` has grown since). The two `supply_chain`
validators are byte-identical between the two, so the results are reproducible 
regardless. But line numbers cited above refer to this branch's source rather than
the wheel.

## Files

| file | what it is |
|---|---|
| `my.yaml` | the scenario, one setting changed, hypothesis in the description |
| `traces/supply_chain.jsonl` | control — the unmodified baseline run |
| `traces/my_supply_chain_scenario.jsonl` | experiment — the byzantine run |
| `report_baseline.html` | `nest report` over the control |
| `report_byzantine.html` | `nest report` over the experiment, metrics identical |

## Relation to other submissions

Two open PRs overlap in part:

- **#89** runs a `byzantine_agents` experiment on `marketplace` rather than
  `supply_chain`.
- **#87** reports a validator blind spot on `consensus`, but one related to liveness.

The class of failure is distinct. An integrity violation that no
validator can observe at any magnitude. Further, the multi-hop chain makes the
difference between a caught and an uncaught failure depend on the corrupted agent's
position in the pipeline.

## Use of AI

I used Claude Code running Opus 5 on Ultracode throughout. The specifics are below,
and I attached a sample prompt in the form submission.

- **Mine:** choosing `byzantine_agents` as the setting and the reasoning for it,
  writing the hypothesis in `my.yaml` before running the experiment, and deciding
  to investigate the unexpected (no visible failure) result.
- **Claude's:** confirming my understanding of the repo's layout and reading
`simulator.py` and `validators.py` to locate the failure mechanism.
I also had it run the magnitude sweep, generate the comparison report, and 
draft the first version of this README.
- **Note:** my finding did not come from reading the run output.
  The run output looks completely normal, which is the point. It came from
  reading the validator source to understand why it was not flagged.

Claude also diagnosed two unrelated packaging problems while setting up: `nest
dashboard` cannot find `apps/dashboard/index.html` from a pip-only install because
the wheel does not ship it, and the quickstart's `python -c` snippet fails on macOS
where only `python3` exists. Neither is part of this experiment.
