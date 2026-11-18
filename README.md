# API migration agent with a retrieved guide

An LLM agent that migrates Python client code from a v1 to a v2 API, one file at a time,
retrieving sections of a migration guide as it goes. The question we tested:
**do retrieved guide sections beat the model's own guesses, and at what token cost?**

Built as the capstone (project P04) for CSP352 AI-Augmented Software Development at IIT Bhilai,
by group g15 (team GREP).

## Result

12 client files, 2 runs per variant (24 files each), same model (temperature 0), same bounds.
"Fully migrated" = the client's tests pass and no v1 calls are left.

| Variant | Fully migrated | Tokens per file | Calls per file |
|---|---|---|---|
| Baseline (no guide) | 22/24 | 3682 | 3.17 |
| Guide, first draft | 20/24 | 4411 (+19.8%) | 3.83 |
| Guide, after fixing 2 sections | **24/24** | **3090 (-16.1%)** | **3.00** |

The first guide did *worse* than no guide. The traces showed why: retrieval found the right
section, but the text was ambiguous (G02 did not say argument order was kept; G09 did not say a
default was removed). Rewriting those two sections fixed both failures.
Caveat: the guide was fixed after seeing results on these same 12 cases, so the last row is
optimistic. Full write-up, failure diagnosis and limitations: [CAPSTONE.md](CAPSTONE.md).

## How it works

- **Agent loop** (`harness/agent.py`): one loop per client file, at most 8 model calls. The model
  answers with one JSON tool call per step.
- **Tools** (`harness/tools.py`): `read_file`, `write_file`, `run_tests`, and in the improved
  variant `search_guide`: IDF-weighted keyword retrieval over `data/guide.json`
  (`harness/retrieval.py`, top 3 sections).
- **Checks in plain code, not the model**: the client's pytest file, and an AST check for leftover
  v1 calls (`harness/v1check.py`) that ignores comments and strings.
- **Safety gate** (`safety/scope.py`): every `write_file` is resolved with `realpath` and denied
  unless it targets the current client. Two adversarial clients try to get the agent to edit the
  library or its own tests; the gate blocks both.
- **Traces**: every run writes `traces/<run_id>.jsonl` (each step's tool, arguments, tokens and
  retrieved sections) and `eval/results/<run_id>.json`.

## Run it

Requires Python 3.9+ and an OpenAI-compatible endpoint.

```bash
pip install -r requirements.txt

# No API key needed: unit tests, and a check that each base client fails
# its tests and each hand migration passes
python -m pytest -q safety/test_scope.py harness/test_v1check.py data/libv/tests
python eval/check_hand.py

# Agent runs (we used a course gateway whose model was named "main"; pass --model for yours)
export OPENAI_BASE_URL=...   OPENAI_API_KEY=...
python -m harness.agent --variant baseline --model <model>
python -m harness.agent --variant improved --model <model> --only case_01,case_02

# Compare two runs case by case
python eval/compare.py eval/results/<baseline>.json eval/results/<improved>.json
```

## Layout

    harness/        agent loop, tools, retrieval, v1 check, model client
    safety/         write-scope gate and its tests
    data/libv/      the toy library: v1.py, v2.py
    data/clients/   12 v1 client modules and their v2 tests
    data/guide.json the migration guide (12 sections)
    eval/cases/     one case per client: needed sections, expected outcome
    eval/hand_migrations/  hand-written expected answers
    eval/results/   per-run results and baseline-vs-improved comparisons
    traces/         per-run JSONL traces
    CAPSTONE.md     design report    EVIDENCE.md  run and commit evidence

## Team

agastyan, paritoshl, varunr, yallappagarid. Who did what is in
[CAPSTONE.md, Contributions](CAPSTONE.md#contributions).

## License

[MIT](LICENSE)
