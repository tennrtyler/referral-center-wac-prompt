# Referral Center — Missing Info / Rejected Writes Prompt

> Customer-agnostic by design. Everything specific to your customer — org
> folder, whether E&B and Qualifications are live, which workers to edit
> (**#4**), stage names, the Provide-button instructions (**#13**), and how
> those workers decide Missing Info vs Rejected — is in the **Linear ticket
> body** (the filled ESE input). The prompt files are attachments on that
> ticket.
>
> **Read that ticket before doing anything.** If a fact is blank, ambiguous, or
> contradicts the repo, **stop and ask**. Do not guess.
>
> **Get the attachments first.** The prompt files are `linear-embed
> node-type="file"` attachments on the issue description. `get_issue` does not
> reliably return inline embeds, and an empty `attachments` array does **not**
> mean there are none. Download every attached file before you start. If the
> ticket names a file you cannot retrieve, **ask**; do not proceed without it.

---

## Your role

Implement **Referral Center Missing Info / Rejected writes** for one Tennr org
in the `tennr-workflows` Workflows-as-Code repo (`~/dev/tennr-workflows`):

1. Missing Info / Rejected / On Track writes inserted into the existing **E&B**
   worker, if **#2** is yes.
2. Missing Info / Rejected / On Track writes inserted into the existing
   **Qualifications** worker named in **#4**. This is mandatory whenever #4
   lists a Qual worker — **naming the worker is the instruction to edit it.**
   Do not skip this because the CRON prompt already ran; if that prompt said
   Qual/E&B writes were out of scope *there*, the exclusion does not apply here.
3. The **Provide-button worker**, only if **#13** says
   `WAC, implement the default Missing Info worker`. Otherwise follow the
   explicit instructions in #13, or skip it and say so if the ESE is handling it.

Which workers, and whether each is live, come from the Linear ticket body
(**#2**, **#3**, **#4**). **Pull each from prod before reading it** (Safety rule
1) — the repo copy is routinely months stale, and these are live
customer-facing workers.

The CRON that syncs Scheduled / Completed from the EHR is a **separate prompt**
(`wac-prompt-cron.md`). Do not implement that work here — but if the CRON's
terminal semantics conflict with the Qual/E&B status writes, **report it**.

**Run that prompt first.** This one demotes a passing Qual decision from
`Completed` to `On Track`, which is only correct if something downstream
eventually completes the order. Without the CRON in place, qualified orders sit
`On Track` forever — worse than where you started. If the CRON does not exist
yet for this org, **stop and say so** before changing the Qual status mapping.

**Never guess a customer-specific fact.** Worker names, assistant IDs, stage
names, credential names, order-type names, and the wording a referring provider
should see all come from the ticket. If the ticket is blank, ambiguous, or
contradicts what you find in the repo, **stop and ask**.

---

## Safety rules (non-negotiable)

**1. The repo is not authoritative for existing workers — pull prod first.**

A checked-in `.ts` can be months behind production. Before reading any existing
workflow to learn how the org behaves:

```bash
tennr pull prod --assistant-id <id> --team-id <teamId> --out <path> --slug <slug>
```

This matters more here than anywhere else: this prompt **edits live
customer-facing workers**, and the decision branch you are inserting next to may
not exist in the repo copy at all. `pull` drops the display name and slug, so
pass `--slug`. Applies to reference workers too — pull before copying.

**2. Test on drafts only. Never execute a live workflow version.**

The CLI always talks to prod (`api.tennr.com`) — "dev" vs "prod" is *which
version executes*, not which host. Before any `tennr run start`, confirm the
target is a draft you intend.

```bash
cd orgs/<org>/workflows/<workflow>
tennr branch create referral-center --team-id <the workflow's teamId>
tennr branch add ./<workflow>.ts
tennr branch switch referral-center --team-id <teamId>
tennr push   --branch referral-center --team-id <teamId> --message "<change>"
tennr run start ./<workflow>.ts --branch referral-center --team-id <teamId>
```

- `branch create` / `switch` / `delete` default to the CLI's *global* active
  team, **not** the workflow's — always pass `--team-id` from the `.ts` metadata.
- `.git/tennr/target.json` (the active-branch pointer) is **shared, mutable
  state**; a concurrent `tennr` command overwrites it. Never trust
  `tennr branch current` to persist — always name the target explicitly with
  `--branch` and `--team-id` on every push/run/check.
- To exercise a cascade on drafts, set **`useDraftVersion: true`** on the spawn
  and revert it before pushing. Never leave it set on a pushed version: it
  permanently points production at the child's draft.
- If you cannot confirm a run targets a draft, **do not run it**. Stop and ask.

**3. No git remote writes.** No `git push`, no PRs, no branch pushes, no
`gh` commands. Leave local edits in the working tree.

**4. "Done" requires real run IDs.** `pnpm typecheck` and `tennr check` are
necessary but never sufficient. A build is complete only when you have actual
`tennr run` IDs whose **step results you read** and which show the expected
status / missing-info / rejection on the expected order. Report the IDs.

**5. Never bind a resource that doesn't exist.** `importCredential`,
`bindSorTable`, `importModule`, and `bindTeamGroup` resolve names that must
already exist in the target org. Confirm with `tennr team list …`,
`tennr team describe module`, and `tennr schema ehr` before writing the binding.

**6. Resolve the org by its `orgs/` folder name, not its display name.** Ticket
**#1** is the folder under `orgs/` (kebab-case). Confirm with
`ls ~/dev/tennr-workflows/orgs | grep -i '<name>'`. If `ls` does not match,
**stop and ask**. Team ID comes from the pulled workflow's metadata, not from
the ticket.

---

## TOM contracts (verified against the installed SDK — treat as hard facts)

### Order status is a closed enum

Exactly four values. Nothing else validates:

```
On Track | Missing Info | Rejected | Completed
```

This prompt only writes **`On Track`**, **`Missing Info`**, and **`Rejected`**.
Do not write `Completed` from E&B or Qualifications.

### Order stage is free text that must match app configuration

`stage` must be an **exact string match** of a stage configured on the team's
Referral Pipeline configuration page — casing, spacing, and punctuation
included. Omitting it defaults to the team's default stage. There is no
validation error for a typo; it just doesn't take effect as intended.

Only change `stage` if ticket **#10** marks an `(e&b)` or `(qual)` stage. These
writes are primarily **status + entity**.

### Hold one handle per entity

`updateTennrObject` takes a `variableReference` to an existing handle. Read the
order once into `order_entity` (or reuse the handle the worker already has) and
update *that* reference. Don't re-read the same order into new variables, and
don't fork `tom_status`/`tom_status_1` copies of entity-derived fields.

### Reading the order

If the worker does not already hold an Order entity, look it up by EHR id:

```ts
b.readTomEntity(
  {
    entitySearch: {
      entityType: "ORDER",
      searchType: EntitySearchType.EXTERNAL_ORDER_ID,
      externalOrderId: "{{order_id}}",
    },
    errorOnNoMatch: true,
  },
  { key: "<uuid>", as: "order_entity", name: "Find Order in Tennr (by EHR ID)" },
);
```

### Missing Info

Fields: `message` (**required** — the human-readable description shown in
Patient Hub and Referral Pipeline), `notes` (optional string list), and
`resolvableBy` (**required** — `ALL` | `PATIENT` | `REFERRING_USER` |
`TENNR_USER`). Default to `ALL` unless the ticket says the referring provider
only.

**Always pair it with the status write.** The entity is what renders; the status
is what filters.

```ts
b.createTennrObject(
  tom.missingInfo(
    {
      message: "{{decision_note}}",
      resolvableBy: "ALL",
    },
    {
      dependencies: {
        entityType: "MISSING_INFO",
        orderReference: { entityType: "ORDER", inputVariableName: "order_entity" },
      },
    },
  ),
  { key: "<uuid>", as: "created_missing_info" },
);
b.updateTennrObject(
  {
    updates: [{ value: "Missing Info", fieldPath: "status" }],
    variableType: "ORDER",
    variableReference: "order_entity",
  },
  { key: "<uuid>", name: "Set Order Status | Missing Info" },
);
```

The `message` must be **externally legible** — a referring provider reads it.
"Missing insurance" is good. "QUAL_FAIL_3" is not.

### Rejection Info

Fields: `reason` (**required**) + `notes` (optional string list). Always paired
with `status: "Rejected"`. **Never set status `Rejected` without a reason** —
that's a QA checklist item.

```ts
b.createTennrObject(
  tom.rejectionInfo(
    {
      notes: "{{notes}}",
      reason: "Rejected for the following reason below:",
    },
    {
      dependencies: {
        entityType: "REJECTION_INFO",
        orderReference: { entityType: "ORDER", inputVariableName: "order_entity" },
      },
    },
  ),
  { key: "<uuid>", as: "created_rejection_info" },
);
b.updateTennrObject(
  {
    updates: [{ value: "Rejected", fieldPath: "status" }],
    variableType: "ORDER",
    variableReference: "order_entity",
  },
  { key: "<uuid>", name: "Set Order Status | Rejected" },
);
```

`notes` is a **LIST** of strings. To build one from a single value:

```ts
b.createVariables(
  [variableOps.createTextList("note", ["{{reject_reason}}"],
    { itemAs: "elem", useCommaSeparatedValues: false })],
  { key: "<uuid>" },
);
```

### Build rules that apply to these inserts

**Format Text in JSON mode silently drops empty/null fields.** They're omitted
from the output entirely, not written as `""`. Default a value with
`{{var,"fallback"}}` rather than assuming it passes through.

**`notes` is a LIST — build it with `useCommaSeparatedValues: false`.** A
decision note containing a comma is otherwise split into several bogus notes.

**Create the entity BEFORE setting the status, and put both in the SAME
`tryCatch`.** If the entity write fails, the status is then never changed and
the order keeps its previous state — you never strand one as `Rejected` with no
reason, or `Missing Info` with nothing to act on. Two details matter equally:

- *Order.* Status-first leaves exactly the broken state this prompt exists to
  prevent.
- *One `tryCatch`.* Giving the entity its own block swallows the error and the
  status is written regardless — same broken state, just harder to spot.

Both reference implementations in `reference-index.md` create first.

(This is the opposite of the note/date ordering in `wac-prompt-cron.md`, and
deliberately so: a note is decoration, so the status write goes first there. A
Missing Info or Rejection entity is *required*, so it goes first here.)

**Isolate the TOM writes.** Put Missing Info / Rejected writes next to the
existing decision branch — the path that already decided "missing insurance",
"OON", "not qualified". Do not invent new decision logic. If you cannot find the
decision site in the named worker, **stop and ask**.

**`pinnedMajorVersion` must match the deployed child's major** if you spawn
anything. Check before pushing.

---

## Phases

Work them in order. Report at each boundary; don't batch up surprises.

### Phase 0 — Verify the input before building

Read the **Linear ticket body** in full (that is the ESE input). Resolve the
org folder from **#1** (`ls orgs | grep -i …`) — see Safety rule 6. Pull the
reference workers listed under "Missing Info / Rejected writes to insert into
existing workers" in `reference-index.md` before reading them, and copy their
*shape* only — stage names, decision labels and message wording belong to the
org they came from.

1. Restate whether E&B is live (**#2**), whether Qualifications is live
   (**#3**), and which workers you will edit (**#4**).
2. If Qualifications is **not** live, missing info may look different —
   **check with Ben Howe** before building. Do not invent a Qual-shaped write
   path.
3. Skip E&B entirely if **#2** is `no`.
4. Read the named workflow folders — from a **prod pull**, not the repo copy.
   Read each folder's `learnings.md` first if one exists. Find where E&B and
   Qual decisions actually happen.
5. **Record what each decision path writes today**, before changing anything. A
   path setting `Completed`, or setting a status with no accompanying entity,
   are both defects this prompt fixes. Report the current mapping alongside the
   new one.
6. Confirm `resolvableBy`: default `ALL`, unless the ticket says Missing Info
   should be resolvable by the referring provider only (`REFERRING_USER`).
7. Confirm every stage name in **#10**. If a stage is marked `(e&b)` or
   `(qual)`, that is the stage to write at the start of that worker.
8. Restate **#13**. If it says `WAC, implement the default Missing Info worker`,
   Phase 3 is the Provide-button worker. If it gives other explicit
   instructions, follow those. If the ESE said they will handle it, skip
   Phase 3 and say so.
9. Create the draft branch and add the workflows you will edit (see Safety
   rule 2). Team ID comes from the workflow `.ts` metadata, not the ticket.

### Phase 1 — E&B writes

Skip entirely if **#2** is `no`. Otherwise, in the existing E&B worker named
in **#4**:

- If the worker does not update stage yet and **#10** marks an `(e&b)` stage,
  update the stage to that name, near the beginning of the worker and before any
  pause for human review.
- Missing insurance information → `status: "Missing Info"` + `tom.missingInfo`
  with a legible message (e.g. "Missing insurance").
- Out of Network → `status: "Rejected"` + `tom.rejectionInfo` with a reason
  stating Out of Network. Same for every other insurance-related rejection path
  in the workflow — find them all, don't stop at the first.
- Otherwise → `status: "On Track"`

**Proof required:** real run IDs showing Missing Info and Rejected (and a
control unchanged), with the entity + status present. Ask the ESE for real
orders if the ticket did not name them.

### Phase 2 — Qualifications writes

In the existing Qualifications worker named in **#4**:

- If the worker does not update stage yet and **#10** marks a `(qual)` stage,
  update the stage to that name, near the beginning of the worker and before any
  pause for human review.
- Qual output path `Missing Info` → `status: "Missing Info"` + `tom.missingInfo`
  with the qual decision note as the `message`.
- Qual output path `Not Qualified` (or the equivalent rejected case) →
  `status: "Rejected"` + `tom.rejectionInfo` carrying the qual output note.
- Qual output path `Qualified` → `status: "On Track"`. **Not `Completed`** —
  passing qualification is not the patient receiving care, and `Completed` shown
  to a referring provider means their patient was taken care of. If the worker
  currently writes `Completed` here, that is the defect; changing it is the
  point. Some reference orgs treat Qual as terminal and legitimately write
  `Completed` — that only holds where nothing downstream delivers.

Use the actual output labels the worker already emits. If they are not
`Missing Info` / `Not Qualified`, map from what you find — do not rename the
qual output to match this prompt.

If **#3** says Qual is only live for a subset (e.g. Medicare only), only write
on the paths that actually run. Do not add Qual writes for populations the
worker does not decide.

**Proof required:** real run IDs showing Missing Info and Rejected from Qual,
with externally legible messages/reasons, and the control order unchanged.

### Phase 3 — Provide-button worker (gated on #13)

Skip entirely — and say so — if **#13** says the ESE is handling it. If #13
gives explicit custom instructions, follow those instead of the base case.

If **#13** says `WAC, implement the default Missing Info worker`:

1. Pull the copyable template from prod:
   `orgs/flomed/workflows/invisible/patient-pipeline-missing-info/`
   (assistantId `6a3b3f8df4284b7abf8922af`, slug `flomed`).
2. Copy the shape: `WorkflowBlockType.MISSING_INFO` root, `confirmInput`
   mapping `externalId | notes | externalFiles`, `combineFiles` →
   `readTomEntity` by `EXTERNAL_ORDER_ID` → `tom.orderNote` → set order
   `status` back to **`On Track`** → `spawnWorkflow` to **this org's** Fax
   Wrangler.
3. Resolve Fax Wrangler in the **target** org (`tennr team list`, live
   `pinnedMajorVersion`). Do not copy Flomed's workflow name or pin.

**Then tell the ESE** they must set this workflow as the Missing Info target in
Referral Pipeline settings (app-side). You cannot do that.

### Phase 4 — Self-review

Verify each item from the CLI (`tennr run list` / `run get` / `run logs`), not
by eyeballing the app. Report pass/fail per line.

- [ ] Orders in `Missing Info` each have a coherent, **externally legible** note
  for what's missing.
- [ ] Every order set to `Rejected` has a rejection reason.
- [ ] E&B insurance-missing and OON (and every other insurance rejection path)
  write status + entity, if E&B is live (**#2**).
- [ ] Qual Missing Info, Not Qualified, and Qualified write status + entity, on
  the populations Qual actually decides (**#3**). **Fail this line if #4 named
  a Qual worker and you did not edit it.**
- [ ] Provide-button worker exists and sets `On Track` + spawns Fax Wrangler,
  **or** #13 said otherwise and you reported what you did (or skipped) instead.
- [ ] The control order is unchanged.

Anything failing goes to Ben Howe, not into a silent workaround.

---

## Reporting

At each phase boundary, report a table:

| Case | Input | Expected | Actual | Run ID |
| - | - | - | - | - |

Plus, explicitly: what you did **not** do and why. If something is blocked,
finish everything else and say precisely what's left. Do not describe a phase as
complete on the strength of a typecheck.
