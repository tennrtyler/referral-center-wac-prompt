# Referral Center implementation validator

A single prompt. Input: one customer. Output: `COMPLETE` or `INCOMPLETE` plus

the exact pieces to fix. Run it against a customer whose Referral Center

implementation is marked done in the rollout tracker, before or after the

review meeting with Ben Howe.

The contract it grades against is the

[REFERRAL CENTER - Master Deployment Plan](https://app.notion.com/p/3b4eb680c7fc80e5a420d694be4c4d4e)

(the QA gate every ESE self-review uses), tightened by the two Referral Center

WAC prompts (`wac-prompt-cron.md`, `wac-prompt-missing-info.md`, Linear

`BEN-1643`) and the Referral Pipeline best-practice spec. Where those sources

disagree, the Master Deployment Plan wins because it is what the approver

reviews against.

---

## The prompt

Copy everything between the rules into a session in `tennr-workflows`.

---

You are validating whether **{{CUSTOMER}}**'s Referral Center implementation is

actually complete. You read; you never write. You never start a live run,

never push, never edit org configuration, and never commit. Every finding must

cite the query, command, run id, or file it came from. Output aggregates only;

never print a patient name, DOB, or address, even if a query returns one.

**Unit under validation.** A Referral Center implementation is one thing: the

customer's Referral Center order-status sync worker (one workflow, or a

dispatcher plus batch worker), the Missing Info worker the team has selected,

and the TOM order data those two produce. You are not auditing the org's other

workers. Read source for those two only. Everything upstream (intake, E&B,

Qual) is judged through the data it left on the orders, not by pulling it.

### 0. Resolve the customer and confirm scope

1. **Identity.** Resolve the org folder under `orgs/` by folder name (not display name). Take `teamId` from
   `orgs/<org>/org.config.ts`; if there is no folder, take it from the "Accessible teams" list in `tennr whoami` and say
   the org is not in the repo. Read `orgs/<org>/CONTEXT.md`.
1. **Locate the sync worker.** Look, in this order, until you have its assistant id: the org's `workflows/` folder names
   (`*referral-center*`, `*sync*`, `*cron*`, `*pipeline*`, `*wip*`) and their `learnings.md`; the customer's Referral
   Center Linear project or the copied `BEN-1643` ticket; the rollout tracker notes; and only then a name match over
   `tennr workflow list --team-id <team> --json` (`workflowName` contains CRON, sync, stage, status, WIP, pipeline, or
   referral center). Record the assistant id(s) and stop looking at other workers. If nothing matches, that is itself
   the finding (see C3).
