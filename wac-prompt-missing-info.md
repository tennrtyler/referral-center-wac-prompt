# Referral Center — Missing Info / Rejected Writes Prompt

> Customer-agnostic by design. Everything specific to your customer — org
> **slug**, whether E&B and Qualifications are live, which workers to edit
> (**#4**), stage names, the Provide-button path (**#14**), and how those
> workers decide Missing Info vs Rejected — is in the **Linear ticket body**.
>
> **Read that ticket before doing anything.** If a fact is blank, ambiguous, or
> contradicts what you just pulled from prod, **stop and ask**. Do not guess. Make sure to review the 3 `linear-embed` `node-type="file"` markdown files from the issue
description. Download them first. If you cannot find
`wac-prompt-cron.md`, `wac-prompt-missing-info.md`, or `reference-index.md`,
**prompt the user** — `get_issue` often omits inline file embeds.

---

## Your role

You are implementing **Referral Center Missing Info / Rejected writes** for one
Tennr org in the `tennr-workflows` Workflows-as-Code repo
(`~/dev/tennr-workflows`).

This prompt covers:

1. **Missing Info / Rejected / On Track writes** inserted into the existing
   **E&B** worker, if **#2** is yes.
2. **Missing Info / Rejected / On Track writes** inserted into the existing
   **Qualifications** worker named in **#4**. This is mandatory whenever #4
   lists a Qual worker. **Naming the worker is the instruction to edit it.**
   Pull prod, then insert the status writes on every Qual decision path
   (Missing Info / Not Qualified / Qualified → Missing Info / Rejected /
   On Track). Do not skip this because the CRON prompt already ran.
3. The **Provide-button worker**, only if **#14** is `WAC, implement Base Case.`
   If #14 is `WAC, hold off. I got this.`, skip it and say so.

Which workers, and whether each is live, come from the Linear ticket body
(**#2**, **#3**, **#4**).

The CRON that syncs Scheduled / Completed from the EHR is a **separate prompt**
(`wac-prompt-cron.md`). Do not *implement* that work here — but if the CRON's
terminal semantics conflict with Qual/E&B status writes, **report it**.

**The single most important rule: never guess a customer-specific fact.** Worker
names, assistant IDs, stage names, credential names, order-type names, and the
wording a referring provider should see all come from the ticket. If the ticket
is blank, ambiguous, or contradicts what you just pulled from prod, **stop and
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

**5. The repo is not authoritative.** A checked-in `.ts` may be months behind
production. Before reading the Qual or E&B worker named in **#4**, or any
referenced example:

```bash
tennr pull prod --assistant-id <id> --out <path> --slug <slug>
```

Never infer live behaviour from a repo file you did not just pull. Pull drops
the display name and slug — always pass `--slug`.

**6. Resolve the org by slug, not display name.** Ticket **#1** is the folder
under `orgs/` (kebab-case). Confirm with
`ls ~/dev/tennr-workflows/orgs | grep -i '<name>'`. If `ls` does not match,
**stop and ask**. Team ID comes from the pulled workflow's `tennr.config.ts`,
not from the ticket.

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

Read the **Linear ticket body** in full (that is the ESE input). Resolve the
org **slug** from **#1** (`ls orgs | grep -i …`) — not the display name.

**Pull the workers named in #4 from prod before you read them.** Example
shapes, after pull (slug `williams-brothers`):

```bash
# E&B  — orgs/williams-brothers/workflows/visible/e-b/
tennr pull prod --assistant-id 69bd2034fd657e39604be575 \
  --out orgs/williams-brothers/workflows/visible/e-b --slug williams-brothers

# Qualifications — orgs/williams-brothers/workflows/visible/qualifications/
tennr pull prod --assistant-id 69c3f2c87a49e536a12b742a \
  --out orgs/williams-brothers/workflows/visible/qualifications \
  --slug williams-brothers
```

Do not treat the checked-in Williams Brothers `.ts` as live.

1. Restate whether E&B is live (**#2**), whether Qualifications is live
   (**#3**), and which workers you will edit (**#4** — name **and**
   assistantId). If #4 lists a Qual worker, Phase 2 is **mandatory**.
2. If Qualifications is **not** live, missing info may look different —
   **check with Ben Howe** before building. Do not invent a Qual-shaped write
   path.
3. Skip E&B entirely if **#2** is `no`.
4. Pull each named workflow from prod, then read it. Read each folder's
   `learnings.md` first if one exists. Find where E&B decisions and Qual
   decisions actually happen.
5. Confirm `resolvableBy`: default `ALL`, unless the ticket says Missing Info
   should be resolvable by the referring provider only (`REFERRING_USER`).
6. Confirm every stage name in **#10**. If a stage is marked `(e&b)` or
   `(qual)`, that is the stage to write at the start of that worker.
7. Restate **#14**. If it is `WAC, implement Base Case.`, Phase 3 is the
   Provide-button worker. If it is `WAC, hold off. I got this.`, skip Phase 3
   and say so.
8. Create the draft branch and add the workflows you will edit (see Safety
   rule 1). Team ID comes from the pulled metadata, not the ticket.

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

**This phase is mandatory if #4 lists a Qual worker.** If you finish this
prompt without editing that worker, you are not done — say so explicitly.

Pull that worker from prod (`--assistant-id` from #4, `--slug` from #1), then
in that file:

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
Report the file path you edited.

### Phase 3 — Provide-button worker (only if #14 is Base Case)

Skip entirely if **#14** is `WAC, hold off. I got this.` Report that the ESE
owns the fancy path.

If **#14** is `WAC, implement Base Case.`:

1. Pull the copyable template from prod:
   `orgs/flomed/workflows/invisible/patient-pipeline-missing-info/`
   (assistantId `6a3b3f8df4284b7abf8922af`, slug `flomed`).
2. Copy the shape: `WorkflowBlockType.MISSING_INFO` root,
   `confirmInput` mapping `externalId | notes | externalFiles`,
   `combineFiles` → `readTomEntity` by `EXTERNAL_ORDER_ID` → `tom.orderNote`
   → set order `status` back to **`On Track`** → `spawnWorkflow` to **this
   org's** Fax Wrangler.
3. Resolve Fax Wrangler in the **target** org (`tennr team list`, live
   `pinnedMajorVersion`). Do not copy Flomed's or Williams Brothers' workflow
   name or pin.

**Then tell the ESE** they must set this workflow as the Missing Info target
in Referral Pipeline settings (app-side). You cannot do that.

### Phase 4 — Self-review

Verify each item from the CLI (`tennr run list` / `run get` / `run logs`), not
by eyeballing the app. Report pass/fail per line.

- [ ] Orders in `Missing Info` each have a coherent, **externally legible** note
  for what's missing.
- [ ] Every order set to `Rejected` has a rejection reason.
- [ ] E&B insurance-missing and OON (and every other insurance rejection path)
  write status + entity, if E&B is live (**#2**).
- [ ] Qual Missing Info, Not Qualified, and Qualified write status + entity
  on the populations Qual actually decides (**#3**). **Fail this line if
  #4 named a Qual worker and you did not edit it.**
- [ ] Provide-button worker exists and sets `On Track` + Fax Wrangler, **or**
  #14 was hold-off and you reported that.
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
