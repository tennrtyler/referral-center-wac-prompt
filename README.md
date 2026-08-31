Implement Referral Center for {{org}} using the WAC prompts below.

Run them in this order, completing each before starting the next:

1. wac-prompt-cron.md — the EHR → TOM Scheduled/Completed sync. Build this first: it is what eventually completes an
   order.
2. wac-prompt-missing-info.md — Missing Info / Rejected writes into the existing E&B and Qualifications workers. This
   demotes a passing Qual decision to On Track, which is only safe once the CRON above exists to complete it.

reference-index.md is a shared reference read by both, not a prompt to run.

{{attach all three files here}}

Referral Center - ESE Input

Fill in the 12 answers below, then pass this ticket into WAC and tell it to go to town.

Answer every one (write none or unknown rather than leaving a line empty).

1. Org name

e.g. williams-brothers

→

2. Is E&B live? (yes / no)

→

3. Is Qualifications live? (yes / no - if no, missing info may look different, and you should check with Ben Howe)

→

4. Workflows to touch: assistant name or assistant id, one per line (E&B and Qual)

e.g. qualifications or <assistantId>

→

5. Credential name + type, exactly as it appears in the org

e.g. Brightree Creds / BRIGHTREE

→

6. "Scheduled" means these EHR WIP/Task states

This will depend heavily on your customer, and is something you should check with them. In general, we want to capture
here for the referring provider that everything has been set up ('scheduled') for the patient to get their treatment,
but they haven't actually gotten it yet. For DME, this usually means the device has been shipped/is scheduled for
delivery. For infusions, this usually means an appointment has been scheduled for the patient to get infused. Copy/paste
exact strings verbatim, casing and punctuation included, one per line.

→

7. "Complete" means these EHR WIP/Task states

This will also depend on your customer and is something you should check with them. We want to capture here what
'Completed' means for the referring provider, not for Tennr. What state in the EMR signals that the patient got their
care? The device was delivered; they showed up for the appointment; etc. Copy/paste exact strings verbatim, casing and
punctuation included, one per line, e.g., WIP - Device Delivered)

→

8. Scheduled date - which EHR field? (or none)

There is often a date associated with the scheduled state such as when the device was shipped or the appointment was
made (e.g., scheduledDate + scheduledTime)

→

9. Completed date - which EHR field? (or none)

There is usually a date recorded when the patient gets their care. Put where to find that in the EMR here, or, if you
think the timestamp when the status is updated will be sufficient, that works too (e.g. actualDate + actualTime)

→

10. Referral Pipeline stages, exactly as configured in the app

One per line, marking (default), (e&b), (qual), (scheduled), (completed). Or write none configured yet. WAC is not able
to fetch this information so it needs to be pasted manually.

e.g. Referral Received (default), Processing Order, Qualifications (qual), Delivery Scheduled (scheduled), Delivery
Completed (completed)

→

11. How do we read order status out of the EHR?

Module path(s) + version, OR report name, OR API endpoint.

If it's an API, attach the docs alongside this file.

e.g. /ehr/brightree/papi/get_sales_orders_with_filters v1.1.0

→

12. Instructions for CRON Looping

THIS IS THE MOST CRITICAL THING TO THINK THROUGH.

Based on the integrations you have to fetch and sync statuses, should you loop through all TOM orders and ping the EMR
for each to check status?

If credential time is a limited resource, is there a way you can bulk fetch statuses and simply loop over them in Tennr
to update TOM?

If you do fetch in bulk, are there any you might miss (e.g., archived/completed/voided orders)? Nothing should slip
through the cracks - we don't want 7mo old orders sitting in On Track.

Think through how many orders you might be looping over. If it's going to be a ton, do you have any direction on how
you'd like this to be done?

How frequently can you get the CRON to run (ideally twice a day but once a day at minimum is recommended).

If provider is not always populated during order creation but they add it into the EMR later, can you backfill using
this CRON worker when you pull order info?

→
