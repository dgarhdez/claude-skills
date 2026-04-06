---
name: cal
description: Use when the user asks to add, list, or manage Apple Calendar events. Supports work, personal, and IE calendars.
version: 1.0.0
---

# Cal Skill

This skill manages events in Apple Calendar via JXA (JavaScript for Automation) using `osascript`.

## Calendar Mapping

Always use these exact calendar names:

| Keyword | Calendar name |
|---|---|
| `work` | `daniel.hernandez@gostudent.org` |
| `personal` | `M&D` |
| `ie` | `Clases IE` |

## Usage Syntax

```
/cal work <title> <date> <time>                        — add work event (1h default duration)
/cal personal <title> <date> <time>                    — add personal event
/cal ie <title> <date> <time>                          — add IE class/event
/cal work <title> <date> <time> to <time>              — with explicit end time
/cal work <title> <date> <time> at <location>          — with location
/cal work <title> <date> <time> to <time> at <location> — end time + location
/cal list                                               — show events for today and tomorrow
/cal list today                                         — show today's events
/cal list tomorrow                                      — show tomorrow's events
/cal list this week                                     — show events for the next 7 days
```

The **first argument is always the calendar** (`work`, `personal`, or `ie`).

Default duration is **1 hour** if no end time is given.

## Add an Event

```bash
# Basic event (1h duration)
osascript -l JavaScript -e '
const app = Application("Calendar");
const cal = app.calendars.byName("daniel.hernandez@gostudent.org");
const start = new Date("2026-04-10T10:00:00");
const end = new Date("2026-04-10T11:00:00");
cal.events.push(app.Event({
  summary: "Team standup",
  startDate: start,
  endDate: end
}));
"Added";
'

# With location
osascript -l JavaScript -e '
const app = Application("Calendar");
const cal = app.calendars.byName("M&D");
const start = new Date("2026-04-10T20:00:00");
const end = new Date("2026-04-10T22:00:00");
cal.events.push(app.Event({
  summary: "Dinner",
  startDate: start,
  endDate: end,
  location: "Restaurante El Patio"
}));
"Added";
'
```

Substitute the calendar name, title, dates, and location from user input. If no end time is given, set `endDate` to `startDate + 1 hour`.

## List Events

```bash
# Today
osascript -l JavaScript -e '
const app = Application("Calendar");
const now = new Date();
const startOfDay = new Date(now.getFullYear(), now.getMonth(), now.getDate());
const endOfDay = new Date(now.getFullYear(), now.getMonth(), now.getDate(), 23, 59, 59);
const cals = ["daniel.hernandez@gostudent.org", "M&D", "Clases IE"];
const results = [];
cals.forEach(calName => {
  try {
    const cal = app.calendars.byName(calName);
    const events = cal.events.whose({
      startDate: { _greaterThanEquals: startOfDay },
      endDate: { _lessThanEquals: endOfDay }
    })();
    events.forEach(e => results.push({
      calendar: calName,
      title: e.summary(),
      start: e.startDate().toISOString(),
      end: e.endDate().toISOString(),
      location: e.location() || null
    }));
  } catch(e) {}
});
results.sort((a, b) => new Date(a.start) - new Date(b.start));
JSON.stringify(results, null, 2);
'
```

For `tomorrow`, offset `startOfDay`/`endOfDay` by +1 day. For `this week`, set end to `startOfDay + 7 days`.

Present events sorted by start time, grouped by date, showing time, title, and location (if set).

## Quick Reference

| Command | Operation |
|---|---|
| `/cal work <title> <date> <time>` | Create event in `daniel.hernandez@gostudent.org` |
| `/cal personal <title> <date> <time>` | Create event in `M&D` |
| `/cal ie <title> <date> <time>` | Create event in `Clases IE` |
| `... to <time>` | Set explicit end time |
| `... at <location>` | Set location |
| `/cal list [today\|tomorrow\|this week]` | List events (default: today + tomorrow) |
