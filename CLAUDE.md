# Daily Planning Routine

This repo drives a scheduled Claude Code routine that builds a daily plan from
Google Calendar and Notion, then **writes the result back to Notion**. Writing
back is not optional — it is the primary deliverable of every run.

---

## Git
Always commit and push directly to the `main` branch. Do not create feature branches.

## What to do on every run

### 1. Pull today's calendar
Call `mcp__Google-Calendar__list_events` for today (midnight → 11:59 PM, local
time America/New_York). List every event with start time, duration, and any
prep notes from the description.

### 2. Pull open tasks from Notion
Fetch the **Claude's ToDo** page (ID `37994fa6010c814b8162ef59db7ca9ab`) to
get the current task list.

### 3. Pull action items from recent recorded meetings
Fetch the **Recorded Meetings** index page
(`782685ced07245419c880a9c7948a232`) to get the list of meeting pages.
Then fetch each meeting from the **last 60 days** and extract action items
assigned to Michael (me). Cross-reference against what is already in
Claude's ToDo so you don't duplicate items.

### 4. Identify today's top priorities
Weigh calendar commitments, open task deadlines, and urgency. Pick the 3–5
highest-priority items for the day.

### 5. Write the updated plan back to Notion — REQUIRED STEP
Call `mcp__Notion__notion-update-page` with:
- `page_id`: `37994fa6010c814b8162ef59db7ca9ab`
- `command`: `replace_content`
- `allow_deleting_content`: `true`

The replacement content must follow this structure (Notion-flavored Markdown):
Today — <Weekday>, <Month> <Day>, <Year>
not done
<highest-priority task for today>
not done
<next task>
...
This Week
not done
<tasks due or urgent this week>
...
Active
not done
<ongoing tasks not due this week>
...
Backlog
not done
<lower-priority / deferred items>
...
Upcoming Milestones
<Date> — <milestone description>
...
Last updated <Month> <Day>, <Year> by Claude

Rules for the content:
- The **Today** section header must always use the current date.
- Tasks that were previously checked (`[x]`) must **not** be deleted. Instead,
  move them to the **Completed Tasks** page (ID `38794fa6010c8175a2d1ced6131423ae`)
  by calling `mcp__Notion__notion-update-page` with `command: insert_content` and
  `position: {"type": "end"}`. Append each completed task as a checked markdown
  checkbox (`- [x] <task text>`) prefixed with the completion date:
  `- [x] **<Month> <Day>, <Year>** — <task text>`.
  Do this **before** writing the updated ToDo so nothing is lost.
- **IMPORTANT:** The Completed Tasks page must **never** be a child page of
  Claude's ToDo. The `replace_content` + `allow_deleting_content: true` call
  will silently delete any child pages of Claude's ToDo that are not referenced
  in the replacement content. Completed Tasks lives under the BOP page
  (`2ae94fa6010c804c9f41c7ea5da52eea`) as a sibling of Claude's ToDo — do not
  move it.
- After archiving, omit the completed tasks from the ToDo replacement content.
- New action items discovered in meeting notes should be **added** to the
  appropriate section with a citation link back to the meeting page.
- Preserve any unchecked tasks from the previous version that are not yet done.
- Keep 🔴 emoji prefix for overdue or hard-deadline items.
- Link each task to its source meeting page using Markdown links.
### 6. Output a summary in chat
After writing to Notion, output the daily plan as formatted text in the
conversation so it is readable in the session transcript. This is secondary
to the Notion write — always do the Notion write first.
---
## Key Notion pages
| Page | ID |
|------|----|
| Claude's ToDo | `37994fa6010c814b8162ef59db7ca9ab` |
| Completed Tasks | `38794fa6010c8175a2d1ced6131423ae` |
| Recorded Meetings index | `782685ced07245419c880a9c7948a232` |
| My Tasks database | `d2f025bcce6c4d6b94a736c8038c07f1` |
## Key context
- **Company:** Aprio (government side); splitting from CanAid ~July 1, 2026
- **Role:** Senior Cybersecurity Architect → CISO as of July 1
- **Primary compliance thread:** DOJ Year 2 Work Plan due July 26
- **Primary security thread:** CrowdStrike renewal (contract expires July 26;
  opt-out/sign deadline after July 1 address change)
- **Time zone:** America/New_York