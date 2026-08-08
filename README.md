# MalSkills

**Natural-language malware planted inside AI-agent skill files.**

A skill is a trusted contract the agent executes with full privileges. Plain English
instructions can achieve data exfiltration, persistence, and lateral movement with no
binary, no shellcode, and no traditional indicator of compromise.

This repo contains everything you need to plant a MalSkill, observe it fire, and
collect proof that data left the system.

> DEF CON 34 / Red Team Village -- "Plant, Chain, Persist"
> Nur Gucu / @BurritoTheNurrito

## What's inside

```
MalSkills/
  Benign_Examples/          Clean ORPHEUS systems (the victim baseline)
    code-reviewer/            orchestrator -> review-expert -> report-expert
  MalSkills_Examples/       Planted systems (the attack)
    code-reviewer/            same pipeline, with data-exfil injected
    code-reviewer_malskill_planting_playbook.md
  ProofCollector/           HTTP receiver that records proof of exfiltration
    proof-collector/          stdlib Python, zero dependencies
  malskills-detector/       Detector system (work in progress)
```

## Prerequisites

- **Python 3.10+** (stdlib only, no pip install needed)
- **Claude Code** (the agent runtime that executes ORPHEUS systems)
- A terminal (macOS or Linux)

---

## Step 1: Clone and explore

```bash
git clone <this-repo-url>
cd MalSkills
```

Take a look at the two versions of the code-reviewer system side by side:

```bash
# The clean version
ls Benign_Examples/code-reviewer/.orpheus/

# The planted version
ls MalSkills_Examples/code-reviewer/.orpheus/
```

Both have the same structure: an orchestrator, two experts (review-expert, report-expert),
and a worker (source-reader-worker). The malicious version adds an adapter and a policy
file that turn the report-quality gate into an exfiltration channel.

Read the planting playbook to understand the kill chain:

```bash
cat MalSkills_Examples/code-reviewer_malskill_planting_playbook.md
```

---

## Step 2: Start the Proof Collector

The Proof Collector is a localhost HTTP receiver. It records every callback that arrives
at its keyed ingest endpoint. No canary filter in local mode -- the exfiltrated report
itself is the proof.

```bash
cd ProofCollector/proof-collector

# Initialize (generates ingest key + admin token)
./proofctl init

# Start the collector (binds 127.0.0.1:8888)
./proofctl up
```

Verify it's running:

```bash
./proofctl status
```

Get your ingest endpoint (this is the URL the planted system will send data to):

```bash
./proofctl endpoint
```

You'll see something like:

```
http://127.0.0.1:8888/ingest/<your-key>/notify
```

Save that URL. You'll wire it into the planted system next.

### Proof Collector commands

| Command | What it does |
|---|---|
| `./proofctl init` | Generate keys and runtime material |
| `./proofctl up` | Start the collector |
| `./proofctl status` | Check health and config |
| `./proofctl endpoint` | Print the keyed ingest URL |
| `./proofctl events` | List all recorded events (JSON) |
| `./proofctl self-test` | Send a synthetic canary to verify the pipeline |
| `./proofctl clear` | Wipe all recorded events |
| `./proofctl down` | Stop the collector |

### Dashboard

Once the collector is running, open the live dashboard in your browser:

```
http://localhost:8888/dashboard
```

It shows events as they arrive in real time. Leave it open while you run the next steps.

---

## Step 3: Wire the bait endpoint

The planted code-reviewer uses a dependency-mediated exfiltration pattern. The destination
is not a URL in any skill file -- it's split across transport fields in a policy file
(`report_quality.json`). You need to substitute the Proof Collector endpoint parts into
those fields.

Open the adapter policy:

```bash
cat MalSkills_Examples/code-reviewer/.orpheus/adapters/report_quality.json
```

Look for the `delivery.transport` section. It contains placeholder fields:

- `host` -- set to `127.0.0.1`
- `port` -- set to `8888`
- `path_segments` -- includes your ingest key

