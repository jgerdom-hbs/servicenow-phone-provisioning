# HBS Phone Provisioning — ServiceNow Automation (Session Handoff)

## How to use this file
Upload this file at the start of a new chat to resume this work. It captures the objective, the ServiceNow data model, the access constraints, the roadblocks hit and how they were solved, the reusable technical patterns, and what was completed. A fresh session should be able to act from this without rediscovering the environment.

Automation runs through the **Claude in Chrome** extension against the ServiceNow **classic UI** (see "Environment findings" for why).

---

## Objective
Reduce the manual effort in a repeatable process: when a phone is allocated to a new faculty member, update the phone's Hardware (asset) record and complete the related "Update Phone Location" task.

## Operator and access constraints
- Standard ITIL user in **prod**; a few extra permissions in the **test** environment only. **Not** a ServiceNow admin (power user, not administrator).
- **No** Power Platform connector in prod (test only, and not shared with the team).
- Because of the above, native platform automation (Flow Designer, etc.) is not available; the approach is browser-driven against the classic UI.

## Baseline manual process
- Trigger: the stockroom manager allocates a phone to a user.
- One phone per consultation/UR.
- Steps: find the phone's serial in the task activity history, locate the phone in the Hardware table, set status / assigned-to / location, save; then complete the "Update Phone Location" task.
- Split adopted this session: Claude scrapes serials/users/locations and runs the Hardware search; the technician does the three-field asset edits (formatting is inconsistent) and the final saves.

---

## Environment findings (roadblocks and solutions)

1. **SOW (Service Operations Workspace) is not automatable.** Everything renders in deeply nested web components (shadow DOM); page text and the accessibility tree come back empty.
   - Solution: work against the **classic UI** instead.

2. **Classic UI is reachable but buried.** The instance wraps classic inside the new shell; the real record form lives in an `iframe#gsft_main` under ~75 shadow-DOM layers, so standard "read the page" tooling sees nothing.
   - Solution: a recursive shadow-piercing helper to reach `gsft_main`, then read/set via its `g_form` / DOM (see snippets).

3. **Raw page-text dumps get auto-blocked** (flagged as cookie/query-string data).
   - Solution: extract only the specific parsed fields (never dump whole `innerText`); keep any IDs inside the page and return only display values.

4. **Phone-block formatting is inconsistent** across Provision Peripherals tasks: some use `Serial:` on its own line, some use `SN:` inline, and at least one recorded **only a MAC** (no serial).
   - Solution: format-tolerant regex; MAC fallback when no serial exists.

5. **The group table can't be read via the data API** for this account (a `sys_user_group` query returns empty even for broad matches), and the assignment-group reference field's type-ahead only fires on **real** keystrokes, not scripted events.
   - Solution: could not set Assignment group programmatically. It was left at its existing value ("CompuCom Stockroom"), which the operator is a member of and which is acceptable for these tasks. If a specific group is ever required, set it by hand.

---

## Data model reference

- **Universal Request** — table `universal_request`, prefix `UR`. The parent record ("Faculty Onboarding …").
- **Universal Task** — table `sn_uni_task_universal_task`, prefix `UNT`. Children of the UR, linked by the **`universal_request`** reference field (query children with dot-walk: `universal_request.number=UR00xxxxx`).
- **Phone info** lives in the activity log of a task whose short description starts **`Provision Peripherals - New Faculty`**. Up to three such tasks per user; the phone is in the **only** one if there is one, or the **second** (by number order) if there are two or more. Only the correct task actually contains the serial/MAC block, which is a useful cross-check.
- **Location + user** come from the task whose short description starts **`Update Phone Location - New Faculty`**:
  - Location: in the description, after "…following location: **X**".
  - User: last ` - ` segment of the short description (e.g., "… - Charvel, Roberto").
- **Hardware / assets** — table `alm_hardware`, keyed on `serial_number`. The MAC field is the custom **`u_mac_address`**, stored **uppercase, no separators** (e.g., `00F82C0721C5`). Use this to locate a phone when the task recorded no serial.
- **Task field facts for completion:** State "Complete" = value **`3`**; "Resolution notes" is the **`close_notes`** field; the current signed-in user's sys_id is `g_user.userID` (with `g_user.getFullName()`).

