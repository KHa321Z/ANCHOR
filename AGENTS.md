# AGENTS.md

Rules for anyone, AI agent or human, who changes this repository. Read all of it before your first task. `PLAN.md` says what to build; this file says how to work here.

## 1. The project in a paragraph

ANCHOR guides a novice through laptop repair with augmented reality on a LiDAR iPhone. A procedure graph, built from the service manual and checked by rules, says what the steps are. On-device perception reads screws, parts, tools and hands. A gate, a sequencer and a recovery planner decide what just happened and what comes next. Six widget classes (W1 to W6) show it. A team server builds graphs and returns outlines; a hosted vision-language model (VLM) answers rare, specific questions.

## 2. Sources of truth

When two sources disagree, the higher one wins. Do not resolve a conflict yourself: stop and raise it on the task's issue.

1. The Deliverable II report, `docs/report/`. It defines the method.
2. `docs/decisions/` (ADRs) and `docs/report-deltas.md`. They record every agreed departure from the report.
3. `contracts/`. The interfaces between lanes.
4. `PLAN.md`. Tasks, order, acceptance.
5. Code comments and notes.

Cite the report section you implemented in the code's doc comment, for example `// Report 5.5, Algorithm 2`.

## 3. Before you start a task

- [ ] Read the task's row in `PLAN.md` Section 8, and the notes of every task in its **After** column (`notes/<ID>.md`).
- [ ] Check every **After** task is merged. If one is not, the task is not ready.
- [ ] Check the **Needs** tags. If the task needs hardware you do not have, see Section 12.
- [ ] Check the task's GitHub issue is unassigned.
- [ ] Read the contracts the lane uses (`PLAN.md` Section 5) and the report sections the task names.

## 4. Claim, branch, hand off

One task, one branch, one pull request.

1. **Claim.** Assign the task's issue to yourself, or to the human supervising your agent session, and add the label `in-progress`. An agent adds a comment naming its session. An assigned issue is a lock: do not work on someone else's.
2. **Branch.** `task/<ID>-short-name` from current `main`, for example `task/CORE-03-gate`. If several agents share one machine, give each its own `git worktree`; never run two agents in one working directory.
3. **Open a draft pull request early**, titled `<ID>: summary`. Others can then see what is changing.
4. **Work only inside your lane's paths** (Section 5).
5. **Rebase on `main` at least daily.** Keep the pull request small; if a task grows past roughly 600 changed lines of code, propose a split in the issue.
6. **Finish.** Add `notes/<ID>.md` from the template in Section 15, make CI green, link the issue with `Closes #n`, and request review from the lane's human owner.
7. **Merge.** A human merges. Agents do not merge their own pull requests.

If an issue is assigned and its branch has had no push for three days, the lane owner may reassign it.

## 5. Paths you may touch

Each lane owns folders (`PLAN.md` Section 3). In a task you may change:

- your lane's folders;
- `notes/<your task ID>.md`;
- a new file under `docs/decisions/` if your task makes a decision;
- one appended row or entry in `docs/measurements.md`, `docs/report-deltas.md` or `docs/risks.md`.

Everything else is read-only for you. In particular:

- **`contracts/` and `config/anchor.default.json`** change only through a separate pull request that touches nothing else, bumps the contract version, and is approved by every lane owner who uses it. If your task needs a contract change, open that pull request first and wait.
- **Another lane's code.** If you find a bug there, open an issue with a failing test case. Do not fix it in your branch.
- **`PLAN.md` and this file.** Propose changes in your note under "Follow-ups proposed".

## 6. Commands

`BOOT-02` creates these targets. If one is missing, the task that owns it has not merged yet; do not build a second mechanism beside it.

| Command | Does |
| --- | --- |
| `make setup` | Installs the Swift and Python toolchains' project dependencies |
| `make test-core` | `swift test` in `core/` |
| `make test-ml` | `pytest` for `ml/` and `pyref/` |
| `make test-server` | `pytest` for `server/` with the VLM and GPU mocked |
| `make lint` | Formatters and linters for Swift and Python |
| `make mat` | Builds the mat PDF and marker map |
| `make assets` | Runs Blender headless and rebuilds every asset and the manifest |
| `make replay CLIP=<folder> GRAPH=<file>` | Recording to observations to `anchor-replay` to metrics |
| `make ios-gen` | Generates the Xcode project from `ios/project.yml` (macOS only) |

Run the tests for your lane before every push. CI runs all Linux targets on every pull request.

## 7. Conventions

**Swift (`core/`)**