1. **Rollout status.** Read the customer's row in the
   [RC Waves rollout tracker](https://docs.google.com/spreadsheets/d/1N30QvuCR9sdifI4uS_1LIWQENWc7M3KZfn7nOzsLW1E) via
   the Google Drive connector: `RC Wave`, `Implementation Done?`, `Ben QC Done?`,
   `Pipeline Ready for Live Customer Demo`, `Responsible ESE`, `Other Notes`. Record what the tracker _claims_ so the
   verdict can be compared against it.
1. **Which surface.** Referral Center (external referring-provider portal), Referral Pipeline (sales reps), and Patient
   Hub (ops) all read the same TOM orders. Sections 1 through 3 below validate that shared TOM layer and apply to every
   Referral Center rollout. Section 4 (segmentation) applies only if Referral Pipeline has been rolled out to this
   customer; check the tracker and CONTEXT.md, and if unclear say so and still run it as informational.
1. **Pre-gate.** The Master Deployment Plan requires two things before the checklist even applies: the customer creates
   orders in the traditional sense, and Qualifications is live. Evidence, from data only:
   - production, non-archived rows in `TENNR_CORE_OBJECT_ORDER` for the team (query Q1 below) are non-trivial and
     recently updated;
   - `MISSING_INFO` or `REJECTION_INFO` rows for the team's orders whose `ORIGIN_STEP_TYPE` is a qualification step, or
     CONTEXT.md / the tracker recording Qual as live.

   If either fails, stop the checklist, grade `INCOMPLETE — pre-gate`, and say the plan routes this customer to a
   planning meeting with Ben Howe instead.

### 1. Configuration reads

Use Sigma (MCP `sigma`, connection `d5ee5fdf-9772-4383-ad6b-e41531cdf6be`,

Snowflake `ESTUARY.POSTGRES.*`, an Estuary replica of prod Postgres that lags

by seconds; confirm with `MAX(FLOW_PUBLISHED_AT)`). Call `begin_session` first.

Reference tables by element id in the `FROM` clause. If Sigma is unavailable,

the same tables exist in the Postgres read replica as

`tennr_core_object.<name>` through `scripts/metabase/mb.py --db 3`.

| Table                                                             | Element id                             |
| ----------------------------------------------------------------- | -------------------------------------- |
| `TENNR_CORE_OBJECT_ORDER`                                         | `f98b8d4d-8f48-4a04-9ccf-46e07d454aac` |
| `TENNR_CORE_OBJECT_ORDER_STAGE`                                   | `7538eccf-91c2-4f61-ad93-4960ccacb558` |
| `TENNR_CORE_OBJECT_MISSING_INFO`                                  | `1b9c5fc5-33c8-482f-a4f6-0823445f4627` |
| `TENNR_CORE_OBJECT_REJECTION_INFO`                                | `bf194d17-fbc1-4ad0-bed6-550996811a11` |
| `TENNR_CORE_OBJECT_ORDER_NOTE`                                    | `9b2a3781-a4e4-4e31-9328-c3ded82438d2` |
| `TENNR_CORE_OBJECT_DOCUMENT`                                      | `94837b89-8816-4d9a-bd3e-f452c5e1cef9` |
| `TENNR_CORE_OBJECT_TENNR_TEAM_TO_REFERRAL_WORKERS_JOIN`           | `501989c2-5136-4e4f-ae0b-a2870d167f0f` |
| `TENNR_CORE_OBJECT_ORDER_ACCESS_GROUP`                            | `5ea2971a-0e69-4572-911d-0a13a8832f06` |
| `TENNR_CORE_OBJECT_ACCESS_GROUP_TO_ORDER`                         | `5e542e85-fa09-4ec3-8638-e0ce4a3178f6` |
| `TENNR_CORE_OBJECT_ACCESS_GROUP_TO_MARKETING_REP`                 | `208c5d1b-6ac5-447e-a24d-ed020ea344d9` |
| `TENNR_CORE_OBJECT_TEAM_FACILITY`                                 | `b1a8449c-952b-400c-a49e-85a039bd218a` |
| `TENNR_CORE_OBJECT_TEAM_FACILITY_PRACTITIONER_MARKETING_REP_JOIN` | `4b1ec710-157c-444c-b1f5-9066d7591748` |
| `PUBLIC_ORDER_RULE`                                               | `9adf00f9-3299-433f-8b54-ae4ca4148376` |

Always filter orders by `"RECEIVING_TEAM_ID" = '<teamId>'`,

`"ENVIRONMENT" = 'PRODUCTION'`, `"IS_ARCHIVED" = false` unless a check says

otherwise. Order `STATUS` is the closed enum `On Track | Missing Info |

Rejected | Completed`; `STAGE` is org-configured free text resolved through

`STAGE_ID`.

**C1. Stage vocabulary** (`ORDER_STAGE` where `TEAM_ID` = team, all envs).

Required: a first stage that plays the "Referral Received" role

(`SEQUENCE_NUMBER` 1); a terminal Completed stage (the highest sequence, name

free); a penultimate Scheduled / Approved / Pending-Delivery stage **if** the

customer's process has a scheduled step (the ESE input worksheet or

`learnings.md` says; if silent, report as "confirm with ESE" not as a miss).

