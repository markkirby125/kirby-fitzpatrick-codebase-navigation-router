# Codebase Navigation Router — Technical Operational Dispatcher

**Framework Author**: William Fitzpatrick (*Writer Science*)  
**Source Lecture**: [A.I. Mistakes Writers Must Stop Making](https://www.youtube.com/watch?v=3kf9rRztJgA)  
**Parent Collection**: [Master Collection](../../kirby-fitzpatrick-writers-collection/SKILL.md) | [Global Help](../../../kirby-help/SKILL.md)  

---

## 1. Cognitive Foundation: The File Is Not the System — Navigation Before Mutation

**The concept.** Every edit is a *claim about a contract*. The file you are about to change is a sentence; the repository is the message. You cannot improve a sentence when you do not know what the message asserts — and you cannot know what the message asserts by reading one sentence harder.

Fitzpatrick's diagnosis of the AI-writing mistake is that writers treat style as a surface layer applied to ideas they already own: *the ideas are mine, AI provides the style*. His rebuttal is structural, not moral: **style and substance are not independent variables.** As style is manipulated, the message is distorted, and no clear style can be derived from an ill-conceived idea. The real defect sits below the surface features of the text — in what he calls the *spirit* of the writing, the animating intention that makes the words mean something.

Transposed to software, the variables rename but the dependency does not:

| Writer's term | Engineering equivalent |
|---|---|
| Style (the surface) | The shape of the diff: naming, idiom, structure, formatting |
| Substance (the idea) | The contracts the code participates in: signatures, wire shapes, invariants |
| Spirit (the animating intention) | The *system's* ownership, ordering, idempotency, and failure semantics |
| The ill-conceived idea | An edit made to a file whose call graph was never traced |

**The engineering inversion.** Fitzpatrick identifies two AI workflows — *AI writes, human edits* and *human has the idea, AI edits* — and shows that the second, which feels more responsible, is **just as pernicious**, because it still starts downstream of the conception. Engineering has the same two workflows, and both skip navigation:

1. **AI proposes a diff → human reviews it.** The diff is already shaped. Review concentrates on the surface, and a missing caller is *invisible in a diff that only shows one file*. The green build is read as proof of something it never tested.
2. **Human states the intent → AI writes the keystrokes.** *"I know what I want; let the agent type it."* Still unnavigated, because **intent is not contract**. You can be certain about the outcome you want and wrong about every edge the outcome travels along.

**Why "navigation" rather than "reading."** Reading is a file operation; navigation is a **graph** operation. The information that decides whether your edit is safe lives outside the file: in the callers, the shape that crosses the network, the deploy order, the dashboard someone parses. A model — or a human — that reads only the file has read the smallest and least informative region of the system, then painted it carefully.

**Navigation debt.** An unnavigated edit does not fail locally; it *defers* its cost to a caller you never opened, at the worst hour, paid by someone who has never heard of your change. Navigation debt is the set of true facts about the system that the change silently assumed and nobody verified. Like the coat of paint, it is invisible precisely because the code reads well.

```text
BEFORE — EDIT-FIRST (the file is the unit of work)

  prompt: "add a `currency` field to Invoice"
       │
       ▼
  ┌────────────────────────────┐        read: 1 file, 12 lines of it
  │ invoice.ts  ◄── EDIT HERE  │ ◄────── pick the first name match, start typing
  └────────────────────────────┘
       │ build
       ▼
  ✗ 3 compile errors → edit 2 more files → ✗ 1 runtime error
  → "let me also patch the serializer" → ✗ a downstream export breaks
  ✔ compiles · CI green · reviewed · merged
       ╰── callers never opened: billing worker · tax service · CSV export · mobile client
           (cost deferred to a consumer you have not met)

AFTER — NAVIGATION-FIRST (the call graph is the unit of work)

        R3  consumers of observable behavior   exports · dashboards · SLAs · other repos
      ┌──────────────────────────────────────────────────────────────────────┐
      │  R2  boundary contracts              producer ⇄ consumer, wire/queue  │
      │   ┌──────────────────────────────────────────────────────────────┐   │
      │   │  R1  direct callers & callees in-process                     │   │
      │   │   ┌──────────────────────────────────────────────────────┐   │   │
      │   │   │  R0  the file named in the prompt        ◄─ edit only │   │   │
      │   │   └──────────────────────────────────────────────────────┘   │   │
      │   └──────────────────────────────────────────────────────────────┘   │
      └──────────────────────────────────────────────────────────────────────┘

  READ OUTWARD   R0 → R1 → R2 → R3 · stop when the next ring adds nothing new
  EDIT INWARD    only inside the perimeter you actually read
  The map is the deliverable. The diff is the residue.
```

**The mental model to hold.** *You are not being asked to be careful; you are being asked to be situated.* Care is a posture. Navigation is a receipt: a list of files read, references enumerated, boundaries opened on both sides, and consumers named. Everything in the next two sections is that receipt, made non-optional.

**Related dispatchers.** Seal the ground truth you must not mutate with [Read-Only Vault Isolation](../../kirby-fitzpatrick-read-only-vault-isolation/SKILL.md); keep correctness ahead of cosmetic polish with [Substance-First Refactoring](../../kirby-fitzpatrick-substance-first-refactoring/SKILL.md); supply the depth a map alone cannot carry with [3D Architectural Grounding](../../kirby-fitzpatrick-3d-architectural-grounding/SKILL.md); state the route as a decision record via the [3-Part Proposal Engine](../../kirby-fitzpatrick-3part-proposal-engine/SKILL.md); bound a change before it travels with the [Rhetorical Preflight Gate](../../kirby-fitzpatrick-rhetorical-preflight-gate/SKILL.md).

---

## 2. Core Transformation Protocols

### Rule 1 — The Edit Perimeter Must Be a Subset of the Read Perimeter

Two sets, computed explicitly:

- **Read perimeter**: every file and symbol you have opened *in this session*, with the region read.
- **Edit perimeter**: every file the diff touches.

The invariant: `edit ⊆ read`, and `read ⊇ direct callers of every changed symbol`. A file that appears in the diff but not in the route was never navigated; remove it from the diff or read it now. "I know this file from last month" is not a reading — the file changed, and so did everything around it.

### Rule 2 — Trace the Contract, Not the Implementation

Before touching a symbol, extract three facts:

1. **The shape**: signature, schema, wire format, column, topic, route.
2. **The producers**: who writes that shape.
3. **The consumers**: who reads it — *these decide compatibility, not you.*

One claim, one command, pasted as evidence:

```bash
rg -n "Invoice\b" --type ts              # textual references
rg -n '"invoice"' -g '!**/dist/**'       # string-keyed serialization & routing
rg -n "register.*Invoice" -g '*.{ts,go,py}'   # DI containers, plugin registries
```

**Grep is one ring, and dispatchers are blind.** Enumerate four reference classes, not one: textual, registry/config, serialization/wire, and test. Dynamic dispatch, reflection, string-keyed routes, JSON deserializers, queue topics, and ORM column mappings resolve names at *runtime*; a name search cannot see them.

### Rule 3 — Build the Call Graph Before the Diff, and Show It

Inbound (who calls me), outbound (whom I call), and the contract at each hop. A grep-derived call graph is a **hypothesis** about a dynamically dispatched system — label it as one, and name the probe that would falsify it. An edit resting on an unlabeled hypothesis is navigation debt with no owner.

### Rule 4 — State the Blast Radius in Tiers, as a Table

Radius is *who breaks*, not *what changed*. A diff stat is not a radius.

| Radius | Who is affected | Breaks if | Detected by | Owner |
|---|---|---|---|---|
| R0 | The module itself | — | its own tests | author |
| R1 | In-process callers | signature/behavior changes | compile + unit suite | author |
| R2 | Sibling service across a boundary | shape changes non-additively | contract test / consumer-driven test | service owner |
| R3 | Human or system consuming behavior (export, dashboard, report, SLA) | field, ordering, or latency moves | monitor / named consumer | named human |

Empty cells are findings, not blanks. A cell may say `N/A` only with a positive reason.

### Rule 5 — A Boundary Crossing Is a Separate Trip

Reading one end of a contract is worse than reading neither, because it produces confidence. For every process, network, queue, or database boundary touched: read **both** ends, declare the compatibility rule (additive, versioned, or breaking), state the order (expand-then-contract; producer before consumer), and record the rollback. **Never change both ends of an asynchronous boundary in one commit** — deploy skew makes the intermediate state real, and the intermediate state is what fails at 03:00.

### Rule 6 — Navigate Outward on the Second Failure

Edit → error → edit → error is a map defect, not a typing problem. If the same file needs a third attempt, stop: the call graph you assumed is wrong. Record the surprise as the *output* of the navigation — "expected a queue consumer, found an in-process call at `worker.ts:140`" — because the surprise is the only information the failed edit produced.

### Rule 7 — Earn the Right to Name, Document, and Format

You may not rename, document, or restyle a symbol until you can state in one sentence what it does and who depends on it. Fitzpatrick: *no clear style can be derived from an ill-conceived idea.* Naming is the last mile of understanding, never the first — and a repo-wide formatting pass inside an unnavigated change obscures the radius while borrowing credibility the change has not earned.

### Rule 8 — Refuse the Inverted Workflow

Both AI workflows skip the conception: *"make this messy patch idiomatic"* and *"just make it compile"* both assume the contract was already known. If a rewrite is justified, it happens **after** the route is drawn and the behavior pinned by tests — so it can be proven compatible rather than merely believed to be.

### Anti-Pattern → Clean Replacement Ledger

| Anti-Pattern (Edit-First) | Clean Replacement (Navigation-First) |
|---|---|
| `rg SymbolName` → open the first hit → edit it | Read the caller list; change the definition, then each caller by decision, not by compiler error |
| Adding a field to a DTO without reading its readers | Trace every reader of the wire shape; make it additive/optional, or version the message |
| "It compiles, so the call graph is fine" | Compilation proves *static* edges only. Enumerate dispatch-blind spots: DI, reflection, string routing, serializers, topics |
| Deleting an "unused" export flagged by the IDE | Check dynamic imports, plugin registries, and config-referenced entry points first |
| One commit changing producer and consumer across an async boundary | Expand-then-contract across two deploys; state the skew window and the rollback |
| Renaming a public symbol whose consumers live in other repos | Alias → migrate → remove, with a deprecation window and a version pin |
| Third edit to the same file to silence cascading errors | The map is wrong, not the code. Stop, trace outward from R0, record the surprise |
| "The agent touched 6 files and the diff looks clean" | A clean diff over an unnavigated graph is a coat of paint: it hides the missing contract inside readable code |
| Tree-wide formatter pass "to see the change clearly" | Navigate first, edit minimally, format in a separate commit |
| "CI is green, so no consumer broke" | CI enumerates *compiled* callers. Name the runtime consumers the pipeline cannot see |
| Calling a single name search "navigation" | Search the *shape* (topic, column, payload key, version), not only the identifier |
| Reviewer asks for blast radius, author pastes `--stat` | Radius is a table of who breaks; the stat is a list of what moved |

### The STOP Signals

```text
✗  writing to a file not read in this session
✗  a changed symbol whose callers were never listed
✗  a boundary crossed with only one side read
✗  "unused" established by an IDE hint alone
✗  the third edit to the same file in one attempt
✗  a green CI run offered as the blast radius
✗  a PR that shows the diff but not the route
```

---

## 3. Engineering Application Scenarios

### 3.1 Code Reviews — Review the Route, Not Just the Diff

The reviewer's job is not to read the change carefully; it is to verify that the change was **situated**. Careful reading of an unnavigated diff rewards the wrong skill: fluency. The first question is never *"is this clean?"* — it is *"where is the map?"*

```text
1. ROUTE      Is the navigation trail present? (files read · references listed)
2. CONTRACT   What shape crosses a boundary — additive, versioned, or breaking?
3. RADIUS     Who breaks, at which tier, detected by what, owned by whom?
4. UNMAPPED   What could not be traced, and who is watching it?
5. Only now   implementation correctness · naming · style · structure
```

Steps 2–4 are the ones reviews skip, and they are the only ones a compile cannot answer.

| Diff-Only Comment | Navigation-First Comment |
|---|---|
| *"LGTM — small, clean change."* | *"Two callers aren't in the route: the retention worker and the CSV export. Which one did you check, and what did it return?"* |
| *"Please add a test."* | *"The test that matters here is a consumer-side one: tax-service parses v1 strictly. Add a contract test or show the tolerant-reader rule."* |
| *"Is this field safe to add?"* | *"It's additive for JSON readers. Name the one consumer that iterates keys positionally — that's the R2 break."* |
| *"Why touch this file?"* | *"This file's function is called only from the queue consumer. That means the change is an R2 boundary change, and the producer hasn't been deployed yet."* |

**The blocking verdict, stated plainly:**

> "I'm blocking on navigation, not on quality. This diff is well-written and it crosses a service boundary with only one side read — the producer adds a required field and I can't see the consumer's parse rule. Paste the route (files read, references enumerated) and the R2 row with a detection path, and this is a one-line re-review. A clean diff over an unmapped graph is exactly what makes the breakage expensive: it will pass review, pass CI, and fail in someone else's service."

### 3.2 PR Descriptions — The Navigation Trail as the Spine

A PR description is not a summary of the diff. It is the map that *justifies* the diff — the artifact that lets a reviewer verify the route without re-walking it. If the description is only the diff, you have outsourced your navigation to the reviewer.

```text
ROUTE
  read:  pkg/billing/invoice.ts (L40–L180) · pkg/billing/worker.ts (L120–L170)
  refs:  rg -n "Invoice\b" --type ts → 14 hits · 6 in-repo callers · 2 registries
CONTRACT DELTA
  Invoice.currency: absent → required ISO-4217 string
  wire: JSON payload v1 → additive field; v1 readers unaffected (tolerant reader)
RADIUS
  R1  worker.ts:140      builds payload        no change needed (additive)
  R2  tax-service (Go)   parses v1 strictly    needs v1.1 reader; pinned @v1.0 today
  R3  finance CSV export appends raw JSON      new column; owner @finance notified
  R4  [UNMAPPED] mobile client vendors payload owner @mobile · probe in staging
ORDER
  deploy tax-service v1.1 (tolerant) → then billing producer
  rollback: revert producer only; consumer stays compatible
EVIDENCE
  cargo test -p billing → 34 passed · contract test v1.1 → 6 passed
```

Three properties make this description load-bearing: it **names the unmapped ring** instead of pretending completeness, it **states the deploy order** so the skew window is a plan rather than a discovery, and it makes the **rollback** a one-line fact. A PR body that describes behavior which never ran is polish; a PR body that describes the *route* is the work.

### 3.3 Architecture RFCs / ADRs — The Map as the Artifact

A design proposal is a **hypothesis about the call graph**. An RFC that presents only the target state is asking reviewers to accept an untraced hypothesis and then holding them to it during the incident. The RFC must present the trace.

Required sections for any RFC that crosses a boundary:

1. **Current call graph**, drawn in text — the rings as they exist today, with the entry points named.
2. **Boundary delta** — for each crossing: shape before, shape after, compatibility rule.
3. **Radius and detection** — the R0–R3 table, with owners, for the *transition*, not just the end state.
4. **Migration order, skew window, rollback** — including what the system does while two versions run.
5. **`[UNMAPPED]` list** — what could not be traced statically, each with the probe that would close it.
6. **Rejected routes** — and the ring at which each died ("split at the table boundary; two R1 callers share the transaction, so the boundary moves to the queue").

| ADR as Opinion | ADR as Route |
|---|---|
| "Split the monolith into services." | "Call graph shows 3 shared tables; split at the queue boundary (R2). Two callers stay in-process, one moves." |
| "Use events for decoupling." | "Producer `worker.ts:210`; 4 consumers, 1 requires ordering per `invoice_id` — partition key chosen for it." |
| "Put it behind a feature flag." | "Flag gates R1–R2 only; the CSV export reads the DB directly (R3) and needs its own path." |
| "Performance motivated this change." | "The R1 hop costs 40 ms; the R2 hop already dominates p99 — this optimizes the wrong ring." |

The last row is the one that pays for the discipline: an untraced RFC does not merely risk a bad decision, it *guarantees re-litigation*, because nothing in it can be checked against the system it claims to describe.

---

## 4. Verification Checklist

- [ ] **Edit perimeter ⊆ read perimeter, with receipts.** Every file in the diff was read at the changed regions in this session; every changed symbol has a stated caller list; the route is pasted as raw commands with raw output. A file in the diff that is absent from the route is unnavigated — remove it or read it now.
- [ ] **Every boundary crossing has both sides read and an ordering rule.** For each process, network, queue, or database boundary touched: the shape before and after, the compatibility rule (additive or versioned), the deploy order, and the rollback are written down. No single commit changes both ends of an asynchronous boundary.
- [ ] **The blast radius is a table, not an impression.** Who breaks, at which tier (R0–R3), detected by what, owned by whom — filled, or `N/A` with a positive reason. "CI is green" appears nowhere as evidence of the radius.
- [ ] **The unmapped residue is named, owned, and probed.** Every fact assumed but unverified — runtime dispatch, a vendored client, a dashboard field, a version pin — is tagged `[UNMAPPED]` with an owner and the probe that resolves it. The count is zero *or* every item is listed; I did not reach zero by not looking.
- [ ] **Spot-audit the first step, not the last.** Handed the finished diff cold, I can state in one sentence what each changed module is for, who calls it, and what crosses the boundary at each hop — and the answers are true. Any changed file I cannot describe is a file whose navigation was skipped, however good the diff looks.