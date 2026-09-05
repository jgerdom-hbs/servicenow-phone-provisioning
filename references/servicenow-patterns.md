# ServiceNow patterns and helpers

Host: `https://hbs.service-now.com`. Everything below targets the classic UI, which on this instance is wrapped inside the new shell: the real record form lives in an `iframe#gsft_main` buried under many shadow-DOM layers. Standard "read the page" tooling sees nothing, so reach the form with the helper first.

## Reaching the classic form (shadow-piercing)
```js
function findFrame(root){
  const f = root.querySelector && root.querySelector('iframe#gsft_main');
  if (f) return f;
  for (const el of root.querySelectorAll('*')) {
    if (el.shadowRoot) { const r = findFrame(el.shadowRoot); if (r) return r; }
  }
  return null;
}
// const frame = findFrame(document);
// const g = frame.contentWindow.g_form;   // classic form API
// const gu = frame.contentWindow.g_user;  // current user (gu.userID, gu.getFullName())
// const d = frame.contentDocument;        // classic DOM
```

Notes:
- After navigating a tab, wait ~2.5–5s before reading; `gsft_main` loads asynchronously. Activity-log content in particular can lag; if a serial read comes back empty, wait longer and retry before concluding it is missing.
- Run `fetch` and DOM reads from the tab's top document (same origin as the instance).

## Navigation URL templates
Open in the classic wrapper via `nav_to.do?uri=<encoded classic path>`:
- **Open a record:** `/nav_to.do?uri=%2F<table>.do%3Fsys_id%3D<SYS_ID>`
- **A UR's child tasks (ordered):** `/nav_to.do?uri=%2Fsn_uni_task_universal_task_list.do%3Fsysparm_query%3Duniversal_request.number%3D<UR>%5EORDERBYnumber`
- **Hardware "serial is one of":** `/nav_to.do?uri=%2Falm_hardware_list.do%3Fsysparm_query%3Dserial_numberIN<S1,S2,S3>%5EORDERBYserial_number`
- **Hardware by MAC:** append `sysparm_query=u_mac_address=<MAC>` to `alm_hardware_list.do` (MAC uppercase, no separators, e.g. `00F82C0721C5`).

`%2F` = `/`, `%3F` = `?`, `%3D` = `=`, `%5E` = `^` (ServiceNow query separator). `IN` is the "is one of" operator with a comma-separated list.

## Identifying tasks in a child list
Reading the classic list, scan anchors to `sn_uni_task_universal_task.do?sys_id=` and classify each row by the short description in its `<tr>` text:
```js
const isProvision = /Provision Peripherals - New Faculty/i.test(rowText);
const isLocation  = /Update Phone Location - New Faculty/i.test(rowText);
```
List DOM order follows the `ORDERBYnumber` query, so among Provision tasks, index 0 is the only/first and index 1 is the second (the one that holds the phone when there are two or more).

## Phone-block extraction (format-tolerant, with MAC fallback)
Run against the Provision Peripherals task's `gsft_main` body text:
```js
const serial = (body.match(/(?:Serial|SN|S\/N)\s*[:#]?\s*([A-Z0-9]{6,})/i) || [])[1] || null;
const mac    = (body.match(/MAC\s*[:#]?\s*([0-9A-Fa-f:]{11,17})/i) || [])[1] || null;
// If serial is null but mac is present, look the asset up in Hardware by u_mac_address to recover the serial.
```
Beware false positives when searching loosely (e.g., the "SN Utils" browser plugin string). Anchor on the labels above rather than bare tokens.

## Location and user (Update Phone Location task)
Via `g_form` on that task:
```js
const s = g.getValue('short_description') || '';
const user = s.split(' - ').map(x => x.trim()).pop();        // e.g. "Charvel, Roberto"
const desc = g.getValue('description') || '';
const location = (desc.match(/following location:\s*([^\n]+)/i) || desc.match(/Location:\s*([^\n]+)/i) || [])[1]?.trim();
```

## Completion field facts (Phase 3)
- `assigned_to` — set to the running user: `g.setValue('assigned_to', gu.userID, gu.getFullName())`.
- `state` — "Complete" = value `'3'`: `g.setValue('state', '3')`.
- Resolution notes — the `close_notes` field: `g.setValue('close_notes', 'Location updated')`.
- `assignment_group` — cannot be set programmatically. The `sys_user_group` table is not readable via the data API for standard ITIL accounts (queries return empty), and the reference field's type-ahead only responds to real keystrokes. Leave the existing value (typically CompuCom Stockroom). If a change is truly required, the technician types it by hand.

Read values back with `g.getDisplayValue(field)` (and `g.getValue('close_notes')`) to verify before reporting done. Do not save.

## Hardware asset record (technician's manual edits)
- Table `alm_hardware`, keyed on `serial_number`.
- MAC lives in `u_mac_address` (uppercase, no separators) — matches the task's MAC format, so it is a reliable fallback key.
- The three fields the technician sets (assigned-to, location, status) are done by hand because of formatting variability; this skill only stages the filtered list.

## Browser tab mechanics
- The extension operates in its own MCP tab group; the technician's pre-existing tabs are outside it. Create tabs in the group as needed rather than reusing the technician's tabs.
- For Phase 3, open each Update Phone Location task in its own tab so they are easy to review and close one at a time.
- Navigating a record inside a batch requires an explicit tab id; creating tabs is done one at a time.