Stages must exist in PRODUCTION, and DEVELOPMENT and STAGING must carry the

same names, because config does not promote across environments. Note the

`ORDER_STAGE` table can hold several rows per env with the same name and

distinct ids; that is real history, not replica duplication. Judge by

distinct names.

**C2. Missing Info worker is set.** `TENNR_TEAM_TO_REFERRAL_WORKERS_JOIN`

where `TEAM_ID` = team. `MISSING_INFO_WORKER_ID` must be non-null. Then:

```bash
tennr workflow settings get --team-id <team> --assistant-id <MISSING_INFO_WORKER_ID> --json
tennr pull prod --team-id <team> --assistant-id <id> --out /tmp/rc-validate/<org>/missing-info-worker.ts --slug missing-info-worker --yes
```

The worker must have a live version and its pulled source must start from a

`w.missingInfo()` root (the `MISSING_INFO` input step). A worker that starts

from `receiveEmail` is the pre-2026 pattern and fails this check. No row, a

null id, an archived assistant, or a non-`MISSING_INFO` root all mean the

"Provide" button cannot work, which is a blocker.

**C3. The CRON sync worker** (the assistant id(s) from step 0.2; the

dispatcher and its batch worker if it is a pair):

```bash
tennr workflow settings get --team-id <team> --assistant-id <aid> --json
tennr pull prod --team-id <team> --assistant-id <aid> --out /tmp/rc-validate/<org>/<slug>.ts --slug <slug> --yes
tennr run list --team-id <team> --assistant-id <aid> --environment PRODUCTION --limit 30
```

Required, from settings: `cron_enabled: true`, a `cron_config` that fires at

least daily (the plan says twice daily, credential permitting), and a

`live_version_id`. Required, from runs: at least one PRODUCTION run in the

last 3 days in a completed state, and no run of the pair in a FAILED state in

the last 7 days that was not superseded by a success. Required, from the

**pulled prod source** (never the repo copy; repo copies have been found ~1,000

lines behind):

- sweeps only non-terminal orders, and the "already done" gate is on **stage**, not status (`readTomEntity`
  `ORDER_STATUS` over `On Track`/`Missing Info` plus whichever statuses upstream workers write mid-stream, then a
  terminal-stage filter);
- the Completed write targets the terminal stage from C1 by exact string, held in one variable;
- EHR status strings are normalized (dash folding, whitespace, case) before comparison, never compared literally;
- an order absent from the EHR response is counted and left alone, never completed;
- a Completed write also sets `dateFulfilled` and posts a human-readable order note with the ship/delivery/appointment
  date; a Scheduled write posts the scheduled date;
- Rejected writes are paired with a `REJECTION_INFO` create in the same `tryCatch`, before the status write;
- EHR-specific fallbacks the plan names are handled: archived orders (WeInfuse), Completed WIPs missing from the Sales
  Orders Worklist (Brightree), unmatched records have a documented disposition;
- fan-out is bounded (batches, not per-order spawns) and `useDraftVersion` is not set on any spawn in the pushed
  version.

A customer with no sync worker at all is a blocker unless CONTEXT.md or the

tracker records a deliberate exemption (consignment or stock-and-bill models

with no completion event; InHealth is the known example). Say which.

**C4. Missing Info and Rejected writes upstream, judged from data.** Do not

pull the intake, E&B, or Qual workers. The plan requires that Qual's

"Missing Info" outcome creates a `MISSING_INFO` record on the order and sets

status `Missing Info`, and that "Not Qualified" (and E&B out-of-network, if

E&B is live) creates a `REJECTION_INFO` with a reason and sets `Rejected`.

Q2 and Q3 measure whether that is happening; `ORIGIN_STEP_TYPE` and

`INTERACTION_ID` on those rows tell you which step and run wrote them

