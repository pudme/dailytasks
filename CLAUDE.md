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
get the current task list. Record **every unchecked task** currently on the
page, including any that have no meeting citation — these are tasks the user
added manually and must be preserved exactly as written.

**Caution — this fetch can be stale.** The Notion fetch/view tool is
sometimes backed by a cached snapshot rather than the live page (an "as of"
timestamp on the fetch result can lag real time by a day or more — this has
been observed directly, see the incident note under Key Notion pages). Treat
this Step 2 read as a first look only. Do not build the Step 5 write-back
from it — re-fetch immediately before writing (Step 5) and use that fresh
copy as the source of truth for what's actually checked/present.

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

**First, re-fetch the ToDo page fresh** (do not reuse the Step 2 read — see
the staleness warning above). Diff this fresh copy against what Step 2 saw:
any box that's checked now but wasn't in Step 2 is a task the user completed
since that read and must be archived, not silently dropped.

**Default to targeted edits, not a full-page overwrite.** Call
`mcp__Notion__notion-update-page` with `command: update_content` and a
`content_updates` array of small `old_str`/`new_str` pairs — one per line
being removed, added, or changed (archived task lines, new action items from
meetings, the Today header date, overdue-day counts, the "Last updated"
line). Copy each `old_str` verbatim from the fresh fetch. This is the whole
point: if the live page has drifted from what you think it looks like, a
mismatched `old_str` makes the call fail loudly instead of silently
clobbering whatever changed. A blind `replace_content` rebuild, by contrast,
overwrites the entire page with your reconstruction — including reverting
any checkbox or edit the user made that your read didn't capture. Only reach
for `replace_content` when the page genuinely needs restructuring beyond
what a set of targeted edits can express, and even then, do the fresh
re-fetch immediately beforehand (never rely on an earlier read) — and treat
that as the exception, not the routine path.

- Do **not** set `allow_deleting_content` (leave it `false`/omitted). The ToDo
  page should never need to delete a block or child page — if the tool ever
  errors out saying content would be deleted, that means something unexpected
  is being removed. **Stop and ask the user** rather than retrying with
  `allow_deleting_content: true`. (See the incident note under Key Notion
  pages — this exact flag previously caused the Completed Tasks page to be
  trashed.)

The page content follows this structure (Notion-flavored Markdown):
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
  move them to the **Completed Tasks** page (ID `38e94fa6010c8143a025dae32dc46d1d`)
  by calling `mcp__Notion__notion-update-page` with `command: insert_content` and
  `position: {"type": "end"}`. Append each completed task as a checked markdown
  checkbox (`- [x] <task text>`) prefixed with the completion date:
  `- [x] **<Month> <Day>, <Year>** — <task text>`.
  Do this **before** editing the ToDo page so nothing is lost.
- After archiving, remove the completed tasks' lines from the ToDo page (via
  a targeted `update_content` edit per line — see above).
- **Every unchecked task currently on the page must be preserved**, including
  tasks the user added manually (i.e. tasks with no meeting citation). Do not
  drop or rewrite them. If you're doing targeted edits, this is automatic —
  you're only touching lines you have a specific reason to change. The only
  tasks that may be removed are ones that are checked `[x]` (archived per
  above) or that are exact duplicates of another task already present.
- New action items discovered in meeting notes should be **added** to the
  appropriate section (via a targeted `update_content` insertion) with a
  citation link back to the meeting page.
- Keep 🔴 emoji prefix for overdue or hard-deadline items.
- Link each task to its source meeting page using Markdown links.
- The page must always end with the "Last updated" line followed by this
  navigational link (this link is no longer what protects the page — see
  incident note below):
  `[→ Completed Tasks](https://app.notion.com/p/38e94fa6010c8143a025dae32dc46d1d)`
### 6. Output a summary in chat
After writing to Notion, output the daily plan as formatted text in the
conversation so it is readable in the session transcript. This is secondary
to the Notion write — always do the Notion write first.
---
## Key Notion pages
| Page | ID |
|------|----|
| Claude's ToDo | `37994fa6010c814b8162ef59db7ca9ab` |
| Completed Tasks | `38e94fa6010c8143a025dae32dc46d1d` |
| Recorded Meetings index | `782685ced07245419c880a9c7948a232` |
| My Tasks database | `d2f025bcce6c4d6b94a736c8038c07f1` |

**Incident note (2026-07-01):** Completed Tasks used to be a nested child
page *under* Claude's ToDo. Every `replace_content` call on the ToDo page
(with `allow_deleting_content: true`) moved it to Trash, because a plain
Markdown link to the page does **not** count as "referencing" a child page
block in Notion's eyes — only an actual embedded page block does, and the
routine's content never included one. This had already happened once before:
the original Completed Tasks page (`38494fa6010c8020950cdcc26cfe2630`) was
permanently deleted around June 22, which is why the page ID in this file
was changed to a freshly-created replacement (`38e94fa6...`) on June 29 —
that fix only added the same ineffective link-at-the-end workaround, so the
new page got trashed again by June 30's run. (There's also an abandoned,
empty duplicate "Completed Tasks" page from ~June 22 living under BOP →
Apprio, presumably a discarded recreation attempt — harmless leftover
clutter, not otherwise touched.)
Real fix applied: the current Completed Tasks page was restored from Trash
and **moved out from under Claude's ToDo to the workspace top level**, so
it's structurally independent and can never be caught up in a ToDo-page
content replacement again, regardless of what that content contains.
Combined with dropping `allow_deleting_content` from the write-back call
(above), this closes the hole permanently — do not re-nest Completed Tasks
under Claude's ToDo, and do not re-add `allow_deleting_content: true` to
that call.

**Incident note (2026-08-02):** Michael reported that boxes he checked off
in Notion kept reappearing unchecked the next day. Root cause: the Notion
fetch/view tool returned a stale cached snapshot of the ToDo page (an "as
of" timestamp a full day behind real time, confirmed by fetching the same
page twice minutes apart and getting two different snapshot ages). The
routine's write-back used `replace_content`, which rebuilds the entire page
from whatever was read — so when that read was stale, the rebuild
reproduced the old, pre-checkmark content and silently erased whatever the
user had checked off since the snapshot was taken. Real fix applied: Step 5
now re-fetches immediately before writing (never reuses an earlier read) and
defaults to targeted `update_content` edits instead of a full `replace_content`
rebuild, so a stale or drifted read fails loudly (mismatched `old_str`)
instead of silently overwriting live edits. Do not revert to building the
ToDo write-back from a Step-2-vintage read, and do not make `replace_content`
the default path again.
## Key context
- **Company:** Aprio (government side); splitting from CanAid ~July 1, 2026
- **Role:** Senior Cybersecurity Architect → CISO as of July 1
- **Primary compliance thread:** DOJ Year 2 Work Plan due July 26
- **Primary security thread:** CrowdStrike renewal (contract expires July 26;
  opt-out/sign deadline after July 1 address change)
- **Time zone:** America/New_York