# Referral Center — Missing Info / Rejected Writes Prompt

> Customer-agnostic by design. Everything specific to your customer — org,
> whether E&B and Qualifications are live, which workers to edit, stage names,
> and how those workers decide Missing Info vs Rejected — is in the **Linear
> ticket body** (the filled ESE input). The prompt files are attachments on
> that ticket.
>
> **Read that ticket before doing anything.** If a fact is blank, ambiguous, or
> contradicts the repo, **stop and ask**. Do not guess.

---

## Your role

You are implementing **Referral Center Missing Info / Rejected writes** for one
Tennr org in the `tennr-workflows` Workflows-as-Code repo
(`~/dev/tennr-workflows`).

This prompt covers **two edits** into existing workers:

1. **Missing Info / Rejected writes** inserted into the existing **E&B** worker.
2. **Missing Info / Rejected writes** inserted into the existing **Qualifications**
   worker.

Which workers, and whether each is live, come from the Linear ticket body
(**#2**, **#3**, **#4**). Read those workers in the repo before writing
anything.

The CRON that syncs Scheduled / Completed from the EHR is a **separate prompt**
(`wac-prompt-cron.md`). Do not do that work here.

**The single most important rule: never guess a customer-specific fact.** Worker
names, assistant IDs, stage names, credential names, order-type names, and the
wording a referring provider should see all come from the ticket. If the ticket
is blank, ambiguous, or contradicts what you find in the repo, **stop and
ask**.

---

## Safety rules (non-negotiable)

**1. Test on drafts only. Never execute a live workflow version.**

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
- Any cascade you drive must spawn children with **`useDraftVersion: true`** so
  the whole chain stays on drafts.
- If you cannot confirm a run targets a draft, **do not run it**. Stop and ask.

**2. No git remote writes.** No `git push`, no PRs, no branch pushes, no
`gh` commands. Leave local edits in the working tree.

**3. "Done" requires real run IDs.** `pnpm typecheck` and `tennr check` are
necessary but never sufficient. A build is complete only when you have actual
`tennr run` IDs whose **step results you read** and which show the expected
status / missing-info / rejection on the expected order. Report the IDs.

**4. Never bind a resource that doesn't exist.** `importCredential`,
`bindSorTable`, `importModule`, and `bindTeamGroup` resolve names that must
already exist in the target org. Confirm with `tennr team list …`,
`tennr team describe module`, and `tennr schema ehr` before writing the binding.

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

Only change `stage` if ticket **#10** marks an `(e&b)` or `(qual)` stage.
These writes are primarily **status + entity**.

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
b.updateTennrObject(
  {
    updates: [{ value: "Missing Info", fieldPath: "status" }],
    variableType: "ORDER",
    variableReference: "order_entity",
  },
  { key: "<uuid>", name: "Set Order Status | Missing Info" },
);
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

**Isolate the TOM writes.** Put Missing Info / Rejected writes next to the
existing decision branch (the path that already decided "missing insurance",
"OON", "not qualified", etc.). Do not invent new decision logic. If you cannot
find the decision site in the named worker, **stop and ask**.

**`pinnedMajorVersion` must match the deployed child's major** if you spawn
anything. Check before pushing.

---

## Phases

Work them in order. Report at each boundary; don't batch up surprises.

### Phase 0 — Verify the input before building

Read the **Linear ticket body** in full (that is the ESE input).
Reference the `E&B` workflow in Williams Brothers to see an example E&B
implementation.
Reference the `Qualifications` workflow in Williams Brothers to see an example
Qualifications implementation.

1. Restate whether E&B is live (**#2**), whether Qualifications is live
   (**#3**), and which workers you will edit (**#4**).
2. If Qualifications is **not** live, missing info may look different —
   **check with Ben Howe** before building. Do not invent a Qual-shaped write
   path.
3. Skip E&B entirely if **#2** is `no`.
4. Read the named workflow folders. Read each folder's `learnings.md` first if
   one exists. Find where E&B decisions and Qual decisions actually happen.
5. Confirm `resolvableBy`: default `ALL`, unless the ticket says Missing Info
   should be resolvable by the referring provider only (`REFERRING_USER`).
6. Confirm every stage name in **#10**. If a stage is marked `(e&b)` or
   `(qual)`, that is the stage to write at the start of that worker.
7. Create the draft branch and add the workflows you will edit (see Safety
   rule 1). Team ID comes from the workflow `.ts` metadata, not the ticket.

### Phase 1 — E&B writes

Skip entirely if **#2** is `no`. Otherwise, in the existing E&B worker named
in **#4**:

- If stage is not updated yet in the worker and **#10** marks an `(e&b)` stage,
  update the stage to that name. This should happen near the beginning of the
  worker before any pause for human review.
- Missing insurance information → `status: "Missing Info"` + `tom.missingInfo`
  with a legible message (e.g. "Missing insurance").
- Out of Network → `status: "Rejected"` + `tom.rejectionInfo` with a reason
  stating Out of Network. Do the same for every other insurance-related
  rejection path in the workflow — find them all, don't stop at the first.
- Otherwise → `status: "On Track"`

**Proof required:** real run IDs showing Missing Info and Rejected (and a
control unchanged), with the entity + status present. Ask the ESE for real
orders if the ticket did not name them.

### Phase 2 — Qualifications writes

In the existing Qualifications worker named in **#4**:

- If stage is not updated yet in the worker and **#10** marks a `(qual)` stage,
  update the stage to that name. This should happen near the beginning of the
  worker before any pause for human review.
- Qual output path `Missing Info` → `status: "Missing Info"` + `tom.missingInfo`
  with the qual decision note as the `message`.
- Qual output path `Not Qualified` (or the equivalent rejected case) →
  `status: "Rejected"` + `tom.rejectionInfo` carrying the qual output note.
- Qual output path `Qualified` → `status: "On Track"`

Use the actual output labels the worker already emits. If they are not
`Missing Info` / `Not Qualified`, map from what you find — do not rename the
qual output to match this prompt.

If **#3** says Qual is only live for a subset (e.g. Medicare only), only write
on the paths that actually run. Do not add Qual writes for populations the
worker does not decide.

**Proof required:** real run IDs showing Missing Info and Rejected from Qual,
with externally legible messages/reasons, and the control order unchanged.

### Phase 3 — Self-review

Verify each item from the CLI (`tennr run list` / `run get` / `run logs`), not
by eyeballing the app. Report pass/fail per line.

- [ ] Orders in `Missing Info` each have a coherent, **externally legible** note
  for what's missing.
- [ ] Every order set to `Rejected` has a rejection reason.
- [ ] E&B insurance-missing and OON (and every other insurance rejection path)
  write status + entity, if E&B is live (**#2**).
- [ ] Qual Missing Info and Not Qualified write status + entity, on the
  populations Qual actually decides (**#3**).
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
