# Referral Center — WAC Implementation Kit

The ESE fills the 13-question worksheet in `ese-input.md`. That filled
worksheet **is the Linear ticket body**. The three other markdown files are
**attached to the same ticket as Linear file embeds**
(`linear-embed` `node-type="file"`). WAC must download those embeds first —
`get_issue` often omits them.

| File | Who touches it | What it is |
| - | - | - |
| `ese-input.md` | **You, the ESE.** Fill it, paste into the ticket body. | Bootstrap + run order + 13 questions. |
| `wac-prompt-cron.md` | Attach to the ticket | Prompt 1 of 2: the EHR → TOM Scheduled / Completed sync. |
| `wac-prompt-missing-info.md` | Attach to the ticket | Prompt 2 of 2: Missing Info / Rejected / On Track writes into E&B + Qual, and the Provide-button worker per #13. |
| `reference-index.md` | Attach to the ticket | Shipped builds to copy *shape* from (after `tennr pull prod`). Not a prompt to run. |

**Run order matters.** The CRON prompt goes first: it is what eventually
completes an order. The missing-info prompt demotes a passing Qual decision to
`On Track`, which is only safe once the CRON exists to complete it. Running
only the CRON prompt and treating the Qual/E&B writes as out of scope is a
miss — both prompts run against the same ticket.

## Before you start — the gate

From the Master Deployment Plan:

1. **Does your customer create orders in the traditional sense** (most DME,
   drugs, etc.)? If **no** → message Ben Howe for a planning meeting. Stop here.
2. **Is Qualifications implemented?** If **no**, missing info may look different
   — check with Ben Howe before building.

If those are clear → fill in the worksheet.

## How to run it

1. Fill in `ese-input.md`. Questions **6, 7, 11, and 12** gate the sync.
   **#10** must list every stage — add a Completed stage in the app if one
   does not exist. **#13** is the Provide-button instruction.
2. Create a Linear ticket whose **description is the filled worksheet** (keep
   the WAC bootstrap / run-order block at the top).
3. Attach `wac-prompt-cron.md`, `wac-prompt-missing-info.md`, and
   `reference-index.md`. If #11 is an API, attach those docs too.
4. Open a WAC session in `~/dev/tennr-workflows` against that ticket. The agent
   downloads the three embeds, then runs both prompts in order.
5. Work the phases. The agent pulls prod before reading any existing worker,
   stops at Phase 0 to confirm mappings, and again before touching any
   historical order.

## What you still own

The agent builds and tests. It cannot:

- **Set the Missing Info workflow in Referral Pipeline settings** (app-side).
- **Schedule the CRON cadence** (app-side; twice a day, credential-permitting).
- **Add pipeline stages** — add a Completed stage yourself if #10 has no
  `(completed)` marker.
- **Approve the stale-order backfill.**

And the part no prompt replaces: **testing and validating the work is still
expected of you.** The agent reports run IDs; you read them.

## Escalate to Ben Howe when

- The Scheduled or Complete mapping is conditional or varies (by SKU, multiple
  WIP states, edge cases) — the prompt will stop rather than guess.
- Qualifications is not live and missing-info behavior is unclear.
- Any self-review item fails.
- Orders legitimately have no Referring Practitioner or Referring Facility.
- You're past a week on this.

## Source

Notion: *REFERRAL CENTER - Master Deployment Plan*. Rollout waves:
`https://docs.google.com/spreadsheets/d/1N30QvuCR9sdifI4uS_1LIWQENWc7M3KZfn7nOzsLW1E`

Note: Referral Center rolls out **independently of Referral Pipeline** in most
cases. Unless RP has been explicitly rolled out for your customer, direct them
to **Patient Hub** to view order statuses.
