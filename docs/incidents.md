# Incidents

> Part of the moonlit-core showcase. The source it describes is in a private repository for now: *this repo*, and every file and test named on this page, mean that source.

These are blameless write-ups of twelve incidents from the private deployment this repository was distilled from: one human operator, one orchestrator session that only dispatches, a fleet of Claude Code worker sessions, Devin as an external agent, and Codex and Gemini as reviewer models. People appear only as roles, and no one is at fault; each incident is a mechanism that was missing. Numbers come from the private deployment's own logs on the dates given (all dates are 2026). Each Fix and each How it stays fixed is split in two: what the private system actually built, and what this repo carries. Where the repo carries nothing, it says "not ported".

Production status, stated once and plainly. In the private system the acceptance checks ran 104 times, all in warn mode, and blocked nothing. The hook gateway has only ever run in `probe` mode, so the orchestrator gate has never blocked a call. The verification-probe ladder has raised 0 flags and made 0 promotions. External-agent reconciliation has run 6 times and found 0 unclaimed outputs. In this repo the gates and checks are exercised by the simulator, the tests and the eval suites.

| | Incident | In this repo |
|---|---|---|
| [INC-001](#inc-001) | Work dispatched without a card | `dispatch.py` |
| [INC-002](#inc-002) | Two cards for one job, one left open for hours | `cards.py` (in part) |
| [INC-003](#inc-003) | Cloud-agent outputs nobody collected | not ported |
| [INC-004](#inc-004) | Cleanup job read archives into memory (15 GB) | not ported |
| [INC-005](#inc-005) | A test called the real model API | `sim.py`, `dispatch.py` |
| [INC-006](#inc-006) | Secrets found in the event log | `events.py`, `hooks.py` |
| [INC-007](#inc-007) | A human-stated rule written only as prose | `rules.py` |
| [INC-008](#inc-008) | Orchestrator stuck busy behind background subagents | `hooks.py` |
| [INC-009](#inc-009) | Usage limit hit mid-task | not ported |
| [INC-010](#inc-010) | Agents turned over-cautious after a rule change | `hooks.py` (in part) |
| [INC-011](#inc-011) | The human queue flooded with reports | `cards.py` |
| [INC-012](#inc-012) | A test orchestrator shared the real orchestrator's name | `config.py` |

<a id="inc-001"></a>
## INC-001 — Work dispatched without a card

Date: 2026-11-21

**What happened** A read-only review of two days asked how many pieces of work went all the way from request, to structured card, to agent, to result. Worker sessions were being briefed from free-form dispatch text files; the card was only bookkeeping next to the real brief.

**Impact** Of about 117 pieces of work, 10 were tracked end to end. 28 were dispatched with no card at all, most of them a batch handed to Devin. Only 9 worker sessions had ever opened their own card.

**Root cause** Dispatch and tracking were separate paths, and nothing forced the brief to come from the card.

**Fix**
- **In the private system** The card became the dispatch: it carries goal, done-when, read-first, do-not-touch, model and effort, the opening brief is generated from it, and a session without a card number is refused with exit code 5.
- **In this repo** `dispatch.py`: `open_session` refuses without an open task card that has a contract (at least a goal and a done-when), with exit code 5 (`DispatchRefused`), and `render_brief` builds the worker's opening prompt from the contract.

**How it stays fixed**
- **In the private system** A dispatch self-test, extended after five rounds of Codex review.
- **In this repo** `tests/test_authorization.py` (no card, or no contract: exit 5) and `tests/test_dispatch.py` (the brief is generated from the contract); rule R-001 in `rules.json`; demo act 1.

**What I would do differently** Measure end-to-end coverage in the first week, not in a retrospective.

<a id="inc-002"></a>
## INC-002 — Two cards for one job, one left open for hours

Date: 2026-11-22

**What happened** A companion program files an action card and closes it itself when the operator clicks. That card reopened at 19:59; at 20:01 the orchestrator filed a second card for the same step. Its prose said so, but nothing machine-readable linked them. At 20:02 the operator clicked; the program closed its card and the second stayed open.

**Impact** The duplicate sat in the operator's queue for over three hours, until a worker closed it at 23:14. The step had been filed 7 or 8 times before.

**Root cause** A new card did not have to say how its completion would be detected, and filing never looked for similar open cards.

**Fix**
- **In the private system** New action and decision cards must name a check, a card they follow, or why no check is possible; a similar-card gate refuses look-alikes; linked cards close together; a pairwise sweep backs it up. The gate has since refused 2 filings; the sweep has found 0 pairs.
- **In this repo** `cards.py`: `file_card` looks for an open card with the same type, project and normalized title (`find_twin`) and updates it instead of filing a second one. It does this under the log's lock, so two sessions filing at once still make one card. Such an update may change the card's contract only if it comes from the orchestrator, the person at a terminal or the program that filed the card, and a new card from a worker is a request. Titles are compared after punctuation becomes a space, so `#12` and `1/2` stay different. The similar-card gate, the completion signal and follow links are not ported.

**How it stays fixed**
- **In the private system** A 57-item self-test, plus a replay of the real event pair.
- **In this repo** `tests/test_idempotency.py`, including eight sessions filing the same job at once; rule R-002.

**What I would do differently** Require a completion signal from the first card type.

<a id="inc-003"></a>
## INC-003 — Cloud-agent outputs nobody collected

Date: 2026-11-22 (private fix 2026-11-23)

**What happened** Devin ran heavier jobs as an external agent. The operator noticed that several Devin jobs from the past few days had never reached the card system, so nothing told him they were done.

**Impact** An inventory went through every Devin session. Today all 101 are matched: 92 to a card and 9 marked as needing no report. 11 report cards had to be back-filled, and they cover 47 sessions. The audit note written at the time recorded 13 unreported outputs; that figure cannot be recomputed from state today.

**Root cause** "Devin finished, so a card exists" depended on the dispatching worker remembering. When that session closed, the link was gone.

**Fix**
- **In the private system** External-agent reconciliation: every hour, a session finished for 2 hours with no card collecting it files an orchestrator task. It has run 6 times and found 0 unclaimed outputs.
- **In this repo** Not ported (it belongs to the supervisor, which this repo leaves out).

**How it stays fixed**
- **In the private system** A 13-item self-test, and a first real run that matched 101 of 101 sessions.
- **In this repo** Not ported.

**What I would do differently** Register the external session against its card at dispatch, so reconciliation is a join rather than a hunt.

<a id="inc-004"></a>
## INC-004 — Cleanup job read archives into memory (15 GB)

Date: 2026-11-22

Private system only: the cleanup job is not ported.

**What happened** A new nightly job moves archived items older than 30 days to the recycle bin, after checking that no shortcut, process or doc still points at them. Its shortcut check used a filter combination that Windows PowerShell 5.1 silently ignores, so it read every file in the folder, including a large zip archive.

**Impact** One process grew to 15 GB of memory before the orchestrator killed it. Nothing was deleted; zero items were due that day.

**Root cause** An unbounded scan with a silently failing filter and no resource cap, inside the safety check meant to make the job harmless.

**Fix**
- **In the private system** Filter by suffix in code, cap the job at 1 GB, and abort the whole round with no deletions if any check fails.
- **In this repo** Not ported. The nearest convention: all paths come from a `Workspace`. The hand-back checks also read a registered Claude Code session's own transcript where Claude Code keeps it (usually under the home directory), at the path the hook noted.

**How it stays fixed**
- **In the private system** A regression test pins the filter, and the scheduler enforces the 1 GB cap.
- **In this repo** Not ported.

**What I would do differently** Cap memory and time on every new scan from its first run.

<a id="inc-005"></a>
## INC-005 — A test called the real model API

Date: 2026-11-19

**What happened** A shared Claude-calling wrapper gained a reviewer step: a stronger model checks a cheaper model's output before it is sent. Its tests were meant to use a fake Claude executable, but a few invoked the real CLI.

**Impact** Five real calls went out. All five were rejected with HTTP 400 because the output schema used non-English keys, so no quota was spent. A debug script also filed one stray card.

**Root cause** Tests reached the real executable by default; the fake was opt-in per test.

**Fix**
- **In the private system** Tests point the executable variable at a file that does not exist, so a leak fails loudly. The 400 exposed a real bug too: schema keys are now English.
- **In this repo** There is no path from the tests, the demo or the evals to a real model. `sim.py` stands in for agents, and only when asked: `moon open N` registers a session and starts nothing, and `--play` puts a simulated worker in it. `moon open --backend claude-code` registers the session and prints the `claude` command; it never runs it (`--dry-run` prints without registering).

**How it stays fixed**
- **In the private system** The wrapper's 50-item self-test, run against the fake executable.
- **In this repo** `tests/test_dispatch.py` (a Claude Code session is registered and its command printed, never run) and `tests/test_sim.py`; rule R-018.

**What I would do differently** Set the safe default once for the whole suite instead of per test.

<a id="inc-006"></a>
## INC-006 — Secrets found in the event log

Date: 2026-11-23

**What happened** Two remote-access passphrases set by the operator had ended up in plain text in card content. The operator asked for them to be erased. A sweep showed they had flowed into the append-only event log and everything derived from it.

**Impact** 39 occurrences on 8 lines of the event log, from 4 cards, 1 handoff migration event and 3 memory-write entries. Outside the log: a rule baseline, a handoff export (5 places), two library reports, one test file, 34 event-log backups and an undo backup.

**Root cause** Free text was stored as-is, and an append-only log and its derived files multiply every secret.

**Fix**
- **In the private system** Under the data lock the values became a placeholder and state was rebuilt by replay; log and state then held zero copies. Copies outside the log went to the orchestrator as a list.
- **In this repo** `events.py`: `EventLog.redact` (CLI `moon redact`) replaces every match in the log's free-text values (evidence, notes, titles, goals and the like) under the log's lock. It leaves every breaker, click, refusal, session, hook and acceptance record whole, so a redaction cannot reopen a switch or pay a key twice, and it refuses a pattern that matches the empty string. Only the orchestrator or the person at a terminal may run it. It keeps the old log as a backup inside the workspace, appends `log.redacted` with the count and who ran it, and says plainly that the backup still holds the secret. State needs no separate rebuild: it is folded from the log on every read. `hooks.py` in `probe` mode records key names, never values.

**How it stays fixed**
- **In the private system** A replay after the rewrite showed zero copies in log and state.
- **In this repo** `tests/test_replay.py` (redact scrubs the log and the state, keeps a backup, is not for workers and never touches structure), rule R-024, and `tests/test_hooks.py` (probe records names and timings, never values).

**What I would do differently** Scan for secrets at write time, and include backups in redaction.

<a id="inc-007"></a>
## INC-007 — A human-stated rule written only as prose

Date: 2026-11-22

**What happened** The operator gave the orchestrator a new rule: never swap in a fresh orchestrator while he is using the current one. It went into prose only. He then asked whether such a slip would even be recorded. It would not; nothing in code noticed.

**Impact** Earlier that day a rules registry had been built for 17 audited rules; its first pass found one with no enforcer. The new rule was in none of it.

**Root cause** Capturing a rule depended on the session that heard it, and prose was the default home.

**Fix**
- **In the private system** A new rule is registered as prose-only with a task to move it into code; if the task closes without an enforcer, the rule turns red. A scan of orchestrator docs is the backstop. The registry now holds 33 rules: 20 green, 9 yellow, 1 red, 3 grey.
- **In this repo** `rules.py`: each rule in `rules.json` names its enforcing function and its test, and its colour is computed from the code. A rule added at runtime (`moon rules add`) never edits `rules.json`: it is recorded grey, files a task to move it into code, and waits for the human's acknowledgement on the web board, whoever typed the command.

**How it stays fixed**
- **In the private system** Registry and new-rule self-tests (93 and 28 items).
- **In this repo** `tests/test_rules.py`; `moon rules check --run` on a fresh clone.

**What I would do differently** The scanner first filed two progress sentences as rules; tune it on labeled cases first.

<a id="inc-008"></a>
## INC-008 — Orchestrator stuck busy behind background subagents

Date: 2026-11-22

**What happened** The orchestrator handed a slow lookup to a background subagent; its own turn ended at 22:47:43. In the desktop chat shell, every subagent message set the conversation's busy flag, but a subagent never sends the result message that clears it. The operator pressed stop at 22:52:04. The runtime interrupted only the subagent, so busy stayed true and his messages queued.

**Impact** The orchestrator was unreachable for about ten minutes, until another session's message started and finished a turn at 22:57. A fresh orchestrator replaced it.

**Root cause** Busy state was inferred from any traffic, and the orchestrator was doing work itself through subagents.

**Fix**
- **In the private system** Subagent messages no longer set busy, and stop clears busy after 8 seconds when no turn is running. Policy: the orchestrator opens sessions, never subagents (DEC-010). The orchestrator gate files subagent spawns as their own category, but only records them: it has never blocked.
- **In this repo** `hooks.py`: the `orchestrator_does_work` gate denies edits, shell writes and subagent spawns by the orchestrator, while it may still read, file cards and open sessions. It denies only in `gate` mode, once the operator sets its category to `block`; until then it records what it would have denied.

**How it stays fixed**
- **In the private system** A 10-item shell test for the stop path.
- **In this repo** `tests/test_hooks.py` and the `orchestrator_does_work` cases in `evals/cases/hooks.jsonl`; rule R-009; demo act 4.

**What I would do differently** Treat an unreachable orchestrator as a first-class failure mode.

<a id="inc-009"></a>
## INC-009 — Usage limit hit mid-task

Date: 2026-10-30 and 2026-11-21 (private fix 2026-11-23)

**What happened** A daily scheduled job hit usage limits on two of three days. Each time it retried 3 times within 3 minutes, gave up, and said nothing; its status still said the result would come later that day. Weeks later the orchestrator paused two worker sessions mid-task until a 14:50 reset and resumed them by hand, and a Codex review round then hit the reviewer's own limit: a 1,440-minute wait.

**Impact** The job delivered on 1 of 3 scheduled days. Stalled sessions stayed stalled until a person noticed.

**Root cause** A usage limit was treated as a failure, not as a scheduled pause.

**Fix**
- **In the private system** The scheduled job re-checks every 30 minutes on a limit error (a failed call costs nothing) and shows the reason. Quota wake: at reset plus 3 minutes the orchestrator and each parked session are woken exactly once per reset; a large, cold session gets a handover and a successor instead (DEC-008).
- **In this repo** Not ported (supervisor).

**How it stays fixed**
- **In the private system** A 30-item self-test over fake usage data, and a dry-run rehearsal.
- **In this repo** Not ported.

**What I would do differently** Plan for usage limits from day one.

<a id="inc-010"></a>
## INC-010 — Agents turned over-cautious after a rule change

Date: 2026-11-22

**What happened** A worker building the orchestrator gate added an over-caution metric: web searches, fetches, browser use and joint searches per 100 tool calls, before and after a cut-off. The first run cut at the evening of 11-20, when a batch of rule changes landed.

**Impact** Over the next two days WebSearch fell from 1.4 to 0.8 per 100 calls and joint searches from 1.3 to 0.8, almost 40 percent. No gate was blocking anything, so the cause was the prose rules or the work mix; the data could not tell, so it was reported, not acted on.

**Root cause** Restrictive rules have a cost nobody measures, and agents comply by doing less.

**Fix**
- **In the private system** A smoke run checks that ordinary worker actions are never blocked, and a 30% drop in the metric files a card.
- **In this repo** `hooks.py`: each gate category has its own mode (`observe` or `block`). `orchestrator_does_work`, the one that judges how an agent works, ships in `observe` and is promoted by the operator. Once promoted, if it blocks more than 20% of the day's distinct calls (at least 30 calls and 5 blocks; a call is its session plus the paths it writes, so a retry is not a new call), it demotes itself to `observe` for the rest of the day and files a task. The three gates that enforce what the card or moon itself says (`do_not_touch`, `protected_paths`, `human_door`) ship in `block` in `gate` mode and never demote: a block there is the rule working, not over-caution (DEC-005). The over-caution metric is not ported.

**How it stays fixed**
- **In the private system** Six smoke runs; the last passed 46 of 46.
- **In this repo** The smoke test and the demotion test in `tests/test_hooks.py`, and the false-block rate on `evals/cases/hooks.jsonl`; rule R-013.

**What I would do differently** Record the behavioural baseline before changing rules.

<a id="inc-011"></a>
## INC-011 — The human queue flooded with reports

Date: 2026-11-22

**What happened** The operator's queue was meant for what only he can do. On 11-21 a new patrol job filed 25 cards on its first run, and the operator remarked the list looked long. The next day he pointed out that reports did not belong in his queue at all.

**Impact** 16 patrol cards were closed under a tightened rule: file only what agents can push forward now. After the redefinition the queue held 19 cards and 12 completion notices moved to the library. Four reports were waiting on a choice with no decision card; one re-asked a choice already made twice.

**Root cause** There was no single definition of the queue, and filing had no notion of who must act.

**Fix**
- **In the private system** One function defines the queue as open decision, action and dated cards; the web page, desktop app and CLI all call it. Completions reach the orchestrator through merged wake-ups in a 10-minute digest.
- **In this repo** `cards.py`: `waiting_on_human()` is the only definition: open action, decision and dated cards that are not waiting on an outside party. `moon inbox`, the board's "Waiting on you" column in `web.py` and the demo all call it. Merged wake-ups are not ported.

**How it stays fixed**
- **In the private system** A 10-item inbox self-test and web-page checks.
- **In this repo** `tests/test_cards.py`, `tests/test_web.py` (the board's column is the same list as `moon inbox`) and `tests/test_authorization.py` (the definition exists exactly once); rule R-006.

**What I would do differently** Define the queue by who must act before building anything that fills it.

<a id="inc-012"></a>
## INC-012 — A test orchestrator shared the real orchestrator's name

Date: 2026-11-19

**What happened** While testing an orchestrator-swap feature on a test copy of the desktop chat shell, a worker started a test orchestrator under the same name as the real one. Session names were shared across the whole machine, so when a nightly report session sent its report to the orchestrator, the test session received it too.

**Impact** The test orchestrator replied with a few lines; it changed no files and dispatched nothing, and the real orchestrator picked up the report. The test shell was stopped on the spot and its orchestrators closed.

**Root cause** Test and live sessions were addressed by a human-readable name in one shared namespace.

**Fix**
- **In the private system** Test orchestrators always carry a distinct name, and their opening prompt only lets them acknowledge.
- **In this repo** `config.py`: every path comes from a `Workspace`, resolved from an explicit argument, then `MOONLIT_HOME`, then the nearest folder holding `.moonlit/`. Session ids are per workspace, and the Claude Code `--session-id` that `moon open` prints is derived from the workspace's full path, so two workspaces that are both called `work` do not share one. A test workspace and a live one share an address only if someone points them at the same folder.

**How it stays fixed**
- **In the private system** A 25-item shell test for the swap path.
- **In this repo** No single test pins it; every test builds its own workspace under `tmp_path` (`tests/conftest.py`).

**What I would do differently** Address sessions by workspace and id, never by display name.