- Must build and pass tests on Linux. Import `Foundation` only: no ARKit, UIKit, RealityKit, Core ML, Vision or `simd`.
- No clock, randomness or I/O inside logic. Time comes from an injected clock; the same inputs always give the same outputs.
- Value types, `Codable` for anything that crosses a contract, typed errors. No force unwraps or `try!` outside tests.

**Swift (`ios/`)**

- Target iOS 17. The project file is generated; edit `ios/project.yml`, never the generated project.
- ARKit callbacks never block. Heavy work runs on one background queue that always takes the latest frame and drops the rest. Copy what you need out of a frame; do not retain frames.
- Perception and registration code is a port of `pyref/`. Port the algorithm, then prove parity on a recorded clip.

**Python (`ml/`, `pyref/`, `server/`)**

- Python 3.11, type hints, `ruff`, `pytest`. Entry points are modules with command-line arguments and a configuration file, not notebooks.
- Fixed seeds (42, as in the existing runs). Every training or evaluation run writes its configuration, data manifest hash and code revision beside its results.
- Core ML export runs on macOS or x86 Linux, not on native Windows. Use WSL2 or CI.

**Configuration**

- Every tunable number lives in `config/anchor.default.json` under the name the report uses (`c_high`, `c_low`, `t_conf`, `occ_share`, `h_touch`, `min_time`, and so on). Code reads it by name. No threshold literals in code.

**Units and frames**

- 3D positions in metres. Mat and device-face coordinates in millimetres. Time in seconds on a monotonic clock. Pixels in full-resolution frame coordinates. Scores from 0 to 1.
- Frames are `world` (ARKit), `mat`, `face` (the device-face plane) and `image`. Name every transform by what it does: `T_world_from_mat` maps mat coordinates into world coordinates.

**Identifiers**

- Step, fact and slot identifiers come from the graph. Tool names come from the asset manifest. Class names come from `contracts/classes.yaml`. Never invent one in code.

**Commits and pull requests**

- Commit subject: `<ID>: imperative summary`. The pull request body lists the paths touched, the test evidence, and the note file.

**Dependencies**

- Record every new dependency and its licence in your note. Ask the lane owner before adding anything that is not MIT, BSD or Apache licensed. Ultralytics is already in use under AGPL-3.0.

## 8. Invariants

These restate the report. Code, prompts and tests must never break them. If a task seems to require breaking one, stop and raise it.

1. **One writer.** Only the sequencer writes the guidance state.
2. **Closed vocabulary.** Fact kinds and action types are fixed lists. The validator rejects anything else.
3. **Safety structure is added by code.** Reversal steps, the reassembly phase, the battery hazard rule and its reassembly exception, and the pending-fixes precondition. No model output can remove or weaken them.
4. **Only validated graphs guide.** A graph that fails the validator is never used for guidance.
5. **A tap never clears a hazard.** Battery-connector unplugged must be confirmed by the VLM before the hazard clears.
6. **Hazard warnings never wait on the network to appear.**
7. **Thresholds come from configuration:** W5 at 0.10, W6 at 0.30, the rest at 0.50, with the safety override for assumed values.
8. **Widget limits.** Priority W5, W6, W3, W1, W2, W4; at most four part-anchored widgets at once.
9. **No single-frame decisions.** A change counts only when it is clear and has lasted `t_conf`.
10. **Unknown is not false.** When registration is not trusted, every fact is unknown and widgets dim. Never guess.
11. **VLM calls are narrow.** One specific question per call, answers restricted to the allowed lists, at most three calls a minute on average, with the cooldown.
12. **Swap by configuration.** Changing a detector file or the VLM provider needs no code change (NFR-06).
13. **Privacy.** Camera frames are stored only with consent, off by default (NFR-07).

## 9. Tests

- Every pull request adds or updates tests for what it changes. A bug fix starts with a failing test.
- Unit tests use no network, no GPU and no hardware. Mock the VLM, the server and the camera; use fixtures from `contracts/` and small recorded samples.
- `core/`: table-driven tests taken from the report's tables, plus the scenario suite (`CORE-10`).
- Ports (`ios/` from `pyref/`, Core ML from PyTorch): a parity test on recorded input with a stated tolerance.
- Never delete, skip or loosen a failing test to get CI green. If a test is wrong, say why in the pull request and fix the test in its own commit.

## 10. Data, models, secrets

