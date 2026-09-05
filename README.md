# servicenow-phone-provisioning

Claude skill. Automates the HBS TSS new-faculty phone provisioning workflow in
ServiceNow using the Claude in Chrome extension.

Given a list of Universal Requests, it pulls each phone's serial, user, and
location from the related Universal Tasks, loads the serials into the Hardware
(`alm_hardware`) list as one filtered view, and pre-fills the "Update Phone
Location" tasks. It reads and pre-fills only. **The technician performs every
save.**

Runs against the ServiceNow **classic UI**. Service Operations Workspace cannot
be automated.

## Contents

| Path | What it is |
|---|---|
| `SKILL.md` | The skill |
| `references/servicenow-patterns.md` | Selector and navigation patterns the skill relies on |
| `phone-provisioning-automation-handoff.md` | Build notes and current state. Hand to a new Claude session. |

## Install

The repo root is the skill directory. Clone it into your Claude skills folder
under its own name:

```bash
git clone https://github.com/jgerdom-hbs/servicenow-phone-provisioning.git \
  ~/.claude/skills/servicenow-phone-provisioning
```

For claude.ai, download a packaged `.skill` from Releases and upload it in
settings.

## Sharing

Shareable across TSS. Nothing in it is specific to one technician.