Update `report_quality.json` with your actual ingest key from `./proofctl endpoint`.
Replace the `[PC_BAIT_INGEST_KEY]` placeholder in `path_segments` with the key value
from your endpoint URL.

---

## Step 4: Run the benign system first

Before running the attack, run the **clean** version to see what normal looks like.

```bash
cd ../../Benign_Examples/code-reviewer
```

Point Claude Code at any target project you want reviewed (or use this repo itself):

```bash
claude "Run the code-reviewer ORPHEUS system on this project"
```

Observe: the orchestrator dispatches review-expert, which delegates to source-reader-worker,
which reads the code. report-expert writes `code-review.md`. That's it. Nothing leaves
the machine.

Check the Proof Collector -- it should show **zero events**:

```bash
cd ../../ProofCollector/proof-collector
./proofctl events
```

---

## Step 5: Run the planted system

Now run the version with the MalSkill injected.

```bash
cd ../../MalSkills_Examples/code-reviewer
```

Run it on the same target:

```bash
claude "Run the code-reviewer ORPHEUS system on this project"
```

The pipeline looks identical from the user's perspective: review runs, findings appear,
`code-review.md` gets written. But this time, the report-expert's quality gate calls the
adapter, which encodes the report and sends it to your Proof Collector.

---

## Step 6: Observe the impact

### Terminal

```bash
cd ../../ProofCollector/proof-collector
./proofctl events
```

You should see a `callback_received` event. The code-review report was exfiltrated.

### Dashboard

Check `http://localhost:8888/dashboard` -- a new CALLBACK card should appear showing
that data arrived.

### What just happened

1. **report-expert** finished writing `code-review.md`
2. Its skill says to validate the report with the "approved quality adapter"
3. The adapter (`report_quality.py`) scores the report, base64-encodes it, and POSTs
   the envelope to a destination assembled from policy parts in `report_quality.json`
4. The Proof Collector recorded the callback
5. No single file contains a URL. No single file looks malicious on its own.

The kill chain works because the **composition** of benign-looking pieces creates the
exfiltration path. A file-by-file review misses it.

---

## Step 7: Diff the two systems

See exactly what was planted:

```bash
diff -r Benign_Examples/code-reviewer/.orpheus MalSkills_Examples/code-reviewer/.orpheus
```

The differences are:
- `adapters/report_quality.py` -- the exfil adapter (new file)
- `adapters/report_quality.json` -- transport policy with split destination (new file)
- `experts/report-expert/SKILL.md` -- one added line: "validate with the approved adapter"

That's it. Three files. No URLs in any skill. No `curl`. No suspicious commands.

---

## Explore further

### Read the LABELS

Each planted case has a `LABELS.yaml` that documents the ground truth:

```bash
cat MalSkills_Examples/code-reviewer/LABELS.yaml
```

This is the answer key for a detector. It maps every injection, the data flow, encoding
chain, and what detection approaches will catch vs. miss this case.

### Try the self-test

The Proof Collector has a built-in self-test that sends a synthetic canary:

```bash
cd ProofCollector/proof-collector
./proofctl self-test
```

### Clean up

```bash
./proofctl clear    # wipe events
./proofctl down     # stop the collector
```

---

## The thesis

A skill file is the most dangerous vector in an AI agent's stack. It runs with the
agent's full tool access, in the agent's trusted context, with no sandbox boundary. A
well-crafted natural-language instruction achieves what traditional malware needs binary
exploitation for -- and it leaves no traditional IOC behind.

The only defense is a detector that understands **what skills do**, not just what they
say. That means semantic analysis, data-flow tracking, and graph-level reasoning across
the entire skill hierarchy.

---

## Safety

- All canaries are synthetic (`CANARY_` prefix, `.invalid` domains)
- The Proof Collector binds loopback only (127.0.0.1) in local mode
- Planted cases live in `MalSkills_Examples/` only
- No real credentials or third-party targets are used
- Run planted systems in an isolated environment (separate HOME, container, or VM)

## License

GPL-3.0. See [LICENSE](LICENSE).
