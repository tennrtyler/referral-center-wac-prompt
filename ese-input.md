# Referral Center — ESE Input

## WAC bootstrap (read first)

This ticket attaches three markdown files as Linear file embeds
(`linear-embed` `node-type="file"`). `get_issue` often does **not** return them.

1. Review those 3 embeds in the issue description.
2. Download them locally before doing any other work:
   `wac-prompt-cron.md`, `wac-prompt-missing-info.md`, `reference-index.md`.
3. If you cannot find any of them, **stop and prompt the user**. Do not proceed
   from ticket-body text alone.

**Run both prompts, in this order, against this ticket, completing each
before starting the next.**
1. `wac-prompt-cron.md` — the EHR → TOM Scheduled / Completed sync. Build this
   first: it is what eventually completes an order.
2. `wac-prompt-missing-info.md` — insert Missing Info / Rejected / On Track
   writes into the Qual worker named in **#4** (and E&B if **#2** is yes).
   **Naming the Qual worker is an instruction to edit it.** This demotes a
   passing Qual decision to On Track, which is only safe once the CRON above
   exists to complete it.

`reference-index.md` is a shared reference read by both, not a prompt to run.

---

Fill in the 13 answers below. This worksheet **is the Linear ticket body**.
Answer every one (write `none` or `unknown` rather than leaving a line empty).

---

**1. Org WAC folder**

The folder name under `orgs/` in `~/dev/tennr-workflows` (kebab-case, not
the display name, e.g., `williams-brothers`). Confirm with `ls ~/dev/tennr-workflows/orgs | grep -i <name>` if you are unsure.

→

**2. Is E&B live?** (yes / no)

→

**3. Is Qualifications live?** (yes / no — if no, missing info may look different, and you should check with Ben Howe)

→

**4. Workflows to touch:** assistant name and assistant id, one per line (E&B and Qual)

Naming a Qual worker here instructs WAC to insert
Missing Info / Rejected / On Track status writes into that worker, so don't
list it if you do not want it edited.

*e.g. `qualifications` / `<assistantId>`*

→

→

**5. Credential name + type, exactly as it appears in the org**

*e.g. `Brightree Creds` / `BRIGHTREE`*

→

**6. "Scheduled" means these EHR WIP/Task states**

This will depend heavily on your customer, and is something you should check with them. In general, we want to capture here for the referring provider that everything has been set up ('scheduled') for the patient to get their treatment, but they haven't actually gotten it yet. For DME, this usually means the device has been shipped/is scheduled for delivery. For infusions, this usually means an appointment has been scheduled for the patient to get infused. Copy/paste exact strings verbatim, casing and punctuation included, one per line.

→

**7. "Complete" means these EHR WIP/Task states**

This will also depend on your customer and is something you should check with them. We want to capture here what 'Completed' means for the referring provider, not for Tennr. What state in the EMR signals that the patient got their care? The device was delivered; they showed up for the appointment; etc. Copy/paste exact strings verbatim, casing and punctuation included, one per line, e.g., `WIP - Device Delivered`

→

**8. Scheduled date — which EHR field?** (or `none`)

There is often a date associated with the scheduled state such as when the device was shipped or the appointment was made (e.g., `scheduledDate` + `scheduledTime`)

→

**9. Completed date — which EHR field?** (or `none`)

There is usually a date recorded when the patient gets their care. Put where to find that in the EMR here, or, if you think the timestamp when the status is updated will be sufficient, that works too (e.g. `actualDate` + `actualTime`)

→

**10. Referral Pipeline stages — list ALL of them, exactly as configured in the app**

One per line, marking `(default)`, `(e&b)`, `(qual)`, `(scheduled)`, `(completed)`.
WAC cannot fetch this, and cannot create stages in the app.

**If you do not have a terminal stage for completed orders, add one in
Referral Pipeline settings now** and mark it `(completed)`. Referring
providers need a Completed column.

*e.g. `Referral Received (default)`, `Processing Order`, `Qualifications (qual)`, `Delivery Scheduled (scheduled)`, `Delivery Completed (completed)`*

→

**11. How do we read order status out of the EHR?**

Module path(s) + version, OR report name, OR API endpoint.

If it's an API, attach the docs to the Linear ticket.

*e.g. `/ehr/brightree/papi/get_sales_orders_with_filters` v1.1.0*

→

**12. Instructions for CRON Looping**

THIS IS THE MOST CRITICAL THING TO THINK THROUGH.

Based on the integrations you have to fetch and sync statuses, should you loop through all TOM orders and ping the EMR for each to check status?

If credential time is a limited resource, is there a way you can bulk fetch statuses and simply loop over them in Tennr to update TOM?

If you do fetch in bulk, are there any you might miss (e.g., archived/completed/voided orders)? Nothing should slip through the cracks — we don't want 7mo old orders sitting in On Track.

Think through how many orders you might be looping over. If it's going to be a ton, do you have any direction on how you'd like this to be done?

How frequently can you get the CRON to run (ideally twice a day but once a day at minimum is recommended).

If provider is not always populated during order creation but they add it into the EMR later, can you backfill using this CRON worker when you pull order info? 

→

**13. When missing info is provided (the "Provide" button)**

The uploaded documents/text must trigger a workflow. The easiest (and
acceptable) thing is to set the Order Status back to `On Track` and then
trigger Fax Wrangler. You may want something more fancy (trigger Qual, a
custom intake, etc.).

Write explicit instructions for the agent below, including whether it should create a new workflow or modify an existing one, or specify that you will do it yourself and the agent should skip this part. If you want to implement the base case described above, specify exactly: `WAC, implement the default Missing Info worker` and the agent will take care of it.

→