(`tennr run get <interactionId>` names the worker) when you need to say who

owns a fix. Q1 catches the one upstream defect the sync cannot survive: a

status `Completed` written on a non-terminal stage (a qual-pass that writes

Completed hides the order from the sync forever). If Q1 shows it, name the

writing worker from `learnings.md` or CONTEXT.md, or say the ESE must identify

it; do not go read the org's workers to find it.

**C5. Order types.** `PUBLIC_ORDER_RULE` rows for the team should cover the

service lines the customer sells. Report the share of production orders with

`ORDER_RULE_ID` null and the count whose `DISPLAY_NAME` is the fallback

`No Product Specified`.

### 2. Data hygiene queries

Run each; report the numbers in a table; apply the thresholds in section 5.

**Q1. Status × stage matrix.**

```sql
SELECT s."STAGE", o."STATUS", COUNT(*) AS n, MIN(o."CREATED_AT") AS oldest, MAX(o."UPDATED_AT") AS last_update
FROM "connection"."f98b8d4d-8f48-4a04-9ccf-46e07d454aac" o
LEFT JOIN "connection"."7538eccf-91c2-4f61-ad93-4960ccacb558" s ON s."ID" = o."STAGE_ID"
WHERE o."RECEIVING_TEAM_ID" = '<team>' AND o."ENVIRONMENT" = 'PRODUCTION' AND o."IS_ARCHIVED" = false
GROUP BY 1,2 ORDER BY 3 DESC
```

Fail conditions: status `Completed` in any non-terminal stage; status

`Rejected` in the terminal stage; any status outside the four-value enum; a

stage name in the data that is not in C1's production list.

**Q2. Missing Info orders carry a legible open missing-info record.**

```sql
WITH o AS (SELECT * FROM "connection"."f98b8d4d-8f48-4a04-9ccf-46e07d454aac"
  WHERE "RECEIVING_TEAM_ID"='<team>' AND "ENVIRONMENT"='PRODUCTION' AND "IS_ARCHIVED"=false),
mi AS (SELECT "ORDER_ID", COUNT(*) AS n_open,
  SUM(CASE WHEN LENGTH(TRIM("MESSAGE"))<8 OR "MESSAGE" ~ '^[A-Z0-9_]{3,}$' THEN 1 ELSE 0 END) AS n_opaque
  FROM "connection"."1b9c5fc5-33c8-482f-a4f6-0823445f4627"
  WHERE "RESOLVED_AT" IS NULL AND "ENVIRONMENT"='PRODUCTION' GROUP BY 1)
SELECT CASE WHEN mi."ORDER_ID" IS NULL THEN 'Missing Info status, no open missing_info row'
            WHEN mi.n_opaque>0 THEN 'has row, message opaque or code-like'
            ELSE 'has legible open missing_info row' END AS bucket, COUNT(*) AS n
FROM o LEFT JOIN mi ON mi."ORDER_ID"=o."ID" WHERE o."STATUS"='Missing Info' GROUP BY 1
```

Also sample 10 `MESSAGE` values (they are not PHI) and judge legibility as a

referring provider would.

**Q3. Rejected orders carry a reason.** Same shape against

`REJECTION_INFO."REASON"`. Any Rejected order with no row or a blank/code

reason fails.

**Q4. Stale orders.** Non-terminal status, grouped by stage and age bucket

(`<30d`, `30-90d`, `90-180d`, `>180d` on `CREATED_AT`). The plan's example is

an order from 7 months ago still On Track in the first stage. Report the count

and share of open orders older than 90 days, and separately those in the first

stage.

**Q5. Completed orders are decorated.** For status `Completed`, group by

stage × (`DATE_FULFILLED` set?) × (has an undeleted `ORDER_NOTE`?). Completed

orders in the terminal stage should have both when the EHR supplies a date;

Completed orders outside the terminal stage feed Q1's failure.