---

## Reusable technical patterns

**Reach the classic form (shadow-piercing):**
```js
function findFrame(root){
  const f = root.querySelector && root.querySelector('iframe#gsft_main');
  if (f) return f;
  for (const el of root.querySelectorAll('*')) {
    if (el.shadowRoot) { const r = findFrame(el.shadowRoot); if (r) return r; }
  }
  return null;
}
// const g = findFrame(document).contentWindow.g_form;
// const d = findFrame(document).contentDocument;
```

**Navigation URL templates** (host: `https://hbs.service-now.com`):
- Open a record: `/nav_to.do?uri=%2F<table>.do%3Fsys_id%3D<SYS_ID>`
- Child tasks of a UR: `/nav_to.do?uri=%2Fsn_uni_task_universal_task_list.do%3Fsysparm_query%3Duniversal_request.number%3D<UR>%5EORDERBYnumber`
- Hardware "serial is one of": `/nav_to.do?uri=%2Falm_hardware_list.do%3Fsysparm_query%3Dserial_numberIN<S1,S2,S3>%5EORDERBYserial_number`
- Hardware by MAC: `…alm_hardware_list.do?sysparm_query=u_mac_address=<MAC>` (uppercase, no separators)

**Format-tolerant phone-block extraction** (run against `gsft_main` body text):
```js
const serial = (body.match(/(?:Serial|SN|S\/N)\s*[:#]?\s*([A-Z0-9]{6,})/i) || [])[1] || null;
const mac    = (body.match(/MAC\s*[:#]?\s*([0-9A-Fa-f:]{11,17})/i) || [])[1] || null;
// If no serial, fall back to locating the asset by MAC via u_mac_address.
```

**Set completion fields** (leave the Save to the operator):
```js
g.setValue('assigned_to', g_user.userID, g_user.getFullName());
g.setValue('state', '3');              // Complete
g.setValue('close_notes', 'Location updated');
// assignment_group could not be set programmatically (see roadblock 5).
```

---

## Three-phase workflow
1. **Read:** for each UR, list children, pick the phone task (rule above), extract serial (+ user, location from the Update Phone Location task). Compile a table sorted by serial to match the Hardware list.
2. **Hardware:** load all serials into the Hardware list with a single "serial is one of" filter, sorted by serial; the operator does the three-field asset edits.
3. **Close tasks:** open each Update Phone Location task, pre-fill assigned-to / state / resolution notes; the operator sets any remaining field and saves.

---

## Completed this session (all saved)
This batch is fully done; nothing is pending from it.

| UR | Faculty | Serial | Location | Provision task | Update-Location task |
|---|---|---|---|---|---|
| UR0030765 | Chaudhary, Niren | WZP21130VSD | Morgan Hall 230 | UNT0006797 | UNT0006800 |
| UR0031764 | Dubrowski, Dan | WZP21130W5C | Baker Library 165 | UNT0006776 | UNT0006780 |
| UR0030562 | Li, Jessica | WZP21130W95 | Baker Library 360 | UNT0006833 | UNT0006836 |
| UR0031756 | Charvel, Roberto | WZP211401GM | Baker Library 165 | UNT0006786 | UNT0006789 |
| UR0031765 | Wu, Charlie | WZP232502AQ | Baker Library 161 | UNT0006808 | UNT0006811 |
| UR0030563 | Vaccaro, Michelle | WZP21460ESA | Morgan Hall 492 | UNT0006838 | UNT0006842 |

Notes:
- UR0030563 (Vaccaro): the Provision task recorded only a MAC (`B4A8B9E8A610`); the serial was recovered from Hardware via `u_mac_address`.
- All six Update Phone Location tasks were pre-filled (assigned-to Jason Gerdom, state Complete, resolution notes "Location updated"), assignment group left as "CompuCom Stockroom," and saved by the operator.

---

## Open items / next steps
- **Package as a repeatable skill** so another technician can run it (design discussed separately).
- **Upstream fix (highest leverage):** standardize how the stockroom records the phone block in Provision Peripherals tasks (always `Serial:` + `MAC:` in a fixed format). Inconsistent formatting was the main reliability snag.
- Consider building the **MAC fallback** into the standard extraction so serial-less tasks resolve automatically.
