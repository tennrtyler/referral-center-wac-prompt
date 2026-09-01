# Referral Center — CRON Sync Prompt

> Customer-agnostic by design. Everything specific to your customer — org
> folder, credentials, WIP mappings, date fields, modules, stages, how to read
> status out of the EMR, and how the CRON should loop — is in the **Linear
> ticket body** (the filled ESE input). The prompt files are attachments on that
> ticket.
>
> **Read that ticket before doing anything.** If a fact is blank, ambiguous, or
> contradicts the repo, **stop and ask**. Do not guess.
>
> **Get the attachments first.** The prompt files are `linear-embed
> node-type="file"` attachments on the issue description. `get_issue` does not
> reliably return inline embeds, and an empty `attachments` array does **not**
> mean there are none — the files can be present in the description body and
> stripped from the response. Download every attached file before you start. If
> the ticket names a file you cannot retrieve, **ask**; do not proceed without it.

---

## Your role

Implement the **Referral Center CRON** for one Tennr org in the
`tennr-workflows` Workflows-as-Code repo (`~/dev/tennr-workflows`) — one build
plus a gated backfill:

1. A **CRON workflow** that syncs order Stage and Status from the customer's EHR
   into TOM for **Scheduled** and **Completed**.
2. A self-review of that sync.
3. A stale-order backfill — **only if approved**.

Missing Info / Rejected writes in E&B and Qualifications are a **separate
prompt** (`wac-prompt-missing-info.md`). Do not **implement** that work here —
but if the E&B or Qual status mapping conflicts with your terminal semantics,
**report it**. (The common case: a Qual worker that sets `Completed` when
qualification passes. See Phase 0.)

**This is prompt 1 of 2.** When this build is done, run
`wac-prompt-missing-info.md` against this same ticket (or state explicitly that
it is the next session). Naming a Qual worker on the ticket is an instruction to
edit it there — do not treat this file as the whole job, and do not leave the
Qual/E&B writes silently undone.

**Never guess a customer-specific fact.** WIP state names, stage names, EHR
field names, module versions, credential names, order-type names, and looping
strategy all come from the Linear ticket body. A wrong WIP state silently syncs
nothing; a wrong stage name silently fails to match. Neither errors loudly.

---

## Safety rules (non-negotiable)

**1. The repo is not authoritative for existing workers — pull prod first.**

A checked-in `.ts` can be months behind production. Before reading any existing
workflow to learn how the org behaves:

```bash
tennr pull prod --assistant-id <id> --team-id <teamId> --out <path> --slug <slug>
```

Never infer live behaviour from a repo file you did not just pull. `pull` drops
the display name and slug, so pass `--slug`. Applies to the reference workers in
`reference-index.md` too — pull before copying.

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
  *and* remove the testing-only stop, then revert both before pushing. An
  `isTestingOnly` `WORKFLOW_END` before the spawn is the stronger guarantee —
  test runs then never spawn at all. Never leave `useDraftVersion` set on a
  pushed version: it permanently points production at the child's draft.
- If you cannot confirm a run targets a draft, **do not run it**. Stop and ask.

**3. No git remote writes.** No `git push`, no PRs, no branch pushes, no
`gh` commands. Leave local edits in the working tree.

**4. "Done" requires real run IDs.** `pnpm typecheck` and `tennr check` are
necessary but never sufficient. A build is complete only when you have actual
`tennr run` IDs whose **step results you read** and which show the expected
stage/status/note on the expected order. Report the IDs.

**5. Never bind a resource that doesn't exist.** `importCredential`,
`bindSorTable`, `importModule`, and `bindTeamGroup` resolve names that must
already exist in the target org. Confirm with `tennr team list …`,
`tennr team describe module`, and `tennr schema ehr` before writing the binding.

**6. Resolve the org by its `orgs/` folder name, not its display name.** Ticket
**#1** is the folder under `orgs/` (kebab-case). Confirm with
`ls ~/dev/tennr-workflows/orgs | grep -i '<name>'` — "Tactile" is not a folder;
`tactile` might be. If `ls` does not match, **stop and ask**. Team ID comes from
the pulled workflow's `tennr.config.ts` / `defineWorkflow` metadata, not from
the ticket.

---

## The target pipeline shape

Build toward this:

1. Starts at **Referral Received** (the default stage) when the order is created.
2. Intermediate stages carry status **`On Track`**, **`Missing Info`**, or
   **`Rejected`** — **never `Completed`**. `Completed` is reserved for the
   terminal stage.
3. Includes a **penultimate Scheduled / Approved / Pending Delivery** stage,
   meaning everything is ready except the item being delivered or the patient
   showing up — **unless the ticket says the EHR has no Scheduled concept**.
   In that case skip Scheduled entirely; do not invent a mapping.
4. Includes a **terminal Completed** stage where status is set to `Completed`.
   This is how the referring provider knows the patient got their care.

A representative configured pipeline:

```
Referral Received → Processing Order → Eligibility & Benefits →
Qualifications → Delivery Scheduled →
Delivery Completed
```

**Before writing any sweep logic, check #10 for a terminal stage.** The common
failure is an org with several stages configured and no terminal one — every
stage is work-in-progress, so a completed order has nowhere to land. That blocks
the whole sync: `updateTennrObject` hard-errors `Order stage not found` on an
unconfigured literal, so every completion fails. Adding a stage is a fast
app-side change — ask for it and get the exact string back before you build.
Same if the org has no stages at all: `Referral Received → Scheduled →
Completed` is a legitimate minimum (drop Scheduled if #6 says to ignore it). Use
the stage names from **#10**, character for character.

**If the stage cannot be added before you build** (or #10 says
`no completed stage`), do not hard-error and do not invent a stage name.
Default — unless the ESE picks otherwise:

1. **Status only, stage behind a flag (recommended)** — always write
   `status = "Completed"` (valid even with no matching stage) plus the
   completed-date note from **#9**, and gate the stage write on a
   `completed_stage` variable that **ships empty**. One-line change the moment
   an admin adds a terminal stage; shippable today; cannot hard-error.
2. **Name the stage now** — the ESE gives the exact terminal stage literal and
   confirms it is configured; you write stage + status unconditionally.
3. **Status only, no stage support** — only ever write `status = "Completed"`.
   Needs a code change later to drive the pipeline column.

Either way, tell the ESE they still need to **add a Completed stage in Referral
Pipeline settings** — referring providers need a Completed column.

---

## TOM contracts (verified against the installed SDK — treat as hard facts)

### Order status is a closed enum

Exactly four values. Nothing else validates:

```
On Track | Missing Info | Rejected | Completed
```

### Order stage is free text that must match app configuration

`stage` must be an **exact string match** of a stage configured on the team's
Referral Pipeline configuration page — casing, spacing, and punctuation
included. Omitting it defaults to the team's default stage. There is no
validation error for a typo; it just doesn't take effect as intended.

### Setting stage and status

Set both in one step where both change:

```ts
b.updateTennrObject(
  {
    updates: [
      { value: "{{target_stage}}", fieldPath: "stage" },
      { value: "{{target_status}}", fieldPath: "status" },
    ],
    variableType: "ORDER",
    variableReference: "order_entity",
  },
  { key: "<uuid>", name: "Update Order Stage + Status in Tennr" },
);
```

### Dates: prefer the first-class fields, and *also* write the note

The Order entity has real date fields the deployment plan doesn't mention:

- **`dateFulfilled`** (DATE) — set when the requested device, medication, or
  service has been delivered to the patient.
- **`authoredOn`** (DATE) — the date the referring provider authored the order.
  When set, Patient Hub / Referral Pipeline show it instead of the
  system-generated received timestamp.

Write `dateFulfilled` when you set status to `Completed` and you have a real
delivery date (ticket **#9**). **Also** write an order note — the note is the
human-legible surface the referring provider actually reads. Use `parseDate` to
convert an EHR date string before writing it to a DATE field:

```ts
b.parseDate({ inputVariableName: "completed_date_raw" },
  { key: "<uuid>", as: "completed_date_parsed" });
```

Other useful Order fields: `orderType` (references a team-configured order type
from a service line), `displayName` (overrides the order title in the UI),
`tags` (free-form list, for filtering/segmentation), `isResupply`.

### Referring provider (for the #12 provider backfill)

`referringPractitioner` is an ordinary `fieldPath` whose **value is an entity
selector object**, not an `entityReferences` slot. `referringPractitionerNpi` is
a plain string alongside it:

```ts
b.updateTennrObject(
  {
    updates: [
      { value: { externalId: "{{provider_lookup.id}}" }, fieldPath: "referringPractitioner" },
      { value: '{{provider_lookup.npi,""}}',             fieldPath: "referringPractitionerNpi" },
    ],
    variableType: "ORDER",
    variableReference: "order_entity",
  },
  { key: "<uuid>", name: "Set Order Referring Practitioner" },
);
```

Two traps:

- **`externalId` is the practitioner's external id — usually a different column
  from the one you matched on.** If you resolved the provider through a lookup
  table, the match key (an EHR contact number) and the external id (often the
  table's `id` column) are different values. Writing the match key resolves
  nothing, silently.
- Only write it when the order has none — read `TEAM_PRACTITIONER` by
  `ORDER_REFERENCE` first and skip if it returns one. Never overwrite a provider
  a human set.

`providerLinking` is **not** an alternative: it is document-driven and
Validate-backed, so it cannot run headless in a CRON.

**When a TOM field looks undiscoverable from the SDK, it usually isn't.** The
SDK does not enumerate ORDER field paths — grep a **prod-pulled** workflow in
the same org that already writes it. The editor can set the field, so some
shipped workflow does too.

### Order notes (append-only — safe to create without reading first)

```ts
b.createTennrObject(
  tom.orderNote(
    { note: "{{note_text}}" },
    {
      dependencies: {
        entityType: "ORDER_NOTE",
        orderReference: { entityType: "ORDER", inputVariableName: "order_entity" },
      },
    },
  ),
  { key: "<uuid>", as: "created_date_note", name: "Post Date Note to Order" },
);
```

Optional `externalId` on the note de-duplicates against the source system — use
it when you're mirroring EHR notes so a re-run doesn't duplicate them.

### Rejection Info (only if the ticket names rejected EHR states)

Fields: `reason` (**required**) + `notes` (optional string list). Always paired
with `status: "Rejected"`. **Never set status `Rejected` without a reason.**

Most CRON builds are Scheduled + Complete only. Do **not** invent a rejection
path the ticket did not ask for.

**Create the entity first, then set the status — in the same `tryCatch`.** If
the entity write fails, the status is never changed and the order keeps its
previous state; you never strand one as `Rejected` with no reason. Two separate
`tryCatch` blocks defeat this: the first swallows the entity error and the
status is written anyway. This is the opposite of the note/date ordering below,
and deliberately so — a note is decoration, a rejection reason is required.

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

### Reading orders

**One order, by its EHR id** — the workhorse:

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

**Every open order** — returns a list variable. This can take many minutes:

```ts
w.readTomEntity(
  {
    entitySearch: {
      entityType: "ORDER",
      searchType: EntitySearchType.ORDER_STATUS,
      orderStatuses: ["On Track"],   // any of the four enum values
    },
    errorOnNoMatch: false,
  },
  { key: "<uuid>", as: "open_orders" },
);
```

**The patient behind an order**, and **the order's line items**:

```ts
searchType: EntitySearchType.ORDER_REFERENCE,
orderReference: { entityType: "ORDER", inputVariableName: "current_order" },
// entityType: "PATIENT_ENTITY" or "ORDER_ITEM"
```

### Hold one handle per entity

`updateTennrObject` takes a `variableReference` to an existing handle. Read the
order once into `order_entity` and update *that* reference throughout. Don't
re-read the same order into new variables, and don't fork
`tom_status`/`tom_status_1` copies of entity-derived fields.

### There is no CRON root block type

Scheduled workflows use the **EMAIL** root and are scheduled app-side. Give the
workflow both roots so a human can also trigger it manually:

```ts
w.root(WorkflowBlockType.MANUAL, (b) => {
  b.fillForm({}, { key: "<uuid>", as: "input_list", name: "Start: Manual Trigger" });
});
w.root(WorkflowBlockType.EMAIL, (b) => {
  b.receiveEmail({}, { key: "<uuid>", name: "Start: Scheduled / Email Trigger" });
});
```

---

## Choosing the sync architecture

**Ticket #12 is the source of truth** for how to loop. Also use #5
(credentials), #11 (how to read status), and any API docs attached to the
ticket. **Pull and read the named reference build before writing any code**, and
copy its *shape*, never its customer data — every reference is a different org,
and `reference-index.md` lists per-pattern what to copy and what not to. When
you cite a reference in your report, say which part you took and why it
generalizes.

| If #12 says | Build | Read first |
| - | - | - |
| Credential-constrained EHR, per-WIP-state fetch is possible, thousands of open orders | **A — dispatcher + batch worker.** Loop the WIP lists, one EHR call per WIP state, chunk the resulting order IDs into batches of 50, spawn a worker per batch. O(1) EHR calls, O(n) TOM ops. | `orgs/williams-brothers/workflows/invisible/cron-wip-stage-sync/` + `.../wip-stage-sync-worker/` |
| Customer has a report, DB, or SOR table that's better than the EHR modules | **B — report/table-driven dispatcher + branch worker.** Query the table, normalize + chunk in a `codeBlock`, spawn a branch worker per batch that maps each row to stage/status/notes. | `orgs/viemed-sleep/workflows/invisible/tom-cron-branch/` (cleanest mapping) and `orgs/flomed/workflows/invisible/patient-pipeline-status-batch/` |
| Low order volume, cheap EHR API, no credential pressure — or #12 says loop TOM orders and ping the EMR per order | **C — single workflow.** `readTomEntity` by `ORDER_STATUS`, `forEach` the orders, branch on `current_order.stage`, read the EHR per order. Simplest; start here if unsure. | `orgs/livwell/workflows/in-dev/treatments-scheduled-and-received-sync-cron/` |
| The EHR's only order endpoint is keyed by **patient**, not by order or WIP state — no bulk "what changed" call exists | **D — TOM-first, patient-keyed.** `ORDER_STATUS` sweep, resolve each order's patient, group **by patient**, chunk, spawn a worker that makes one API call per patient and reconciles the response against that patient's open order ids. | `orgs/tactile/workflows/cron-referral-center-order-sync/` + `.../referral-center-order-sync-worker/` |

If #12 is blank, ambiguous, or you cannot tell whether bulk-fetch would miss
archived/completed/voided orders, **stop and ask**. Do not pick a pattern by
guessing.

### EHR-specific guidance

**Prefer the ticket.** Use the notes below only when #11 / #12 name that EHR
and leave a gap the ESE did not fill.

**Brightree — PAPI (credential-constrained).** Looping every TOM patient and
checking Brightree is too costly. Two options:

- **Sales Orders Worklist — not recommended.** It excludes orders marked
  "Completed" in Brightree, so if the customer marks orders that way (most do)
  it misses exactly the orders you are trying to update. Only use it if you have
  confirmed that doesn't apply.
- **Ad-Hoc Audit Report — recommended.** Check whether the ESE specified audit
  reports for you to use — WIP transitions, voided orders, completed sales
  orders, etc. One report call replaces the many per-order Brightree requests
  that hog credentials. Example — a WIP-transition report is built in Brightree:
  Ad-Hoc Reports → "Design A New Report" → "Audit Trail (Last 12 Months)" →
  Select All → skip the next screen → filter:
  - `Audit_Audit Type` **Equals** `Sales Order`
  - `Audit_Audit Detail` **Begins With** `WIP State changed`
  - `Audit_Audit Date` **In Time Period** `Yesterday`
  - plus any exclusions (e.g. `Changed By_Login Name` **Doesn't Equal** the SNAP
    API login)

  Pull it with the `REQUEST_REPORT` PAPI. The Audit Date column carries a
  timestamp, so a `codeBlock` can window to only changes since your last sync.

**Brightree — SOAP.** Fast enough in theory to query every TOM order, but that
costs the customer real money at volume. Still default to pulling only WIP
states that changed since the last sync. **Verify the API response includes
orders in a Completed status** — if it doesn't, fall back to the ad-hoc report.

For reading order details, `ehr: "Brightree"` with `type: "GET_ORDER"` returns
the **full** order object including `scheduledDate` / `scheduledTime` /
`actualDate` / `actualTime`. The `"Brightree V2"` variant returns a minimal
object **without** those Order-tab fields.

**WeInfuse.** Check each TOM order against WeInfuse rather than pulling all flow
tasks — stale flowtasks confuse the latter, and TOM orders slip through and
never get updated. Handle **archived orders** explicitly, or they stick
permanently at `On Track`:

- Archived with treatment complete → `Completed` + the matching appointment date.
- Archived with a clear rejection reason (OON, patient went elsewhere) →
  `Rejected` + the reason.
- Archived for any other reason → default to `Completed` with a note saying the
  order was archived and why.

For `Scheduled`, pull the order's appointments and put the appointment date on
as a note.

---

## Build rules and gotchas

Every one of these is a real failure observed in shipped code. Follow them.

**Never compare an EHR status string literally — normalize it.** Verbatim
strings from a customer routinely contain a Unicode en dash (`\u2013`), a
non-breaking space, or doubled spacing. `status === "Shipped – Approved"`
compiles, runs, throws nothing, and matches **zero** rows forever — the worst
failure mode a sync has, because it looks like "nothing shipped yet". Fold dash
variants to ASCII, collapse whitespace, lowercase, then compare:

```ts
function normalizeStatus(v: unknown): string {
  return String(v ?? "")
    .replace(/[\u2010-\u2015\u2212\uFE58\uFE63\uFF0D]/g, "-")
    .replace(/\s+/g, " ").trim().toLowerCase();
}
```

Keep the *normalization*, not the literal, as the contract. Applies to any
verbatim string from **#6** / **#7**.

**Absence is not completion.** An order the EHR did not return means "we learned
nothing", never "it finished". Count those separately and write nothing. Only an
explicit terminal status completes an order.

**Carry the target stage explicitly on each item.** Emit `{ id, stage }` per
order, where `stage` is set by *which bucket produced it* (Scheduled vs
Complete). Never infer the stage by pattern-matching a WIP name — arbitrary new
WIP names (`DELIVERY`, `POD - Delivery`, `RTC-*`) then route correctly for free.

**Emit batches as real arrays, not JSON strings.** A string element gets
re-encoded (quoted) when referenced in a template, so the worker's `parseArray`
sees a string instead of an array. Return `JSON.stringify(batches)` where
`batches` is `T[][]`, not `T[]` of strings. The receiving worker should *still*
carry the unwrap guard, because `{{batch}}` can double-encode:

```ts
let parsed: any = [];
try { parsed = JSON.parse(input.chunk_str ?? "[]"); } catch { parsed = []; }
if (typeof parsed === "string") {          // double-encoded — unwrap once more
  try { parsed = JSON.parse(parsed); } catch { parsed = []; }
}
if (!Array.isArray(parsed)) parsed = [];
```

**Coerce `confirmInput` variables before a `codeBlock`.** A variable arriving
with `inputType: "OTHER"` has an ambiguous multi-type that a `codeBlock` input
rejects. Pass it through a `formatText` first to get a clean STRING.

**`forEach` needs `shouldThrowErrors: true`** (lint requirement) — so put a
per-item `tryCatch` *inside* the loop so one bad order ID doesn't sink the whole
batch. Do the core stage/status update **first**, then wrap best-effort extras
(the date fetch, the note) in their own nested `tryCatch` so a failure there
can't undo the update that matters. This ordering applies to **optional
decoration only**. A required entity — `tom.rejectionInfo`, `tom.missingInfo` —
goes *before* the status write and shares its `tryCatch`; see "Rejection Info".

**Isolate each EHR fetch in a `tryCatch`.** An invalid or renamed WIP state is
rejected by the EHR outright; without isolation it aborts the entire run instead
of skipping one WIP state.

**Credentials: minimum scope, and beware the pause point.** Keep only the steps
that need credentials inside `withCredential`, and put login in
`retryable({ retries: 5 })`. **An inline credential block is a pause/re-enqueue
point that defeats a concurrency-of-1 setting** — this produced 108+ duplicate
cards in production, twice, at the same customer. If the sync needs a true
single-run-at-a-time guarantee, use worker-level credential configuration
instead.

**Chunk and cap fan-out.** 50–300 items per batch. Never spawn per-order runs
unbounded — every documented concurrency incident traces to unbounded fan-out.

**`pinnedMajorVersion` must match the deployed child's major**, not whatever the
repo file last said. Check before pushing; a stale pin spawns a nonexistent or
old version.

**Set `card_name`** so the run is legible in the queue:

```ts
variableOps.createText("card_name",
  'WIP Stage Sync — {{parsed_orders.length,"0"}} order(s)')
```

**Add a testing-only stop before the spawn** so you can exercise the dispatcher
without fanning out:

```ts
b.raw(StepType.WORKFLOW_END, {},
  { key: "<uuid>", isTestingOnly: true, name: "STOP (Test Runs Only): Skip Spawn" });
```

**Guard the dispatch on a non-empty batch list** — `cond.var("parsed_batches.length").gt("0")`.

**Format Text in JSON mode silently drops empty/null fields.** They're omitted
from the output entirely, not written as `""`. The failure surfaces later, at
whatever consumes the JSON. Default a value with `{{var,"fallback"}}` rather
than assuming it passes through.

**Verify every push actually landed.** `tennr push` can fail with a server error
(e.g. `canceling statement due to statement timeout`) while printing enough
output to look successful. Confirm with
`tennr check <file> --branch <slug> --team-id <id> --refresh-baseline` and
require `documents: equal` before you run anything. Without `--refresh-baseline`
you compare against a stale pinned baseline, not the branch.

**Question headers you inherit.** When copying an integration call from another
worker, verify each header is actually required. Auth fragments get cargo-culted
between workers — a bearer minted for one system routinely ends up on calls to a
different one that never needed it. Test by removing it.

---

## Phases

Work them in order. Report at each boundary; don't batch up surprises.

### Phase 0 — Verify the input before building

Read the **Linear ticket body** in full (that is the ESE input). Resolve the
org folder from **#1** (`ls orgs | grep -i …`) — see Safety rule 6. Pick your
reference build from `reference-index.md` and pull it before reading it, e.g.
the Williams Brothers dispatcher (keep in mind it is Brightree-specific):

```bash
tennr pull prod --assistant-id 6a554b8389b7cfba3b8d5b7f --team-id <teamId> \
  --out orgs/williams-brothers/workflows/invisible/cron-wip-stage-sync \
  --slug williams-brothers
```

Its batch worker is assistantId `6a554b748a58b9cb80d4ccd0` at
`orgs/williams-brothers/workflows/invisible/wip-stage-sync-worker/` — pull it
too before copying. Do not read the checked-in `.ts` as if it were live.

1. Restate the Scheduled and Complete WIP mappings (**#6**, **#7**) and the
   date fields (**#8**, **#9**) back to the ESE for confirmation — everything
   else rests on these. If **#6** says there is no Scheduled concept, confirm
   you will skip Scheduled entirely.
2. **Stop and escalate** if the mapping is conditional or varying (by SKU,
   several WIP states, edge cases). Do not guess a rule.
3. Confirm every stage name in **#10** exists in Referral Pipeline settings,
   character for character. Stages marked `(scheduled)` and `(completed)` are
   the ones this CRON writes. **If #10 has no terminal stage, ask for one to be
   added; if it can't be added now, take the empty-`completed_stage` flag
   path** — see "The target pipeline shape".
4. **Audit who else writes order status.** Grep the org's *prod-pulled* workers
   for `fieldPath: "status"` and `tom.order({ status })`. If any worker sets
   `Completed` before the terminal event — a Qual worker doing it on a passing
   decision is the usual case — then:
   - your sweep **must** include `Completed`, or those orders are invisible to
     the sync and can never receive a date, note, or terminal stage; and
   - you **must** gate re-processing on `stage`, not `status`, so a daily run
     skips orders already at the terminal stage instead of rewriting them and
     duplicating notes.

   Report the conflicting worker. Fixing it belongs to
   `wac-prompt-missing-info.md`; do not silently work around it, and do not edit
   that worker from this prompt.
5. Confirm the credential (**#5**), how status is read (**#11**), and any SOR
   table or attached API docs resolve in the target org: `tennr team list …`,
   `tennr team describe module`, `tennr schema ehr`.
6. Restate the looping plan from **#12** — per-order vs bulk, what might be
   missed, volume, cadence, and any provider backfill. If #12 is thin, stop
   and ask before picking an architecture.
7. Create the draft branch and add the workflows (see Safety rule 2). Team ID
   comes from the workflow `.ts` metadata, not the ticket.

### Phase 1 — CRON sync (the main build)

Build per #12 and the selected pattern. Requirements:

- Both `MANUAL` and `EMAIL` roots.
- Explicit buckets from the ticket: Complete from **#7**, and Scheduled from
  **#6** unless the ticket says to ignore Scheduled. Verbatim WIP strings.
- Stage **and** status advance together. Derive status from the target stage: a
  Complete-stage order is `Completed`; a Scheduled-stage order (set up for care,
  not finished) is `On Track` — never left as `Missing Info`.
- Dates written to `dateFulfilled` where **#9** provides a field, and as a note
  either way. Scheduled dates from **#8** go on the note (and only on a date
  field if the ticket says so).
- If — and only if — the ticket noted rejected/cancelled/archived EHR states,
  map them to `Rejected` + a `tom.rejectionInfo` reason. If you believe one is
  needed and the ticket is silent, ask.
- If **#12** specified any backfill (e.g. provider populated in the EMR later),
  follow those instructions when you pull order info. Only fill a field that is
  currently empty; never overwrite one already set.
- If the ticket asks for a **QA audit worker**, clone the shape of
  `orgs/williams-brothers/workflows/invisible/wip-stage-sync-audit/`
  (assistantId `6a621cbf3fc6676a7d39e1d3`) after pulling prod. Report-only —
  it never writes TOM. Do not build one the ticket did not ask for.
- **Idempotent by construction.** This runs daily against the same orders, so a
  second run must be a no-op: skip orders already at the terminal stage before
  they cost an EHR call. Gate on `stage`, not `status` — if Phase 0 found a
  worker that sets `Completed` early, status alone cannot tell "done" from
  "qualified".

**Proof required:** real run IDs showing a Completed transition (and a
Scheduled transition if #6 applies), with notes and dates present. Ask the ESE
for one real order per case plus a control that must not change, if they did
not already name them on the ticket.

**Then tell the ESE, explicitly, what you cannot do app-side:** scheduling the
CRON cadence (from **#12**; twice a day ideally, once a day minimum), and adding
a Completed pipeline stage if #10 had none. The build is not live until cadence
is set.

**Then run `wac-prompt-missing-info.md` against this same ticket** (or state
that it is the next session). The Qual/E&B status writes are still owed.

### Phase 2 — Self-review

Verify each item from the CLI (`tennr run list` / `run get` / `run logs`), not
by eyeballing the app. Report pass/fail per line.

- [ ] Orders that should be complete are `Completed` by the CRON, with their
  note/date present.
- [ ] Orders that should be scheduled are advanced by the CRON, with their
  note/date present — **or N/A if #6 says ignore Scheduled**.
- [ ] The control order is unchanged.
- [ ] Every order the CRON set to `Rejected` (if any) has a rejection reason.
- [ ] Bulk fetch (if used) cannot miss archived/completed/voided orders, per
  **#12**. No 7-month-old order sitting `On Track` in Referral Received →
  feeds Phase 3.
- [ ] **Re-running the sync immediately is a no-op** on the orders it just
  completed — no second write, no duplicate note. Prove it with a second run ID,
  not by reasoning.
- [ ] Any EHR state that is terminal-but-not-complete (returned, cancelled,
  voided) has an agreed destination. If the ticket is silent, these sit open
  forever — surface the count and ask; do not invent a rule.

Anything failing goes to Ben Howe, not into a silent workaround.

### Phase 3 — Stale-order backfill (gated — approval required)

Only after Phase 1 is proven on real runs.

1. Enumerate stale orders: `readTomEntity` by `ORDER_STATUS: ["On Track"]`,
   filter by age.
2. Classify them — genuinely still open, silently completed, abandoned,
   duplicates.
3. **Stop.** Present the count, the breakdown, and the proposed disposition rule.
   **Wait for explicit human approval.**
4. Only then run a one-off sweep, on the draft branch, and report the run IDs.

Never auto-run this. Never skip the approval, even if **#12** stated a
disposition — a stated preference is not approval of a specific set of orders.

---

## Reporting

At each phase boundary, report a table:

| Case | Input | Expected | Actual | Run ID |
| - | - | - | - | - |

Plus, explicitly: what you did **not** do and why. If something is blocked,
finish everything else and say precisely what's left. Do not describe a phase as
complete on the strength of a typecheck.
