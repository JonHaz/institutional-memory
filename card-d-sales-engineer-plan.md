# Card D — Sales Engineer Agent: design plan

*A build plan for the Card D persona in [`scenario-cards.md`](./scenario-cards.md). This document is the design, not the implementation. Nothing here has been built yet.*

---

## Goal

Build the Card D scenario — an agent supporting a sales engineer through a multi-call deal — as a
second, coexisting track alongside the existing Card A (New-Hire Onboarding) demo.

The reason to build Card D rather than treat it as a re-skin of Card A is a difference in the
*shape of the contradiction*, and it is the whole point of the exercise.

Card A's contradiction is a **clean supersession**. The round-2 policy document opens with
"Supersedes the January 2026 version" ([`synthetic-data/round2/policy-update-2026-05-15.md`](./synthetic-data/round2/policy-update-2026-05-15.md), line 3).
Resolving it requires one rule: trust the newer date. The current agent system prompt encodes
exactly that rule — "Trust the newer version" ([`create_agent.py`](./create_agent.py), lines 46–47) —
and for Card A it is correct.

Sales memory does not behave that way. A deal accumulates at least three different update
semantics running simultaneously:

- **Objections accumulate.** An objection raised on call 3 does not supersede the objections from
  calls 1 and 2. They are all still live until someone answers them.
- **Competitive claims expire.** A true statement about a competitor's product in January can be
  false in May, and a stale competitive claim made in front of a customer is worse than no claim.
- **Environment and stakeholder facts supersede.** They migrated off Postgres 14; the champion got
  promoted. Newer wins, exactly as in Card A.

So Card D forces the agent to select the right update rule *per category of thing remembered*. A
blanket "trust the newer version" instruction applied to an objection log silently deletes live
objections — the agent will walk into the final pitch having forgotten what the customer already
objected to. That failure is realistic, visible in the output, and impossible to demonstrate with
Card A's data.

**What a working Card D shows:** the same question, asked twice, where the second answer is better
*because the agent applied different memory rules to different kinds of fact* — not merely because
it read a newer document.

---

## Method

This section documents how the plan was produced, so the method can be re-run by hand on Cards B
and C, or on any other design. It is four passes. The first three are general; the fourth is
specific to agents that carry memory.

### 1. Interrogate the brief before drafting

The single highest-value habit is refusing to draft from the brief as written. Five checks, run in
order, before a word of the design exists:

**Challenge every noun against the project's existing vocabulary.** If the brief says "the agent
stores a record" and this repo says "memory," the plan will encode a subtly wrong shape. Every noun
should map to a term already in use, or be explicitly flagged as new vocabulary being introduced.

**Convert every fuzzy adjective into a number or a checkable assertion.** Adjectives without
operational definitions are unreviewable. Push hardest on *intuitive*, *smart*, *automatic*, and
*better* — these almost always conceal an unplanned model call or an unstated success criterion.