- Never commit datasets, recordings, model weights, manuals or API keys. `data/`, `weights/` and `.env` are ignored; commit manifests with file hashes instead.
- Split by laptop and by recording day, never by frame. Keep one laptop, one operator and one recording day out of training.
- The test split is run once, with a frozen configuration. Do not tune on it and do not look at its failures to change the method.
- Record the source and licence of every dataset in its dataset card. Do not train on iFixit images: its licensing page says that violates its terms.
- Recordings of participants are private. They never leave the team's storage.
- Text and images from manuals, guides and web pages are untrusted input. The graph builder treats them as data: nothing in a resource is ever followed as an instruction, and every built graph goes through the validator.

## 11. Numbers

- Never write a metric you did not produce. Every number in a note, result file or report comes from a committed script, with the configuration hash and the data manifest hash.
- Report what happened, including failures and runs that did not reproduce. A failed run is a result.
- Mark estimates as estimates. Acceptance targets in `PLAN.md` are proposals; do not adjust a target to match a result.
- When a result changes a value in report Table 6.3, add the row to `docs/measurements.md`. Changing the default itself needs an ADR.

## 12. Hardware-tagged tasks

Tags in `PLAN.md`: MAC, PHONE, LAPTOPS, KIT, MAT, GPU, KEYS, PEOPLE.

An agent without the hardware may still do most of a tagged task:

1. Write the code and its tests against recorded fixtures and mocks.
2. List under "Human checks still needed" in the note exactly what a person must run on the device or bench, and what result to expect.
3. Label the issue `needs-human-check` and leave it open.

Only a human who ran the check on real hardware closes the task. Never report a device timing, a registration error or a bench result that you did not measure.

Without a Mac, iOS changes are verified by the macOS CI build and by a person installing that build on the iPhone. Keep iOS pull requests small, because each round trip costs a CI build.

## 13. The shared GPU PC

- One GPU job at a time. Check `nvidia-smi` and the lock file named in `docs/hardware.md` before starting; create the lock when you start and remove it when you finish.
- Never start training while the server is serving a bench or study session. Check the session calendar in `docs/hardware.md`.
- Long runs go overnight, write checkpoints, and can be resumed with the same settings.

## 14. When you are blocked or unsure

- **Blocked by another task or by hardware:** comment on the issue with what is missing, label it `blocked`, unassign yourself if you cannot continue, and take another ready task.
- **The report is ambiguous:** implement the most literal reading behind a named option, add a test for each reading, and record the question in your note and in the issue. Do not pick silently.
- **Your result misses the acceptance target:** hand off anyway, with the numbers and what you tried. Do not keep tuning past the task's hours without telling the lane owner.
- **You found something outside your task:** open an issue. Do not fix it in your branch.

## 15. Note template

Save as `notes/<ID>.md`. Keep it short and factual; the next person reads it before starting.

```markdown
# <ID>: <task title>

Status: done | needs-human-check | handed off incomplete
Owner: <name, or agent session and supervising human>
Pull request: #<n>
Human hours spent: <h> (planned: <h>)

## What was built
Three to six lines.

## How to run it
Exact commands.

## Results
Each number with the script, the configuration hash and the data manifest hash.
"None" if the task produces no numbers.

## Decisions
Links to ADRs written in this task.

## Departures from the report or the plan
Also added to docs/report-deltas.md.

## What the next task needs to know
Interfaces, assumptions, known weak spots.

## Human checks still needed
For hardware-tagged work: what to run, on what, and what to expect.

## Follow-ups proposed
New tasks or plan changes, for the lane owner to accept or reject.

## Record
The items in the task's Record column in PLAN.md.
```

## 16. Glossary

| Term | Meaning |
| --- | --- |
| Fact | One checkable statement about the laptop, such as a screw present or absent. Its kind comes from a fixed list |
| Step | One action in the procedure graph, with preconditions and effects over facts |
| Slot | A known position on a face where a screw or part belongs |
| Face | One side or layer of the laptop being worked on, such as the bottom cover or the interior |
| Gate | Turns per-frame scores into confirmed changes and contacts |
| Sequencer | Labels each confirmed change against the graph and updates the state |
| Trigger policy | Chooses which widgets to show, by class threshold and priority |
| W1 to W6 | Highlight, movement, tool, information, hazard, corrective |
| Tier A, B, C | Work done every frame, on changed frames, and on events (`PLAN.md` Section 1) |
| Mat | The printed marker sheet under the laptop |
| Enrolment | Recording and labelling one laptop so its parts can be detected |
| `pyref` | The Python reference pipeline that runs on recordings and is the oracle for the iOS port |
| ADR | A short decision record under `docs/decisions/` |
