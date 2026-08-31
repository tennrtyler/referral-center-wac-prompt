# Referral Center — Reference Builds

Shipped implementations of this exact sync, already in `~/dev/tennr-workflows`.
Read the relevant one **before** writing code. Copy the *shape*; never copy the
customer data (WIP names, stage names, module versions, credential names, table
names — all org-specific).

This index is maintained separately from the WAC prompt files so new builds can
be added without retouching them. Architecture selection is driven by Linear
ticket **#12** (CRON looping), plus **#5** (credentials) and **#11** (how to
read status). Copy the *shape*; never copy customer data from the ticket.

---

## Pattern A — Dispatcher + batch worker (WIP-list driven)

**Use when:** credential-constrained EHR (Brightree PAPI is the canonical case),
per-WIP-state fetching is possible, thousands of open orders.
**Cost profile:** O(1) EHR calls per WIP state, then O(n) TOM operations.

| File | Role |
| - | - |
| `orgs/williams-brothers/workflows/invisible/cron-wip-stage-sync/cron-wip-stage-sync.ts` | Dispatcher: two WIP buckets → per-WIP EHR fetch → accumulate → chunk to 50 → spawn a worker per batch |
| `orgs/williams-brothers/workflows/invisible/wip-stage-sync-worker/wip-stage-sync-worker.ts` | Worker: parse batch → per order, resolve TOM order → set stage+status → best-effort date note |

**Copy this:**
- The two explicit `createTextList` buckets (`scheduled_wips`, `complete_wips`)
  with `useCommaSeparatedValues: false` — commas inside WIP names are then safe.
- The `raw_groups` accumulator + `formatText` emitting
  `{"wip":"...","stage":"Scheduled","orders":{{get_sales_orders,"[]"}}}` per WIP.
- The chunker `codeBlock`: flatten groups, tag each order with the stage its
  bucket implies, de-dupe by order id, slice into batches of 50, return
  `JSON.stringify(batches)` as a real `T[][]`.
- The credential-block boundary: login (`retryable`, 5 retries) and the EHR
  fetches inside `withCredential`; combine/chunk/dispatch entirely outside it.
- The worker's stage → status `codeBlock`: `complete` → `Completed`, everything
  else → `On Track`.
- The nested `tryCatch` layering: core update first, date-note fetch second and
  isolated.
- The `isTestingOnly: true` `WORKFLOW_END` before the spawn.

**Do not copy:** the WIP state string lists (deeply Williams-Brothers-specific),
`pinnedMajorVersion: 2`, the Brightree module versions.

---

## Pattern B — Report / SOR-table driven

**Use when:** the customer has a report, database, or SOR table that's a better
source than the EHR modules — or the EHR is too resource-constrained to poll.

| File | Role |
| - | - |
| `orgs/viemed-sleep/workflows/invisible/tom-cron-branch/tom-cron-branch.ts` | **The cleanest status/stage mapping in the repo.** Read this first. |
| `orgs/flomed/workflows/invisible/patient-pipeline-status-batch/patient-pipeline-status-batch.ts` | Dispatcher: `queryTable` a SOR table → normalize + chunk (300/batch) → spawn per batch |
| `orgs/flomed/workflows/invisible/patient-pipeline-status-cron/patient-pipeline-status-cron.ts` | The WeInfuse per-order branch worker |
| `orgs/flomed/workflows/invisible/patient-pipeline-batch-scheduling/` + `.../patient-pipeline-status-scheduling/` | The scheduling half, split from the status half |

**Copy this, from `tom-cron-branch.ts`:**
- The `get_tom_values(status, has_bill, is_void)` mapping function: a **priority
  ladder** returning `{ tom_stage, tom_status }`. Billed wins, then void, then
  pre-void, then per-status mappings, then an empty-string default. This is the
  right way to express a many-EHR-states → few-Tennr-stages mapping.
- Emitting `""` for both stage and status when nothing matches, then guarding the
  write with `cond.and(var('tom_status,"").isNotEmpty(), var('tom_stage,"").isNotEmpty())`.
  A blank mapping writes nothing rather than clobbering a stage.
- The void → `tom.rejectionInfo` + reason-with-date path.
- The order-tag upsert `codeBlock` (`Order Status: <x>` replacing any existing
  `order status:` tag) — a good pattern for surfacing raw EHR status without
  abusing `stage`.
- Mirroring EHR notes with `tom.orderNote({ note, externalId })` inside a
  per-note `tryCatch` — `externalId` is what makes re-runs idempotent.