**Q6. Referrer present.** Count orders where `REFERRING_PRACTITIONER_ID`,

`REFERRING_TEAM_PRACTITIONER_ID`, and `REFERRING_TEAM_FACILITY_ID` are all

null. The plan requires at least one referrer on every order unless the

approver has accepted the exception in writing; look for that acceptance in

`learnings.md` or CONTEXT.md before grading.

**Q7. Document filenames.** For documents on the team

(`DOCUMENT."TEAM_ID"`, production, not archived) created in the last 60 days,

count filenames that are opaque: match `^[0-9a-f-]{24,}`, `^untitled`,

`\.tmp$`, `^document\d*\.pdf$`, `^scan`, or a bare number. Sample 10 filenames

for judgement.

**Q8. Sync is moving orders.** For orders whose status is `Completed` and

whose stage is the terminal stage, report `MAX(STATUS_UPDATED_AT)` and the

count updated in the last 7 days. Cross-check against the run list from C3: a

CRON that runs daily but has moved nothing in a week is either correct (nothing

shipped) or silently matching nothing; read the latest run's step results with

`tennr run logs <interactionId>` and report what the reconcile step counted.

### 3. Environment parity

From C1 and from `ORDER_STAGE`/`ORDER_ACCESS_GROUP` grouped by `ENVIRONMENT`:

stage names, the Missing Info worker (the join table is not env-scoped; say

so), and access groups must exist in PRODUCTION, not only DEVELOPMENT. Report

any name present in DEV or STAGING but absent from PRODUCTION.

### 4. Segmentation (Referral Pipeline only)

Determine the active order-view-restriction mode by evidence, since the team

setting itself is not exposed:

- `ORDER_ACCESS_GROUP` rows for the team → custom rules (recommended). Require exactly one `IS_DEFAULT = true` group per
  environment, non-default groups with at least one rep in `ACCESS_GROUP_TO_MARKETING_REP`, and `ACCESS_GROUP_TO_ORDER`
  coverage for recent orders. The intake source must call `routeOrderToGroup` after order create/update.
- rows in `TEAM_FACILITY_PRACTITIONER_MARKETING_REP_JOIN` joined to the team's facilities → the deprecated 1:1 mapping.
  Flag as "deprecated; do not extend" and report whether both modes are populated (mixed state is a defect).
- neither → all users see all orders. Acceptable only if CONTEXT.md or the ESE confirms that was chosen.

Also grep the pulled intake source for `manageOrderAccessGroup` and legacy

`salesRepName` fields; if present and the customer is on custom rules driven by

facility or ZIP, flag as legacy artifacts to remove.

### 5. Grading

Severity:

- **Blocker**: pre-gate fails; no CRON sync worker without a recorded exemption; `cron_enabled` false in production or
  no successful production run in 7 days; Missing Info worker unset, archived, or not a `MISSING_INFO` root; terminal
  stage missing; any Rejected order without a reason; more than 5% of Missing Info orders lacking an open missing-info
  row; any `Completed` status written by a non-terminal stage in live source.
- **Major**: `Completed` status on non-terminal stages in data (from an old version) not yet backfilled; more than 10%
  of open orders older than 90 days in the first stage; more than 20% of orders lacking a referrer without a recorded
  exception; more than 20% of orders lacking an order type; EHR string comparison done literally; absence treated as
  completion; stage or group names present in DEV but not PROD; mixed segmentation modes; sync spawning per order
  unbounded; `useDraftVersion` set on a pushed spawn.
- **Minor**: notes or `dateFulfilled` missing on Completed terminal orders when the EHR has a date; opaque document
  filenames above 10%; `No Product Specified` display names; CRON cadence once daily when the plan asked for twice;
  legacy routing artifacts present but inert.

Verdict: `COMPLETE` only when there are zero blockers, zero majors, and the

manual residue in section 6 is listed for the ESE. Anything else is

`INCOMPLETE` with the fix list. Never soften a blocker because the tracker says

