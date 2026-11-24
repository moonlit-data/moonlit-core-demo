# Provenance

> Part of the moonlit-core showcase. The source it describes is in a private repository for now: *this repo*, and every file and test named on this page, mean that source.

Where this repository comes from, what it carries, what it leaves behind, and how the private data was kept out.

## Where it comes from

In November 2026 one operator ran his day-to-day projects through Claude Code. A long-lived orchestrator session took his requests and opened worker sessions to do the work; Devin ran some heavier jobs as an external agent, and Codex and Gemini reviewed designs. Around that fleet grew a private system: a card ledger on an append-only event log, acceptance checks on hand-back, a tool-call hook, a kill switch, a rules registry, a supervisor, and a few personal programs around them.

This repository is a clean rewrite of that system's core mechanisms. No file was copied. Claude Code sessions wrote it from a written spec, a few modules per session, under the decisions in [decisions.md](decisions.md); the operator set the operating model and made or approved each of those decisions. The private system was read only as a reference for behaviour and edge cases. Its content (cards, logs, documents, names) stayed where it was.

The private system is young, and so is the evidence. The orchestrator has run since 1 November 2026. The event log started on 18 November, dispatch-by-card on 21 November, the hook and the acceptance checks on 22 November. The numbers below cover days, not months.

## What it carries

| Mechanism | Here | Private system |
|---|---|---|
| Append-only event log, state as a fold, undo as an event, redaction | `events.py`, `state.py` | the same idea, with more event types |
| Five fixed card shapes, dedupe, requests, one inbox definition | `cards.py` | more card fields, a look-alike gate, linked cards |
| Dispatch from a card contract, brief from the contract | `dispatch.py` | also launches real sessions through a desktop shell |
| Hand-back with evidence and four acceptance checks | `handback.py`, `acceptance.py`, `transcripts.py` | eight checks, daily spot re-checks |
| Tool-call hook: off / probe / gate, four gate categories | `hooks.py` | nine orchestrator gate categories |
| Kill switch in front of money and account actions | `breaker.py` | the same rules, in front of real programs |
| Rules registry with computed status | `rules.py`, `rules.json` | 33 rules about the private system itself |
| The human's door: a one-time link and a key held by one browser tab, same-origin checks, click provenance | `web.py`, `board.html` | a larger dashboard |

## What it leaves behind

Left out on purpose, so that the repo stays small enough to read in an hour and change in an interview:

- **Launching real sessions.** The private system opens sessions through a desktop shell built on the Claude Agent SDK. Here `sim.py` plays the agents. It sends tool calls in the exact shape of a Claude Code hook event and writes transcripts in Claude Code's shape, so the gates and checks it exercises are the ones a real session would hit. `moon open --backend claude-code` registers a real session on its card and prints the command to start it, without running it; `moon hook` then gates that session, and notes where its transcript is so the hand-back checks read it (see `examples/claude-code/`).
- **The supervisor**: merging wake-ups so the orchestrator is not interrupted for every completion, patrolling stale cards, waking parked sessions after a usage-limit reset ([INC-009](incidents.md#inc-009), [DEC-008](decisions.md#dec-008)), reconciling cloud-agent runs against cards ([INC-003](incidents.md#inc-003)).
- **The look-alike card gate**, completion signals and linked cards ([INC-002](incidents.md#inc-002)). Only exact dedupe is here.
- **Probes that earn the right to close cards on their own** ([DEC-018](decisions.md#dec-018)). Here a probe runs once, at hand-back.
- **The single writer for project documents**, with fingerprints that notice edits made around it, and the size limits it enforces ([DEC-015](decisions.md#dec-015)); the counter that asks to delete prose once code enforces a rule ([DEC-004](decisions.md#dec-004)).
- **Four acceptance checks tied to the private system's own documents** (rule acknowledgement, stop-at and next-step lines, where a rule is written, whether a record was written), and the daily spot re-checks.
- **The over-caution metric** ([INC-010](incidents.md#inc-010)) and the usage and memory figures in the pre-flight line ([DEC-009](decisions.md#dec-009)).
- **The cleanup job** ([INC-004](incidents.md#inc-004)) and everything personal.
- **A model anywhere in the guardrail path.** On purpose: every check and gate here is plain code.

## How the private data was kept out

- **Mechanisms, not content.** Each module was written from a description of behaviour, in English. Decisions are paraphrased; nothing is quoted from the operator.
- **Invented data.** The demo, the tests and the evals use a fictional company, Kestrel Outfitters, with projects `storefront`, `billing`, `support-desk` and `ops`. People appear only as roles: the operator, the orchestrator, a worker.
- **Ids instead of records.** Incidents and decisions are referred to as `INC-` and `DEC-` ids. No production card numbers, session ids, paths or names of other projects appear.
- **A test that fails on leaks.** `tests/test_repo_hygiene.py` scans every tracked file for CJK characters, drive-letter and home-folder paths, and email addresses other than noreply ones.
- **History that starts here.** The git history begins with an empty folder and records the work as it happened, under a noreply identity.

## Production status

Stated plainly, because the repo exercises more than production has. In the private system the acceptance checks have only ever run in warn mode: 104 runs, none blocked. The hook has only ever run in `probe` mode, so no gate has blocked a call there. The probe ladder of [DEC-018](decisions.md#dec-018) has never fired. In this repo every gate and check is exercised by the simulator, the tests and the eval suites, which is not the same as weeks against a live model.

<a id="numbers"></a>
## Numbers

From the private system, counted read-only on 23 November 2026. Each line gives what was counted and the time window.

1. **Sessions dispatched:** 158 agent sessions were dispatched in 52 hours (Nov 21 06:05 → Nov 23 09:58), with a peak of 92 in one day. 152 of them were opened straight from a card; the other 6 were registered against a card afterwards or opened another way. Each one records its model and effort.
2. **Events and cards:** 2,726 events and 619 cards in 4.5 days, in an append-only event log (Nov 18 21:34 → Nov 23 10:07). 414 cards are closed. The median time from filing to done is 2.2 h across all cards and 0.85 h for task cards.
3. **Hook layer:** 25,523 hook calls across 170 sessions (Nov 22 18:54 → Nov 23 10:07). The hook's in-process time on PreToolUse is p50 1.4 ms and p95 2.8 ms, not counting interpreter start-up. 0 hook errors were logged. The hook has only ever run in observe-only probe mode, so nothing was blocked.
4. **Acceptance checks on hand-back:** 104 check runs, 29 of them with warnings, 54 warnings in total (Nov 22 22:57 → Nov 23 10:01). Warnings from the four checks in this repo: `evidence_shape` 1, `covers` 7, `do_not_touch` 5, `probe` 0. The other 41 came from checks that are not part of this repo. 0 hand-backs were blocked, because every check ran in warn mode. 14 daily spot re-checks of closed cards found 0 problems.
5. **Hand-backs:** 336 "done" hand-back attempts through the card CLI, 5 refused (Nov 18 → Nov 23 10:07). 2 of the 5 were a session trying to close a rule card from the CLI; only the operator closes that card, with a click on the web page, so both were refused. Of the other 3, 1 was refused because project documents had been edited around the single writer, and 2 were usage errors.
6. **CLI refusals:** the card CLI refused 65 of 2,183 calls, and 8 of those were approvals only the orchestrator may make (Nov 18 → Nov 23 10:07).
7. **Rules registry:** 33 operator rules are tracked: 20 green (enforced by code and a test), 9 yellow (waiting on work), 1 red (not enforced), 3 grey (text only). As of Nov 23 09:56.

**How these were counted.** Small scripts read the private system's event log, the hook's probe log, the card CLI's usage log and the local session transcripts, and wrote nothing back; event counts come from replaying the log with the same fold the system itself uses. Every figure was recomputed on the date above from those raw files, not copied from notes, and figures that could not be recomputed were left out of this list. They will be recomputed once more before publication, so they may move slightly.

### Other figures in the incidents and decisions, recomputed the same way

- **152 of 158 sessions opened from a card** (line 1, [DEC-002](decisions.md#dec-002)): the other 6 are 3 registered against a card afterwards, 1 reattached, 1 opened from a free-form dispatch file and 1 opened by the operator himself.
- **101 human clicks** ([DEC-006](decisions.md#dec-006)): every click event from Nov 18 21:42 to Nov 22 23:47; one of them was later taken back.
- **The breaker asked 3 times: 2 allowed, 1 denied** ([DEC-012](decisions.md#dec-012)): its audit file, all during its acceptance test on Nov 22; the denial was a test host not on the allow-list.
- **19 warnings for rules not yet acknowledged** ([DEC-016](decisions.md#dec-016)): part of the 41 warnings from checks not in this repo (line 4).
- **The similar-card gate refused 2 filings; the sweep found 0 pairs** ([INC-002](incidents.md#inc-002)): refusals of command-line filings in the card CLI's usage log, and the sweep's own state file. A program that is refused in-process leaves no log, so it is not counted.
- **101 external-agent sessions reconciled** ([INC-003](incidents.md#inc-003)): the reconciliation registry holds 101 entries, 92 linked to a card and 9 marked as needing no report; 11 back-filled report cards cover 47 of them. The hourly reconciliation has run 6 times and found 0 unclaimed outputs.
- **Six gate smoke runs, the last 46 of 46** ([INC-010](incidents.md#inc-010)): the smoke test's own log.

<a id="numbers-from-write-ups"></a>
### From the incident write-ups, not recomputed

These figures appear in the incidents, the decisions and the README. Each comes from the write-up made when the incident happened, or from the private notes of that day, not from a script rerun over the raw logs on 23 November, so the paragraph above does not cover them. Where a figure is a difference between two times in a write-up, it says so. Every other figure in [incidents.md](incidents.md) and [decisions.md](decisions.md) that is in neither list (clock times, self-test item counts, counts of cards within one incident) is of this kind too.

- **About 117 pieces of work, 10 tracked end to end, 28 dispatched with no card** ([INC-001](incidents.md#inc-001), [DEC-002](decisions.md#dec-002)): the count reported by the read-only review of two days; two notes give the same figures.
- **The orchestrator has run since 1 November 2026**: the operator's notes from the day it was set up. The event log, which could confirm it, starts on 18 November.
- **39 occurrences on 8 lines** ([INC-006](incidents.md#inc-006)): the count recorded when the log was redacted.
- **About ten minutes** unreachable ([INC-008](incidents.md#inc-008)): the difference between two message times in the write-up.
- **Over three hours** in the queue ([INC-002](incidents.md#inc-002)): the difference between the time the first card was closed and the time a worker closed the second (3 h 12 min).
- **Five live calls** ([INC-005](incidents.md#inc-005)): the write-up's count of real calls, all of which the API rejected.
- **15 GB** ([INC-004](incidents.md#inc-004)): the memory the process had reached when it was killed, as the write-up records it.
- **WebSearch 1.4 → 0.8 and joint searches 1.3 → 0.8 per 100 calls, almost 40 percent** ([INC-010](incidents.md#inc-010), [DEC-016](decisions.md#dec-016)): the first run of the over-caution metric, as reported on 22 November.
- **830 existing rules baselined, ten reviewed a week** ([DEC-016](decisions.md#dec-016)): the figures recorded in the private notes when the rule was set up on 22 November.
- **14,992 to 7,682 characters** ([DEC-015](decisions.md#dec-015)): the doc's size before and after it was tidied, as the private notes of 23 November record it.
- **53 stored reports** ([DEC-020](decisions.md#dec-020)): the files in the folder that keeps full-length reports, counted by listing it on 23 November, not by the scripts above.
