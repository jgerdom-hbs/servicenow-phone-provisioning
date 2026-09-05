---
name: servicenow-phone-provisioning
description: >-
  Automate the HBS TSS new-faculty phone provisioning workflow in ServiceNow using the Claude in Chrome
  extension: given a list of Universal Requests (URs), extract each phone's serial, user, and location from
  the related Universal Tasks, load the serials into the Hardware (alm_hardware) list as one filtered view for
  editing, and pre-fill the "Update Phone Location" tasks for the technician to review and save. Use this skill
  whenever the user mentions phone provisioning, faculty phone allocation, updating phone locations, the
  "Provision Peripherals" or "Update Phone Location" tasks, pulling serials from URs/UNTs, or batching phone
  asset updates in ServiceNow, even if they don't name the skill directly. Runs against the ServiceNow CLASSIC
  UI (Service Operations Workspace cannot be automated). Read-only scraping plus field pre-fill only; the
  technician always performs the final saves.
---

# ServiceNow Phone Provisioning (HBS TSS)

## What this does
When a phone is allocated to a new faculty member, three things must happen in ServiceNow: the phone's Hardware asset record gets updated (assigned-to, location, status), and the related "Update Phone Location" task gets completed. This skill handles the repetitive parts across a batch of Universal Requests (URs): reading the phone data out of the tasks, loading the phones into one Hardware list for editing, and pre-filling the location tasks.

It is built for a standard ITIL user (not a ServiceNow admin) with no Power Platform access in production, so everything runs through the Claude in Chrome extension against the ServiceNow classic UI.

## Prerequisites
- The Claude in Chrome extension, connected to the browser holding the technician's ServiceNow session.
- A signed-in ServiceNow session with normal ITIL access at the HBS instance (`https://hbs.service-now.com`).
- For completing the location tasks: membership in the assignment group already on those tasks (typically **CompuCom Stockroom**), so the group can be left as-is.

## Non-negotiable guardrails
These protect a production asset system, so follow them every time.
- **Never save or submit.** This skill reads records and pre-fills fields only. The technician reviews and clicks Save on every record. Do not call any save/submit control.
- **Work in the classic UI, never SOW.** Service Operations Workspace renders in nested shadow DOM and cannot be read or driven reliably. Use the classic URL templates in `references/servicenow-patterns.md`.
- **Verify, don't assume.** After building the Hardware filter, confirm every serial matched a record. After pre-filling tasks, read the values back before reporting done. A misread serial means a phone assigned to the wrong person.
- **Extract only the fields you need.** Never dump whole-page text; the browser tooling blocks output that looks like session data. Pull specific parsed values and keep any record IDs inside the page.

## Inputs and outputs
- **Input:** a list of UR numbers (e.g., `UR0031756`). The technician can paste them directly.
- **Output:**
  1. A reference table sorted by serial number (User, Serial, Location, UR) that matches the Hardware list order.
  2. The Hardware list filtered to those serials in one view, ready for the technician's edits.
  3. The "Update Phone Location" tasks pre-filled and left unsaved for review.

## Data model (summary)
- **Universal Request** — table `universal_request`, prefix `UR`. The parent record.
- **Universal Task** — table `sn_uni_task_universal_task`, prefix `UNT`. Children of the UR, linked by the `universal_request` field.
- **Phone data** lives in the activity log of the task whose short description starts `Provision Peripherals - New Faculty`. There can be up to three such tasks per user; the phone is in the **only** one if there is a single task, or the **second** (by number order) if there are two or more. Only the correct task actually contains a serial/MAC block, which is a useful cross-check.
- **Location and user** come from the task whose short description starts `Update Phone Location - New Faculty` (location in the description after "…following location: X"; user is the last ` - ` segment of the short description).
- **Hardware / assets** — table `alm_hardware`, keyed on `serial_number`; the MAC field is the custom `u_mac_address` (uppercase, no separators).

Exact URL templates, field values, and the JavaScript helpers are in `references/servicenow-patterns.md`. Read that file before running the steps below.

## Workflow

### Phase 1 — Extract (read-only)
For each UR:
1. Open the UR's child tasks as a classic list, ordered by number (see the child-list query template).
2. Identify the `Provision Peripherals - New Faculty` task(s) and the `Update Phone Location - New Faculty` task.
3. Open the correct Provision Peripherals task (only one, or the second of two or more) and extract the serial (and MAC/model) with the format-tolerant matcher. If no serial is present but a MAC is, record the MAC for a fallback lookup in Phase 2.
4. Open the Update Phone Location task and read the location and the user.

Compile the results into a table sorted by serial number ascending, to line up with the Hardware list.

### Phase 2 — Hardware list (read-only; technician edits)
1. Load all collected serials into the Hardware list with a single "serial is one of" filter, sorted by serial number.
2. Confirm each serial matched a record. For any phone that had only a MAC, search Hardware by `u_mac_address` to recover its serial, confirm it is the right unit, and add it to the table.
3. Hand the filtered list to the technician for the asset edits (assigned-to, location, status). Formatting quirks make these edits a human step.

### Phase 3 — Pre-fill location tasks (no saving)
1. Open each Update Phone Location task in its own browser tab so they are easy to review and close.
2. On each, set:
   - **Assigned to** = the current user (`g_user.userID` / `g_user.getFullName()`).
   - **State** = Complete (value `3`).
   - **Resolution notes** (`close_notes`) = `Location updated`.
   - **Assignment group** — leave as-is (typically CompuCom Stockroom). It cannot be set programmatically (data API blocked for standard users, and the field's type-ahead only fires on real keystrokes). If a different group is genuinely required, tell the technician to type it by hand.
3. Read the values back and report the per-task summary. The technician confirms and saves each one.

## Known limitations
- **SOW is not automatable** — classic UI only.
- **Phone-block format varies** in Provision Peripherals tasks (`Serial:` on its own line, `SN:` inline, or MAC-only). The format-tolerant matcher plus the MAC fallback handle this; if the source format is standardized upstream, extraction gets more reliable, but keep the fallback for older records.
- **UI brittleness** — because this drives the classic UI through heavy shadow DOM, a ServiceNow UI change can require updating the helpers in the reference file.

## Example triggers
**Example 1:**
Input: "I've got a batch of new-faculty phone URs to process — can you pull the serials and set up the Hardware list?"
Action: run Phases 1 and 2, present the sorted table and the filtered Hardware list.

**Example 2:**
Input: "Pre-fill the Update Phone Location tasks for these URs so I just have to save them."
Action: run Phase 1 to identify the tasks, then Phase 3 to pre-fill and report back for the technician to save.