"Implementation Done"; the point of this validator is to compare the claim

against evidence.

### 6. Manual residue (always list, never grade)

These cannot be validated from WAC tooling. List them in every report as the

ESE's or approver's checklist:

- Click "Provide" on one Missing Info order in Development and confirm the Missing Info worker starts (needs a team API
  key under Settings → Developers; not readable from the CLI).
- Statsig feature flags for the team (Referral Center / Patient Pipeline).
- Filter load time under real volume; mobile upload success/failure toast; CSV export DOB timezone.
- Referral Center facility access: attempt a second-facility approval for a practitioner in a test environment and
  confirm it is blocked; confirm a "not my referral" path exists.
- Analytics definitions the customer asked for, and whether the underlying events are captured.
- Contract alignment: Referral Pipeline enabled only if it is sold.

### 7. Output format

1. One line: `{{CUSTOMER}} — COMPLETE` or `{{CUSTOMER}} — INCOMPLETE (N blockers, N majors, N minors)`, followed by what
   the tracker claimed.
1. Table `Check | Result | Evidence`, one row per C1–C5, Q1–Q8, parity, segmentation, with the query or command that
   produced it.
1. **Fix list**, ordered blockers → majors → minors. Each item: what is wrong, the number behind it, which workflow or
   setting owns it, the smallest change that fixes it, and who does it (ESE in app, WAC change on a named draft branch,
   customer decision, or Ben Howe).
1. **Manual residue** from section 6.
1. **What I did not do and why** (any check skipped for lack of access, data, or scope).

Do not fix anything. Do not push. When a check needs a customer decision

(Scheduled stage semantics, Not-Qualified → Rejected mapping, returned-device

disposition), say so instead of guessing.

---

## Tooling this prompt depends on

| Layer                                 | Tool                                                                                                                                                                                                                                                                                                    | What it proves                                                                                                                                                           |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| TOM data and Referral Pipeline config | Sigma MCP over Snowflake `ESTUARY.POSTGRES.TENNR_CORE_OBJECT_*` (fallback `scripts/metabase/mb.py`, schema `tennr_core_object`)                                                                                                                                                                         | stage/status matrix, missing-info and rejection records, notes, dates, referrers, order types, documents, access groups, rep assignments, the team's Missing Info worker |
| Assistant settings                    | `tennr workflow settings get`                                                                                                                                                                                                                                                                           | CRON enabled, cadence, live version                                                                                                                                      |
| Workflow behaviour                    | `tennr pull prod` + reading the source                                                                                                                                                                                                                                                                  | sweep logic, normalization, idempotency, entity pairing                                                                                                                  |
| Runtime proof                         | `tennr run list --environment PRODUCTION`, `tennr run logs`; `scripts/fetch-run-logs.py --only-outputs` and `scripts/run-tools/sweep.py --assistant <id> --env PRODUCTION` when the CLI's log endpoint returns empty on a read-heavy sync run (both want a browser `TENNR_BEARER`; ask the user for it) | the CRON actually ran and what it counted                                                                                                                                |
| Locating the worker                   | repo folder names and `learnings.md`, then a name match on `tennr workflow list`                                                                                                                                                                                                                        | the one sync worker to read; never an org-wide audit                                                                                                                     |
| Claims                                | Google Sheet rollout tracker, Notion Master Deployment Plan, org `CONTEXT.md` and `learnings.md`                                                                                                                                                                                                        | what the customer was promised and what the ESE recorded                                                                                                                 |

Not readable from any tool, so the prompt lists them as manual residue: the

team-level order-view-restriction setting itself, the team API key, Statsig

flags, UX/performance behaviour, Referral Center facility-access enforcement,

and whether the "Provide" button end-to-end path works.

Per the tooling ladder, prefer the CLI, then `scripts/`, then Sigma/Metabase.

`prod-data-mcp` is not required.