**Do not copy:** the Bonafide-specific status strings, the phone-number
validation helper, `manageOrderAccessGroup` / `routeOrderToGroup` (sales-rep
routing, unrelated to Referral Center), the SOR table names.

---

## Pattern C — Single workflow, loop TOM orders

**Use when:** low order volume, cheap EHR API, no credential pressure. Simplest
starting point when the right pattern isn't obvious.

| File | Role |
| - | - |
| `orgs/livwell/workflows/in-dev/treatments-scheduled-and-received-sync-cron/treatments-scheduled-and-received-sync-cron.ts` | Whole sync in one workflow: `ORDER_STATUS` read → `forEach` → branch on `current_order.stage` |

**Copy this:**
- `readTomEntity` with `EntitySearchType.ORDER_STATUS`, `orderStatuses: ["On Track"]`,
  `errorOnNoMatch: false` — the "every open order" entry point.
- Declaring the stage names as variables at the top
  (`handed_off_stage`, `scheduled_stage`, `treatment_received_stage`,
  `completed_status`) and comparing with `cond.var("current_order.stage").eq("{{scheduled_stage}}")`.
  Stage strings then appear exactly once each.
- The state-machine detour: one `d.case` per current stage, each advancing the
  order at most one step. Idempotent on re-run by construction.
- The session-reuse `defineFunction` (`Login and Store Session Data`): try a
  cheap authenticated call, and only log in if it errors.
- Writing `occurrenceDateTime` onto each `ORDER_ITEM` via an
  `ORDER_REFERENCE` read + `forEach`, after `parseDate`.

**Do not copy:** the WeInfuse appointment-matching `codeBlock`s (specific to
WeInfuse's `data[].attributes["order-id"]` shape), `is_prod` handling, the
livwell stage names.

---

## Missing Info worker

| File | Role |
| - | - |
| `orgs/flomed/workflows/invisible/patient-pipeline-missing-info/patient-pipeline-missing-info.ts` | The "Provide" button target. ~180 lines; copy nearly verbatim. |

**Copy this:** the `WorkflowBlockType.MISSING_INFO` root with
`rawExact(StepType.MISSING_INFO)`; the `confirmInput` mapping
`MISSING_INFO: externalId|notes|externalFiles` alongside `MANUAL: Order_id`;
`combineFiles` → `readTomEntity` by `EXTERNAL_ORDER_ID` → `tom.orderNote` →
`spawnWorkflow`. Note the defaulted note reads
`{{notes[0].message, "No known missing info"}}` — always default, because the
referring provider can submit with an empty note.

**Do not copy:** `importWorkflow("Fax Wrangler Extended")` /
`pinnedMajorVersion: 5` — resolve the customer's own target and its live major.

---

## Missing Info / Rejected writes to insert into existing workers

| File | What's there |
| - | - |
| `orgs/golden-tom/workflows/visible/qualifications-new/qualifications-new.ts` (~2073 MISSING INFORMATION, ~2222 NOT QUALIFIED) | The reference Qual pair: `updateTennrObject` status + `tom.missingInfo` / `tom.rejectionInfo`, inside `d.case` on `overall_decision`. `golden-tom` is the org's golden reference — prefer it. |
| `orgs/williams-brothers/workflows/visible/e-b/e-b.ts` (~3396) | The E&B missing-insurance pair: `status: "Missing Info"` + `tom.missingInfo({ message: "Missing insurance", resolvableBy: "ALL" })`. |
| `orgs/williams-brothers/workflows/visible/qualifications/qualifications.ts` (~2317) | Same pair with `message: "{{decision_note}}"` — carrying the qual note through as the legible message. |
| `orgs/viemed-sleep/workflows/invisible/tom-create-update/tom-create-update.ts` | An org that centralizes **all** TOM writes in one delegated workflow. Only relevant if the target org already works this way — see the one-posture rule below. |

---

## Repo conventions worth knowing

- **`docs/workflow-building-best-practices/06-tom.md`** is the repo's TOM
  charter. Two rules bear directly on this work: *read before you create*
  (avoid duplicates on re-run; notes are the exception, being append-only), and
  **one write posture per org** — inline `createTennrObject`/`updateTennrObject`
  is the default, but if the org delegates to a `TOM Create/Update` workflow
  (viemed-sleep), match that and don't mix.
- **Wrap a repeated read+branch+update in a `defineFunction`** rather than
  inlining it at every call site (`Patient_TOM` / `Order_TOM` in
  `orgs/viemed-respiratory/workflows/visible/patient-intake/`).
- **Every workflow folder carries a `learnings.md`.** Read it before you build;
  append to it before you finish. It's how the next session avoids relearning.