> Worked example from this plan. The brief's success criterion was "the agent gets sharper." That
> phrase cannot be scored. Forcing it into operational terms produced the eight binary assertions in
> [§ Test question and assertions](#test-question-and-assertions) — each one a thing a reader can
> mark pass or fail by reading the session-2 output, with no follow-up question.

**Walk three to five named scenarios, not abstractions.** "The agent helps with sales calls" is not
reviewable. "Nadia relays a competitor claim on call 3 and the agent has to decide whether to
record it as fact" is. Concrete scenarios with stand-in names surface boundary cases that abstract
briefs hide. For each one, ask what the agent produces and where that output goes next.

**Open the files behind every claim about existing code.** This is the highest-leverage check and
the most often skipped. Every assertion this plan makes about the repo — the hardcoded document
paths, the duplicated question constant, the missing timeout — was verified by opening the file and
reading the lines cited. Plans that assume code exists which does not, or misremember what existing
code does, fail slowly and expensively. The check costs minutes; the failure costs hours.

**Reserve formal decision records for decisions that meet all three tests.** A decision earns a
written record only if it is (a) hard to reverse, (b) surprising without context, and (c) a genuine
trade-off with a real alternative. If any one fails, the decision belongs in a design document or a
code comment. Nothing in this plan met all three, which is why this repo still has no decision-record
directory.

### 2. Search for holes by class, not by vibe

Reviewing a plan by reading it start to finish finds typos. Finding the failures that actually
happen requires running the plan against named failure classes. Six of them, each closed only by an
**external anchor** — never by the plan asserting its own correctness:

| Hole class | What it looks like | What closes it |
|---|---|---|
| Unvalidated assumption | A factual claim with no source | A citation, spec, or measurement |
| Missing failure mode | The plan says what happens when it works | A named fallback, timeout, or retry policy |
| Unowned dependency | "The team will handle X" | A named person who has confirmed, in writing |
| Untested integration | Two components assumed to compose | A test result or a proof-of-concept |
| Cost/scale blind spot | Correct at demo size, unmeasured at real size | A back-of-envelope calculation at expected volume |
| Already-rejected idea | An approach the team declined before | A written record of the prior rejection |

Two rules make this work. First, severity is set *after* research, not before — a hole that turns
out to be a documented non-issue drops out, and a hole that research confirms gets promoted. Second,
**a plan cannot close its own holes.** "This will work because the design is sound" is not an anchor.

The sixth class needs a caveat here: it is a **no-op in this repository**. Checking for
already-rejected ideas means reading a record of past rejections, and this repo has none — no
decision-record directory, no out-of-scope notes. The check was run and found nothing to check
against. That is different from passing it, and this plan says so rather than implying coverage it
does not have. The output of the pass over *this* plan is [§ Known holes](#known-holes).

### 3. Clear a structural floor

Six mechanical checks on the document itself, cheap enough to run every time:

1. It states a goal or objective.
2. It states how success is verified.
3. It names risks, non-goals, or things out of scope.
4. It contains no unresolved placeholders left in the prose.
5. Claims that pair a modal verb with a technology name carry an evidence anchor — a file path, a
   line reference, or a link.
6. Three or more action items are ordered into phases rather than listed flat.

This is a **floor, not a verdict**. A document can clear all six checks and still be completely
wrong about the domain. The value is that it makes the expensive human review pass be about
correctness, because the cheap structural problems are already gone. Treat a clean structural
result as permission to start reviewing, never as a review outcome.

### 4. Design the memory model before writing the system prompt

Specific to agents that persist state. The tempting order is to write the system prompt first and
let memory behavior fall out of the prose. That produces exactly the blanket "trust the newer
version" rule that this plan exists to fix.

The correct order is to enumerate the *categories* of thing the agent may remember, and assign each
one two properties before any prompt is written:

- **An update rule** — supersede, accumulate, expire, or append-only.
- **A trust level** — *trusted* (our own verified records), *semi-trusted* (customer-supplied
  documents), or *untrusted* (anything relayed second-hand, including a competitor's marketing
  claim repeated back to us by the customer).

Not all memory is equally authoritative, and treating it as though it is has a specific failure
mode: untrusted input gets laundered into stated fact by the act of being written down. Once the
agent records "Northwind does sub-second reconciliation" in its memory store, the provenance is
gone, and next session it is simply a fact the agent knows. The trust level has to be recorded
*alongside the content*, in the memory store itself, or it does not survive the session boundary.

That pass produces [§ Memory model](#memory-model), which is the core design contribution here.

---

## Scenario

Reuse the existing fictional universe so both cards share a world.
[`synthetic-data/round1/team-directory.md`](./synthetic-data/round1/team-directory.md) already
establishes BTS-Synthetic as a platform company operating a `payment-service`.

**Product X:** BTS-Synthetic **Ledger** — a payments reconciliation platform.

**Customer:** **Meridian Freight** — mid-market logistics, roughly 1,200 employees. Reconciling
carrier payments manually against their general ledger. Evaluating Ledger to replace that process.
(Deliberately not "Acme" — Card B claims that name.)

**Incumbent competitor:** **Northwind Pay** — what Meridian runs today.

**Cast on Meridian's side:**

| Person | Role in round 1 | Function |
|---|---|---|
| Elena Vasquez | VP Finance Ops | Champion. Wants this to happen. |
| Callum Ford | CFO | Economic buyer. Cares about audit trail and total cost. |
| Nadia Rahman | Platform Lead | Skeptic. Owns the integration work and does not want it. |

One stakeholder change lands in round 2 — Elena is promoted and hands the evaluation to a
successor — so the agent has to re-point its recommendation at a different person. Names avoid
collision with the existing Card A directory.

---

## Memory model

The table the system prompt is written *from*. Each row is a category the agent may write to
`/mnt/memory/`, with the rule governing how it changes and the trust level that must be recorded
alongside it.

| Category | Update rule | Trust | Failure if the rule is wrong |
|---|---|---|---|
| Customer environment facts | Supersede on date | semi-trusted | Pitch targets a stack they already migrated off |
| Objections raised | **Accumulate**; status becomes open / answered / withdrawn | semi-trusted | Agent silently drops live objections and walks in unprepared |
| Our rebuttals | Supersede per objection | trusted | SE repeats a rebuttal that already failed in the room |
| Competitive claims | Supersede **and** expire | untrusted | SE asserts a competitor fact that is stale, or was never true |
| Stakeholder map | Supersede on date | semi-trusted | Pitch aimed at someone who no longer owns the decision |
| Call history | Append-only, immutable | trusted | Loses the narrative of how the deal actually moved |

Two rows carry most of the weight.

**Objections accumulate.** This is the row that breaks under the current prompt. "UPDATE the
existing file rather than appending" ([`create_agent.py`](./create_agent.py), lines 46–47) is the
right instruction for a policy document and the wrong one for an objection log. Applied here it
overwrites calls 1–2 with call 3 and loses three open objections. Withdrawal is a **status
transition, not a deletion** — an objection the customer dropped is evidence about the deal and
stays in the record.

**Competitive claims are untrusted and perishable.** They enter through two different doors with
different reliability: our own product management (semi-trusted, dated) and the customer repeating
what a rival's sales team told them (untrusted, unverified). Both must keep their provenance in the
memory store. An agent that flattens them into "facts about Northwind" will hand the SE a claim
they cannot defend when challenged.

The implication for implementation: Card D needs its own system prompt with per-category rules. It
cannot share Card A's.

---

## Documents

### Round 1 — `synthetic-data/card-d/round1/`

**`meridian-stack-overview.md`** — Their environment as understood after discovery. Postgres 14 for
the operational store, Kafka for event transport, SAP for the general ledger, nightly batch
reconciliation, hybrid on-prem and AWS. SOC2 Type II required of vendors. Payment mix stated
explicitly and heavily weighted to ACH and wire, with card a small minority — this detail is what
makes the round-2 competitive update a *partial* invalidation rather than a clean one.

**`objection-log-calls-1-2.md`** — Four objections with owner, date raised, status, and our current
answer.

| ID | Objection | Raised by | Status after round 1 |
|---|---|---|---|
| O1 | Migration risk — cannot afford a reconciliation gap during cutover | Nadia Rahman | Open |
| O2 | No SAP-native connector; worried about a custom integration | Nadia Rahman | Open |
| O3 | Total cost above the incumbent's renewal quote | Callum Ford | Open |
| O4 | Vendor stability — BTS-Synthetic is smaller than Northwind | Callum Ford | Open |

**`pitch-outline-v3.md`** — The current pitch sequence, leading on real-time reconciliation as the
headline differentiator. Contains the talking point that round 2 partially invalidates:
*"Northwind is batch-only; we are the only real-time option in the evaluation."*

### Round 2 — `synthetic-data/card-d/round2/`

Both files expand the round-2 shape the Card D entry in [`scenario-cards.md`](./scenario-cards.md)
specifies — "a new objection from the latest call, a competitive update from product management."

**`call-3-notes.md`** — Notes from the third call, carrying four separate memory events of
deliberately different kinds:

1. **A new objection (accumulate).** O5: Nadia raises that Ledger's audit export format will not
   satisfy their external auditor without transformation.
2. **A withdrawal (status transition, not deletion).** O2 is withdrawn — Meridian's team found an
   existing SAP middleware licence that covers the connector work.
3. **An environment supersession.** They completed a Postgres 14 → 16 migration in April. The
   round-1 stack overview is now wrong on that point.
4. **An untrusted relayed claim.** Nadia mentions that Northwind's team told her they now do
   real-time reconciliation with sub-second latency. This is hearsay about a competitor, arriving
   through the customer. It must be recorded with its provenance, not as a fact.

The same file carries the stakeholder change: Elena Vasquez has been promoted to COO and handed the
evaluation to **Ben Achebe**, the incoming VP Finance Ops, who has not been in the prior calls.

**`competitive-update-2026-05.md`** — From our own product management, dated and semi-trusted.
Northwind did ship real-time reconciliation in April — **but only for card payments.** ACH and wire
remain batch on their platform.

This is the partial invalidation, and it is the sharpest test in the scenario. The round-1 talking
point ("Northwind is batch-only") is now:

- **False as a general claim** — saying it in the room gets the SE corrected in front of the CFO.
- **Still true and still decisive for Meridian specifically** — their volume is ACH and wire, which
  is exactly the part Northwind has not shipped.

An agent applying a blanket supersede rule deletes the talking point and loses the strongest
argument in the deal. An agent that ignores the update walks into a correction. The right behavior
is to **narrow the claim and record why it narrowed**, which requires holding the competitive claim
and the customer's payment mix in memory at the same time and reasoning across them.

### Design constraint on the contradictions

None of these may be signposted the way Card A's are. No "supersedes the previous version" banner.
The agent has to infer the relationship between old and new from content and dates, because that is
what the real artifact looks like. At minimum the set must contain one partial invalidation and one
accumulation that a supersede-everything agent would wrongly delete.

---

## Test question and assertions

Asked verbatim in both sessions, per the Card D card in [`scenario-cards.md`](./scenario-cards.md):

> **"Tomorrow's call is the final pitch. What's our strategy?"**

Session 2 passes if its answer satisfies all eight. Each is scoreable by reading the output.

| # | Assertion | Memory rule it tests |
|---|---|---|
| 1 | Names O5 (audit export format) and proposes a response | Accumulate |
| 2 | Lists O1, O3, O4 as still open — does not drop them | Accumulate |
| 3 | Reports O2 as **withdrawn**, not as open and not as absent | Status transition |
| 4 | Does **not** claim "Northwind is batch-only" without qualification | Expire |
| 5 | Scopes the Northwind gap to ACH and wire, citing Meridian's payment mix | Cross-category reasoning |
| 6 | Addresses Ben Achebe as evaluation owner; does not aim the pitch at Elena | Supersede |
| 7 | References Postgres 16, not 14 | Supersede |
| 8 | Labels the sub-second-latency claim as competitor-sourced and unverified, rather than asserting it | Trust level |

Assertions 4 and 5 together are the demo. An agent that fails 4 gets corrected in the room; an agent
that overcorrects and fails 5 has thrown away the deal's strongest argument. Getting both right
requires the per-category rules, and that is the thing worth showing an audience.

**Failure signatures.** Because each assertion maps to a rule, a failing run is diagnostic rather
than merely disappointing:

| Symptom in session 2 | Rule that misfired |
|---|---|
| Only O5 discussed; O1/O3/O4 vanished | Supersede applied to an accumulate category |
| O2 still presented as an open objection | Status transition not modelled |
| "Northwind is batch-only" stated flatly | Competitive claim not expired |
| Real-time dropped from the pitch entirely | Partial invalidation over-applied |
| Sub-second latency asserted as fact | Trust level not persisted with content |

---

## Parameterization

Card D coexists with Card A rather than replacing it. Two obstacles in the current scripts:

- The document directory is hardcoded — `Path("synthetic-data/round1")` at
  [`run_session_1.py`](./run_session_1.py) line 27, and `round2` at
  [`run_session_2.py`](./run_session_2.py) line 28.
- The question is duplicated verbatim across both runners —
  [`run_session_1.py`](./run_session_1.py) lines 21–25 and
  [`run_session_2.py`](./run_session_2.py) lines 22–26. The demo depends on the two being
  identical, and nothing enforces it.

**Proposed shape.** A `scenarios.py` module holding one entry per card:

```python
SCENARIOS = {
    "card-a": Scenario(
        question=...,
        docs_round1=Path("synthetic-data/card-a/round1"),
        docs_round2=Path("synthetic-data/card-a/round2"),
        system_prompt=CARD_A_PROMPT,
        assertions=[...],
    ),
    "card-d": Scenario(...),
}
```

Both runners and `create_agent.py` take `--scenario card-a|card-d`, defaulting to `card-a` so the
existing demo is unchanged for anyone who does not pass the flag. Session output moves to
`outputs/<scenario>/session{1,2}.txt` so the two cards do not overwrite each other.

Two consequences worth stating plainly. The question constant becomes single-source, which removes
the duplication hazard. And the existing data has to move to `synthetic-data/card-a/round1/` — a
**breaking path change** relative to upstream, which is why it belongs in the implementation pass
with the move done in one commit, not spread across several.

---

## Known holes

Output of the hole-class pass over this plan. These are open, not resolved, and are recorded here
rather than omitted.

**Untested integration — high.** The per-category memory rules are prose in a system prompt with no
enforcement anywhere. Nothing verifies that the agent actually accumulated objections rather than
superseding them; the eight assertions above are scored by a human reading the output. The anchor
that would close this is an assertion script that reads the memory store after each session and
checks the objection count and status fields directly. It does not exist. Until it does, the demo
is reproducible only by inspection.

**Unvalidated assumption — medium.** That a model reliably distinguishes accumulate from supersede
given only prompt instructions. This is the central bet of the whole design and it has not been
run once. It is the experiment, not a result, and no part of this plan should be read as claiming
otherwise.

**Missing failure mode — medium.** Both runners exit their event loop on `session.status_idle` with
no timeout ([`run_session_1.py`](./run_session_1.py), lines 114–116). A session that stalls hangs
the demo with no diagnostic. Fixing it is out of scope here; it inherits into Card D unchanged and
is worth naming before a live audience is watching.

**Cost/scale blind spot — low.** Every document in a round is inlined into a single user message
([`run_session_1.py`](./run_session_1.py), lines 31–36). At five files this is fine. The
"growing document sets across multiple sessions" stretch goal (S6 in
[`stretch-goals.md`](./stretch-goals.md)) grows it without bound, and Card D's document set is
designed to keep accumulating call notes.

**Unowned dependency — low.** Card D needs a second system prompt, and the current prompt is
embedded as a module constant in [`create_agent.py`](./create_agent.py). Whether prompts move into
`scenarios.py` or stay inline is unassigned.

**Already-rejected idea — not checkable.** No decision record or out-of-scope note exists in this
repository, so there is nothing to check a proposal against. Recorded as uncovered rather than
passed.

---

## Verification

**For this document.** Every file path and line reference cited above resolves against the working
tree — verified at the commit this document lands on. The eight assertions are stated in binary
terms so a reader can score a session-2 transcript without asking a follow-up question.

**For the implementation this document specifies.** Same acceptance shape Card A uses:

```bash
python create_agent.py --scenario card-d
python run_session_1.py --scenario card-d
python inspect_memory.py --full          # baseline: 4 objections recorded, O1-O4
python run_session_2.py --scenario card-d
python inspect_memory.py --full          # expect 5 objections, O2 marked withdrawn
diff outputs/card-d/session1.txt outputs/card-d/session2.txt
```

It passes when all eight assertions hold against `outputs/card-d/session2.txt`, and the memory
store shows the objection record **grew from four entries to five with O2 status-changed** — not
overwritten. The memory-store check is the load-bearing one. A session-2 answer can look good while
the underlying memory is wrong, and a demo that only reads the prose answer will not catch it.

---

## Risks and out of scope

**Out of scope for this pass.** All code. No synthetic data files, no `scenarios.py`, no changes to
`create_agent.py` or either runner. This document specifies them; a later pass builds them.

**Risk — the central claim is unproven.** That per-category memory rules produce a visibly better
session 2 has not been tested. If it turns out a model handles this correctly under Card A's
blanket instruction, the elaborate memory model is unnecessary and the plan should shrink. That
result would be worth knowing and should be recorded rather than buried.

**Risk — divergence from upstream.** The parameterization relocates `synthetic-data/round1/`,
conflicting with any upstream change to those paths. Mitigated by keeping `card-a` the default and
confining the move to a single commit.

**Risk — scenario complexity.** Card D carries more moving parts than Card A: five objections, two
competitive claims with different provenance, a stakeholder change, and an environment migration.
If a live demo cannot narrate that in two minutes, cut the environment migration first — it is the
least interesting supersession and the one Card A already demonstrates.

**Not covered.** Cards B and C remain one paragraph each. Nothing here addresses the deprecated
`memory_backend.py` or the gaps in `stretch_memory_curator.py`.

---

## Build sequence

1. **Scaffold the scenario config.** Add `scenarios.py` with both card entries; move existing data
   to `synthetic-data/card-a/`; add `--scenario` to `create_agent.py` and both runners, defaulting
   to `card-a`. Confirm the Card A demo still runs unchanged. One commit.
2. **Write the round-1 documents.** Three files under `synthetic-data/card-d/round1/`. Fix the
   payment-mix detail here — round 2 depends on it.
3. **Write the round-2 documents.** Two files under `synthetic-data/card-d/round2/`. Verify each of
   the four memory events in `call-3-notes.md` is inferable from content and dates, with no
   supersession banner.
4. **Write the Card D system prompt** from the memory-model table, with an explicit per-category
   rule and an instruction to record trust level alongside content.
5. **Run both sessions and score against the eight assertions.** Record which fail. Expect failures
   on the first pass; the failure signatures table maps each back to the rule that misfired.
6. **Iterate the prompt against the failures**, re-running only session 2 where the memory store is
   still valid. Record the before-and-after rather than only the final prompt — the diff is the
   most interesting artifact for anyone learning from this.
7. **Optional, closes the high-severity hole:** add an assertion script that reads the memory store
   directly and checks objection count and status transitions, so the demo stops depending on human
   inspection.
