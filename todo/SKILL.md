---
name: todo
description: Use when the user asks to add, list, complete, remove, or manage todos, tasks, or reminders. Supports work and personal lists in Apple Reminders.
version: 1.1.0
---

# Todo Skill

This skill manages todos in Apple Reminders via JXA (JavaScript for Automation) using `osascript`. It maintains two lists: **Work** and **Personal**, always scoped to the **iCloud** account so they sync across all Apple devices.

## Prerequisites

- macOS with Reminders.app (syncs via iCloud across Apple devices)
- `osascript` is built-in — no installation needed

## Usage Syntax

```
/todo work <description>           — add to Work list
/todo personal <description>       — add to Personal list
/todo list                         — show all incomplete todos (both lists)
/todo list work                    — show only Work todos
/todo list personal                — show only Personal todos
/todo done <description>           — mark a todo as complete (searches both lists)
/todo remove <description>         — delete a todo entirely
```

The **first argument always determines the action or target list** — never infer or guess which list.

## Important: Always Use the iCloud Account

Always scope all operations to `app.accounts.byName("iCloud")`. The machine may have multiple accounts (e.g. Exchange/work). Using `app.lists.byName(...)` directly may resolve to the wrong account.

```javascript
const app = Application("Reminders");
const icloud = app.accounts.byName("iCloud");
// use icloud.lists.byName(...) for all operations
```

## Setup: Ensure Lists Exist

Run this before any operation to ensure the Work and Personal lists exist under iCloud:

```bash
osascript -l JavaScript -e '
const app = Application("Reminders");
const icloud = app.accounts.byName("iCloud");
const existing = icloud.lists().map(l => l.name());
["Work", "Personal"].forEach(name => {
  if (!existing.includes(name)) {
    icloud.lists.push(app.List({ name }));
  }
});
"OK";
'
```

## Add a Todo

```bash
# /todo work Buy coffee
osascript -l JavaScript -e '
const app = Application("Reminders");
const icloud = app.accounts.byName("iCloud");
const list = icloud.lists.byName("Work");
list.reminders.push(app.Reminder({ name: "Buy coffee" }));
"Added to Work";
'

# /todo personal Call mom
osascript -l JavaScript -e '
const app = Application("Reminders");
const icloud = app.accounts.byName("iCloud");
const list = icloud.lists.byName("Personal");
list.reminders.push(app.Reminder({ name: "Call mom" }));
"Added to Personal";
'
```

To add with a **due date**, include it in the Reminder object:

```bash
osascript -l JavaScript -e '
const app = Application("Reminders");
const icloud = app.accounts.byName("iCloud");
const list = icloud.lists.byName("Work");
list.reminders.push(app.Reminder({
  name: "Submit report",
  dueDate: new Date("2026-04-07T10:00:00")
}));
"Added to Work";
'
```

## List Todos

```bash
# Both lists
osascript -l JavaScript -e '
const app = Application("Reminders");
const icloud = app.accounts.byName("iCloud");
const results = {};
["Work", "Personal"].forEach(listName => {
  const list = icloud.lists.byName(listName);
  const items = list.reminders.whose({ completed: false })().map(r => ({
    name: r.name(),
    dueDate: r.dueDate() ? r.dueDate().toISOString().split("T")[0] : null
  }));
  results[listName] = items;
});
JSON.stringify(results, null, 2);
'

# Single list — replace "Work" with "Personal" as needed
osascript -l JavaScript -e '
const app = Application("Reminders");
const icloud = app.accounts.byName("iCloud");
const list = icloud.lists.byName("Work");
const items = list.reminders.whose({ completed: false })().map(r => ({
  name: r.name(),
  dueDate: r.dueDate() ? r.dueDate().toISOString().split("T")[0] : null
}));
JSON.stringify(items, null, 2);
'
```

Present the output as a clean list grouped by list name. Show due dates inline when present.

## Complete a Todo

Search both lists for the closest name match and mark it complete:

```bash
osascript -l JavaScript -e '
const app = Application("Reminders");
const icloud = app.accounts.byName("iCloud");
const target = "Buy coffee";  // replace with actual name
let found = false;
["Work", "Personal"].forEach(listName => {
  if (found) return;
  const list = icloud.lists.byName(listName);
  const matches = list.reminders.whose({ name: target, completed: false })();
  if (matches.length > 0) {
    matches[0].completed = true;
    found = listName;
  }
});
found ? "Completed in " + found : "Not found";
'
```

If no exact match is found, list the incomplete reminders from both lists and ask the user to clarify which one they meant.

## Remove a Todo

```bash
osascript -l JavaScript -e '
const app = Application("Reminders");
const icloud = app.accounts.byName("iCloud");
const target = "Buy coffee";  // replace with actual name
let found = false;
["Work", "Personal"].forEach(listName => {
  if (found) return;
  const list = icloud.lists.byName(listName);
  const matches = list.reminders.whose({ name: target })();
  if (matches.length > 0) {
    app.delete(matches[0]);
    found = listName;
  }
});
found ? "Removed from " + found : "Not found";
'
```

## Quick Reference

| Command | JXA operation | Target |
|---|---|---|
| `/todo work <text>` | `icloud.lists.byName("Work").reminders.push(...)` | Work list (iCloud) |
| `/todo personal <text>` | `icloud.lists.byName("Personal").reminders.push(...)` | Personal list (iCloud) |
| `/todo list` | `reminders.whose({completed: false})()` | Both lists (iCloud) |
| `/todo list work` | `reminders.whose({completed: false})()` | Work list only (iCloud) |
| `/todo list personal` | `reminders.whose({completed: false})()` | Personal list only (iCloud) |
| `/todo done <text>` | `reminder.completed = true` | First match in Work, then Personal |
| `/todo remove <text>` | `app.delete(reminder)` | First match in Work, then Personal |
