---
name: cal
description: Use when the user asks to add, list, or manage Apple Calendar events. Supports work, md (shared with wife), and personal calendars.
version: 2.0.0
---

# Cal Skill

This skill manages events in Apple Calendar via JXA (JavaScript for Automation) using `osascript`.

## Calendar Mapping

Always use these exact calendar names:

| Keyword | Calendar name |
|---|---|
| `work` | `daniel.hernandez@gostudent.org` |
| `md` | `M&D` |
| `personal` | `dgarhdez@gmail.com` |

## Usage Syntax

```
/cal work <title> <date> <time>                         — add work event (1h default duration)
/cal md <title> <date> <time>                           — add event to shared calendar with wife
/cal personal <title> <date> <time>                     — add personal event
/cal work <title> <date> <time> to <time>               — with explicit end time
/cal work <title> <date> <time> at <location>           — with location
/cal work <title> <date> <time> to <time> at <location> — end time + location
/cal list                                               — show events for today and tomorrow
/cal list today                                         — show today's events
/cal list tomorrow                                      — show tomorrow's events
/cal list this week                                     — show events for the next 7 days
```

The **first argument is always the calendar** (`work`, `md`, or `personal`).

Default duration is **1 hour** if no end time is given.

## Add an Event

Without location:

```bash
osascript -l JavaScript -e '
const app = Application("Calendar");
const cal = app.calendars.byName("daniel.hernandez@gostudent.org");  // or "M&D" or "dgarhdez@gmail.com"
const start = new Date("2026-04-10T10:00:00");
const end = new Date("2026-04-10T11:00:00");  // startDate + 1h if no end time given
cal.events.push(app.Event({
  summary: "Team standup",
  startDate: start,
  endDate: end
}));
"Added";
'
```

With location:

```bash
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

Substitute calendar name, title, dates, and location from user input. If no end time is given, set `endDate` to `startDate + 1 hour`.

## List Events

```bash
osascript -l JavaScript -e '
const app = Application("Calendar");
const now = new Date();
// For today: startOfDay/endOfDay same day
// For tomorrow: offset by +1 day
// For this week: end = startOfDay + 7 days
const startOfDay = new Date(now.getFullYear(), now.getMonth(), now.getDate());
const endOfDay = new Date(now.getFullYear(), now.getMonth(), now.getDate(), 23, 59, 59);
const cals = ["daniel.hernandez@gostudent.org", "M&D", "dgarhdez@gmail.com"];
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

Present events sorted by start time, grouped by date, showing time, title, and location (if set).

## Quick Reference

| Command | Calendar |
|---|---|
| `/cal work ...` | `daniel.hernandez@gostudent.org` |
| `/cal md ...` | `M&D` |
| `/cal personal ...` | `dgarhdez@gmail.com` |
| `... to <time>` | Set explicit end time |
| `... at <location>` | Set location |
| `/cal list [today\|tomorrow\|this week]` | List events across all 3 calendars |
