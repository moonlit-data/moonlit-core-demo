# Architecture decisions

> Part of the moonlit-core showcase. The source it describes is in a private repository for now: *this repo*, and every file and test named on this page, mean that source.

Twenty decisions behind moonlit-core, carried over from the private deployment. ◆ the operator proposed it, or picked it among options. ◇ someone else (an external review agent or a worker session) proposed it and the operator approved it, with or without his own edits. No mark: the implementing session decided. Production figures come from the private system; there, the acceptance checks have only ever warned, the hook has only run in probe mode, and no gate has blocked anything yet. Where each figure comes from is listed in [provenance.md](provenance.md#numbers). All dates are 2026.

| | Decision | Mark | In this repo |
|---|---|---|---|
| [DEC-001](#dec-001) | One orchestrator that only dispatches | ◆ | `hooks.py`, `dispatch.py` |
| [DEC-002](#dec-002) | Dispatch is a card | ◇ | `dispatch.py`, `cards.py` |
| [DEC-003](#dec-003) | Rules belong in code | ◆ | `rules.py` |
| [DEC-004](#dec-004) | Delete the prose once code enforces it | ◆ | not ported |
| [DEC-005](#dec-005) | Guardrails ship in warn mode; the human promotes them | ◇ ◆ | `acceptance.py`, `board.html` |
| [DEC-006](#dec-006) | Human-only cards accept only a human web click | ◇ | `web.py`, `cards.py` |
| [DEC-007](#dec-007) | The inbox holds only decisions, actions, dated items | ◆ | `cards.py` |
| [DEC-008](#dec-008) | On a usage limit, wait for the reset and resume | ◆ | not ported |
| [DEC-009](#dec-009) | No hard cap on sessions; show load instead | ◆ | `dispatch.py` |
| [DEC-010](#dec-010) | The orchestrator opens sessions, never subagents | ◆ | `hooks.py` |
| [DEC-011](#dec-011) | Hooks installed once, switched by a file | ◇ | `hooks.py` (in part) |
| [DEC-012](#dec-012) | A kill switch in front of money and account actions | ◇ | `breaker.py` |
| [DEC-013](#dec-013) | The premium model is selectable only by the human | ◆ | `cards.py`, `dispatch.py` |
| [DEC-014](#dec-014) | Authorisation tiers | ◆ | in part |
| [DEC-015](#dec-015) | Docs get shorter: size limits in code | ◆ | not ported |
| [DEC-016](#dec-016) | Human approval only for decisions made once that keep working | ◆ | `rules.py` |
| [DEC-017](#dec-017) | Hooks fail open, the breaker fails closed | ◇ | `hooks.py`, `breaker.py` |
| [DEC-018](#dec-018) | Probes earn automation: 3 right, 0 wrong | | not ported |
| [DEC-019](#dec-019) | Effort is the orchestrator's judgment | ◆ | `dispatch.py` |
| [DEC-020](#dec-020) | Over-long reports are stored, not refused | ◇ | not ported |

<a id="dec-001"></a>
## DEC-001 — One orchestrator that only dispatches ◆

**Context.** The operator talked to every worker session directly and kept every thread in his own head.
**Options.** Keep talking to each session; or route everything through one session that plans and dispatches but does no work.
**Decision.** 2026-11-01: the operator talks only to the orchestrator, which opens worker sessions for the work. The do-no-work half was later written as a gate instead of a prompt.
**Trade-off.** One extra hop, in exchange for one place that knows what is running.
**In the private system.** An orchestrator gate with nine categories, observe-only so far; it has never blocked a call.
**In this repo.** `hooks.py` (`orchestrator_does_work` gate), `dispatch.py` (a worker opens no session); and in `cards.py` every card a worker files goes to the orchestrator as a request, so nothing reaches the human straight from a worker, and a worker cannot close or shelve the human's cards. Roles are self-declared labels, so this routes cooperative agents; it does not stop one that lies about its role. Tests: `test_hooks.py`, `test_authorization.py`.

<a id="dec-002"></a>
## DEC-002 — Dispatch is a card ◇

**Context.** A read-only review of two days found about 117 pieces of work: 10 went through the system end to end, 28 were dispatched with no card (INC-001).
**Options.** Better prompts telling sessions to read the card; or make the card the only way to start a session.
**Decision.** 2026-11-21: a worker session proposed four changes and the operator approved all four; the next day he restated it himself: the card carries everything a worker needs. Dispatch without an open task card is refused.
**Trade-off.** Ad-hoc work costs a card first. The private system keeps a logged override; this repo has none.
**In the private system.** Of 158 registered sessions, 152 were opened from a card.
**In this repo.** `dispatch.py` (`open_session`, exit code 5), `cards.py` (`MINIMUM_CONTRACT`: a goal and a done-when). The contract belongs to whoever dispatched the card: it changes only by the orchestrator, the person at a terminal, or the program that filed the card (`guard`, rule `contract`). A worker cannot route around that with a card of its own, which becomes a request, or by shelving its card, or by handing back any card but its own session's (rule `own_card`). `moon open` registers the session and starts nothing. Tests: `test_authorization.py`, `test_dispatch.py`.

<a id="dec-003"></a>
## DEC-003 — Rules belong in code ◆

**Context.** Rules lived in long prose that every session had to read, remember and interpret. The orchestrator repeated mistakes it had been warned about.
**Options.** Rewrite the prose more forcefully; or register each rule with the function that enforces it and the test that pins it.
**Decision.** 2026-11-22: the operator made it explicit that everything agreed in a large design review must be managed by code, and that a rule sessions ignore needs code or a mechanism, not hope. Rule status is computed.
**Trade-off.** Some rules cannot become code; they stay grey with a written reason.
**In the private system.** A registry of 33 rules: 20 green, 9 waiting on a card, 1 red, 3 prose-only.
**In this repo.** `rules.py` and the repo's own `rules.json`. Test: `test_rules.py`.

<a id="dec-004"></a>
## DEC-004 — Delete the prose once code enforces it ◆

**Context.** A rule enforced in code leaves its prose copy behind to drift; sessions follow whichever they read last.
**Options.** Keep both as documentation; or delete the prose once the code copy is proven.
**Decision.** 2026-11-19: after a session noted that enforced rules could leave the prose, the operator made deletion mandatory, to stop the two copies disagreeing. On 2026-11-22 he approved a review's tightening: delete only when a test pins the rule and a health check agrees, keep a one-line pointer where needed daily, and ask him before deleting from top-level instructions.
**Trade-off.** Less to read, at the price of trusting the registry.
**In the private system.** A counter files a deletion request once a rule has stayed green long enough.
**In this repo.** Not ported.

<a id="dec-005"></a>
## DEC-005 — Guardrails ship in warn mode; the human promotes them ◇ ◆

**Context.** New hand-back checks could block honest work if they misfire, and an agent recommending settings for its own guardrails would be grading itself.
**Options.** Ship blocking; warn forever; or warn for a fixed period, then ask the human per check.
**Decision.** 2026-11-22: the warn-only week and daily spot checks came from an external review; the operator approved them and asked that checks reuse what the card already states. The no-recommendation half is his: the orchestrator must not make these calls, so each promotion card offers block, keep warning or off, with identical buttons.
**Trade-off.** A week of weaker enforcement; promotion is slow but owned.
**In the private system.** 104 check runs, all in warn mode: 29 raised warnings, 0 blocked.
**In this repo.** `acceptance.py`: every check starts in `warn`; after seven days `moon checks review` files one human-only decision per check that warned, with no recommendation; the human's web answer becomes the check's mode. `board.html` renders a decision without a recommendation with every option in the same button style, and `check_shape` refuses a recommendation on any card whose answer becomes a setting (one with a `subject`). Warn first covers the acceptance checks and the `orchestrator_does_work` gate. The other three gates (`do_not_touch`, `protected_paths`, `human_door`) enforce what the card or moon itself already says, so once the hook is in `gate` mode they ship in `block` and never demote. Daily spot checks are not ported. Tests: `test_evidence.py`, `test_hooks.py`.

<a id="dec-006"></a>
## DEC-006 — Human-only cards accept only a human web click ◇

**Context.** A session can run the same CLI command the human would, so a CLI close proves nothing about who decided.
**Options.** Trust self-reported identity; or accept a click only when it carries proof that it came through the human's web door.
**Decision.** 2026-11-22: the operator said rule cards are his to answer and a program clicking for him goes too far. The provenance mechanism came from an external review and he approved it: a web click is checked for where it came from; the same answer through the CLI is refused and logged. He left the wider identity question open.
**Trade-off.** Rule changes wait for him to open the board.
**In the private system.** 101 human clicks; two hand-backs refused because only a web click may close a rule card.
**In this repo.** No role on the command line is the human, not even `human`: everything only the human may do needs the web door. `web.py` is the only place that builds the human-via-web actor, for a request that carries the key from the one-time link `moon serve` prints in the human's terminal, this server's Host and an Origin of this server. The key is written once into the page that opened the link and kept in that page's `sessionStorage`; there is no cookie, and nothing on disk opens the board. `cards.py` (`guard`) refuses and logs every other door. Nobody on the command line can change a human-only card's fields or status once it is filed, or undo the human's click: changing his mind takes a new card. `moon redact` can still scrub a secret out of the card's text, never its options' names or the click. The program that filed a human-only card may retire it, for good: `breaker off` retires the open switch-on cards, so switching on again takes a new card. Every decision card is human-only, and so are the cards that promote a check, switch the breaker on, acknowledge a rule or allow the premium model; an answer becomes a setting only on a card filed as the program that asked. With the hook gate on, `human_door` denies the usual ways an agent session would open the board or knock on it, but the key is what keeps the door shut: a session that starts its own server from a script, or with the gate off, gets a board of its own. That is why this is a guardrail for cooperative agents and not a sandbox. Tests: `test_web.py`, `test_authorization.py`, `test_hooks.py`.

<a id="dec-007"></a>
## DEC-007 — The inbox holds only decisions, actions, dated items ◆

**Context.** Reports and completion notices piled into the human queue, burying the few items that needed him (INC-011).
**Options.** Filter separately in each view; or define the queue once.
**Decision.** 2026-11-22: the operator said reports do not belong in the waiting-on-you list; it is for what he must decide or do. The definition became one function used by every view. Reports and done notices go to the library.
**Trade-off.** A report quietly waiting for a choice must become a decision card or go unseen; the first pass found four.
**In the private system.** One function shared by the web board, the desktop app and the CLI.
**In this repo.** `cards.py` (`waiting_on_human`), called by `moon inbox` and the board. Tests: `test_cards.py`, `test_web.py`.

<a id="dec-008"></a>
## DEC-008 — On a usage limit, wait for the reset and resume ◆

**Context.** Sessions stopped at usage limits mid-task, and nothing woke them after the reset (INC-009).
**Options.** Restart by hand; or park and wake automatically.
**Decision.** 2026-11-23: the operator asked that, when quota runs out, the system wait for the reset and then wake everything to carry on. Wakes come at reset plus three minutes, once per reset. Not waking large, cold sessions was also his demand; they hand over to a successor instead.
**Trade-off.** Idle hours are accepted: work waits for the reset.
**In the private system.** A supervisor job every two minutes, tested with a dry-run rehearsal.
**In this repo.** Not ported.

<a id="dec-009"></a>
## DEC-009 — No hard cap on sessions; show load instead ◆

**Context.** The orchestrator could open at most five worker sessions at once. The cap guarded quota but blocked dispatch whenever the operator wanted more in parallel.
**Options.** Keep the cap; raise it; or remove it and show load.
**Decision.** 2026-11-22: the operator asked to drop the cap. Before opening a session, the orchestrator sees what is open and proceeds.
**Trade-off.** Overload becomes possible; it is shown rather than prevented, and the human stays the throttle.
**In the private system.** The pre-flight line shows open sessions, five-hour and weekly usage, and free memory.
**In this repo.** `dispatch.py`: `moon open` prints how many sessions are open before this one, and which; there is no cap. Usage and memory are not shown. Test: `test_dispatch.py`.

<a id="dec-010"></a>
## DEC-010 — The orchestrator opens sessions, never subagents ◆

**Context.** The orchestrator handed a slow job to a background subagent. The desktop chat shell then showed it busy without end; stop did nothing, and the orchestrator had to be replaced (INC-008).
**Options.** Fix the shell and keep subagents; or open full worker sessions only.
**Decision.** 2026-11-22: the operator ruled that the orchestrator opens sessions only, no subagents, background ones included, because he interrupts it often and it must stay responsive.
**Trade-off.** A session costs more to start than a subagent.
**In the private system.** The gate records subagent calls as their own category; it runs in probe mode and has never blocked one.
**In this repo.** `hooks.py` (`orchestrator_does_work` denies the `Agent` and `Task` tools for the orchestrator). Test: `test_hooks.py`.

<a id="dec-011"></a>
## DEC-011 — Hooks installed once, switched by a file; agent settings are human-only ◇

**Context.** Every edit to agent settings risks breaking every session, and agents editing their own permissions is the wrong kind of power.
**Options.** Edit settings whenever a guardrail changes; or install one entry point once and switch it from a file.
**Decision.** 2026-11-22: install-once with three modes (off, probe, gate) came from an external review; the operator approved it. Keeping settings edits for himself is his own, from his authorisation table, so a session prepared install and uninstall scripts and he ran them. Uninstall restores the file byte for byte.
**Trade-off.** One human step up front; afterwards modes change without touching settings.
**In the private system.** Probe mode only since install: 25,523 hook calls across 170 sessions, 0 swallowed errors.
**In this repo.** In part: `moon hook` is the one entry point and reads its mode from `.moonlit/config.json` on every call; it never edits agent settings. With the gate on, `protected_paths` ships in `block` and never demotes: it denies the writes it can see to the workspace's `.claude/settings.json` and `.claude/settings.local.json`, to `rules.json` and to anything under `.moonlit/`, and deleting or moving a folder that holds them. It reads command lines, so a nested session started with settings of its own (`claude --settings ...`) or pointed at another workspace is not seen; eval cases H-159 and H-160 keep both as expected misses. `examples/claude-code/` holds the settings snippet a person installs once. Install and uninstall scripts are not ported. Test: `test_hooks.py`.

<a id="dec-012"></a>
## DEC-012 — A kill switch in front of money and account actions ◇

**Context.** Several programs could touch payments or accounts, each with ad-hoc limits; one bug could spend money twice.
**Options.** Per-program checks; or one shared breaker in front of every sensitive action.
**Decision.** 2026-11-22: an external review designed one breaker with switches, limits, a host allow-list and idempotency keys, where no answer means no; the operator approved it. His two edits: limits change when he tells the orchestrator, and preparation he started himself is never blocked, since only the final payment is his.
**Trade-off.** False denials happen; they are cheaper than a double charge.
**In the private system.** Asked three times so far, all during its acceptance test: two allowed, one denied.
**In this repo.** `breaker.py` (`ask`): a global switch, a switch per program, per-call and per-day limits, a host allow-list and an idempotency key, all decided under the log's lock. Test: `test_breaker.py` (eight processes, one key, exactly one allowed).

<a id="dec-013"></a>
## DEC-013 — The premium model is selectable only by the human ◆

**Context.** The orchestrator should choose model and effort for each task, but the premium model tier costs far more per call.
**Options.** Let the orchestrator choose any model; or choose automatically except for the premium tier.
**Decision.** 2026-11-19: the operator asked that the orchestrator pick model and effort automatically, with the premium tier reserved for his own selection. Every automatic path that opens sessions refuses it and says why. He can still pick it by hand.
**Trade-off.** Tasks that might benefit from the premium tier wait for him.
**In the private system.** Three automatic open paths refuse it; a default set to it falls back to the standard top model.
**In this repo.** `config.py` (`models.human_only`), `dispatch.py` (`_human_allows`): a session on the premium model opens only after the human answers `allow`, with a click on the web board, on the `model:<card>:<model>` decision that dispatch filed for that card and that model. The first time it is asked for, dispatch files that decision itself and refuses; if the human shelves it, the next request files a new one, and another model needs its own answer. `use the default` opens the session on the default model. Tests: `test_authorization.py`.

<a id="dec-014"></a>
## DEC-014 — Authorisation tiers: reversible means act, money or irreversible means ask ◆

**Context.** Agents asked permission for small things, wasting the operator's attention, while a few actions really did need him.
**Options.** Ask about everything; act on everything; or tier by reversibility and cost.
**Decision.** 2026-11-04: the operator told sessions to go ahead with anything not major, unless it involves money or is irreversible and high-impact. On 2026-11-14 he wrote it into a three-tier table: act without asking; act and tell him in one line; only he may do it (payment, final submit, identity checks, permission settings).
**Trade-off.** Agents sometimes act where he would have chosen differently; reversibility keeps that cheap.
**In the private system.** A table in the shared instructions, read by every session.
**In this repo.** Not ported as a policy. Its hard edges are here: human-only cards in `cards.py`, and the breaker in `breaker.py`.

<a id="dec-015"></a>
## DEC-015 — Docs get shorter: soft and hard size limits in code ◆

**Context.** Each project's agent doc grew with history and process notes. Every session reads it at the start, so its size is paid in every context window.
**Options.** A prose rule asking for brevity; or limits enforced by the only sanctioned writer.
**Decision.** 2026-11-22: the 10,000-character line and the always-shorter direction were the operator's, written into his instructions on 2026-11-14. A review proposed adding a hard refusal; he set it at 15,000 instead of the reviewer's tighter number. Above 10,000 a write goes through and files a tidy-up task.
**Trade-off.** A hard refusal can block an urgent note; the soft band leaves room.
**In the private system.** The core's own doc went from 14,992 to 7,682 characters.
**In this repo.** Not ported.

<a id="dec-016"></a>
## DEC-016 — Human approval is spent only on decisions made once that keep working ◆

**Context.** A review proposed that the operator click a web card for every gate change and rule edit; he refused to approve everything himself. Yet sessions kept adding must, never and only rules, and after one rule change web searches per 100 tool calls fell about 40% (INC-010).
**Options.** Approve everything; approve nothing; or ask only where one answer keeps paying off.
**Decision.** 2026-11-22: the operator drew the line: a one-time effort with a lasting effect is worth asking him; trivia is not. The instance: a new restrictive rule waits on a card until his click. The rest he relays in conversation.
**Trade-off.** New guardrails start later.
**In the private system.** 830 existing rules baselined, reviewed ten a week; unacknowledged rules raised 19 warnings at hand-back.
**In this repo.** `rules.py` (`add_rule`): a rule added from the command line, whoever typed it, files a human-only `rule:<id>` card and shows as pending until the human acknowledges it on the web board. Test: `test_rules.py`.

<a id="dec-017"></a>
## DEC-017 — Hooks fail open, the breaker fails closed ◇

**Context.** A tool-call hook that crashes could freeze every session; a money check that crashes could let a payment through.
**Options.** One failure policy for both; or opposite policies matched to the cost of each mistake.
**Decision.** 2026-11-22: an external review proposed both and the operator accepted them unchanged. The hook entry always exits 0, and any exception means allow plus an error-log line. The breaker treats any error, missing config or unreadable amount as deny. A gate category blocking over a fifth of its calls in a day demotes itself to observe.
**Trade-off.** A broken gate stops guarding until someone reads the log.
**In the private system.** The hook has swallowed 0 errors so far; the gate has never run, so no demotion has happened.
**In this repo.** `hooks.py` (`handle_event`, `respond`; errors go to `.moonlit/hooks-errors.log`), `breaker.py` (`ask`). Only `orchestrator_does_work` demotes itself, once the operator has promoted it; the three gates that enforce what the card or moon itself says never do. The demotion needs at least 30 distinct calls and 5 blocks in the day before it can trigger, and a call is its session plus the paths it writes, so one session retrying one write counts once. Tests: `test_hooks.py`, `test_breaker.py`.

<a id="dec-018"></a>
## DEC-018 — Probes earn automation: 3 right, 0 wrong

**Context.** The operator wanted the system to notice when he had done something and move on without his click. Auto-closing on a probe that might be wrong would hide open work.
**Options.** Close on the first probe hit; never close automatically; or shadow mode with promotion.
**Decision.** 2026-11-22: a probe hit only marks the card possibly done; three confirmed hits and none wrong promote it to closing automatically, and one wrong close demotes it. The implementing session set the threshold; the direction, that shadow mode is only a transition, was the operator's.
**Trade-off.** A slow start: promotion needs three confirmations.
**In the private system.** It has never fired: no card has been flagged by a probe yet, so no promotion.
**In this repo.** Not ported. `acceptance.py` runs a card's `probe` once, at hand-back, without promotion.

<a id="dec-019"></a>
## DEC-019 — Effort is the orchestrator's judgment, not a per-card-type default ◆

**Context.** A review proposed fixed effort levels by card type (medium for research, high for code), and that code open a successor session automatically after every handover.
**Options.** Hard-code both; or let code compute and suggest while the orchestrator decides.
**Decision.** 2026-11-22: the operator rejected both. Choosing effort was a power he had already given the orchestrator; a per-type table may serve only as a reference. For handovers he chose the compromise: code records the handover and prepares the successor's brief, and the orchestrator decides with one command whether to open it.
**Trade-off.** Judgment can be inconsistent; a table cannot see the task.
**In the private system.** Effort is set per session at dispatch.
**In this repo.** `dispatch.py`: the effort is the one passed to `moon open`, else the one on the card, else one config default; there is no per-type table. A handover note left by `moon close` goes into the next session's brief; nothing opens that session automatically.

<a id="dec-020"></a>
## DEC-020 — Over-long reports are stored, not refused ◇

**Context.** Workers sometimes sent the orchestrator long reports, and every word lands in its context window. The first design refused reports over the length limit.
**Options.** Refuse long reports; truncate silently; or accept, store the full text, and forward only the start.
**Decision.** 2026-11-22: a second review proposed accepting instead of refusing and the operator approved it. The full text is saved to a file; the orchestrator gets the first paragraph plus a pointer. For long direct messages to the orchestrator, he approved measuring for a week, record only, before any limit applies.
**Trade-off.** The orchestrator may miss detail it never opens.
**In the private system.** Reports over 300 characters are stored in full (53 so far); the message-length week is still being measured.
**In this repo.** Not ported.
