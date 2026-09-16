# kanban

Single-page IT project management Kanban board for an internal "UOB IT PMO" demo/training tool.

Vanilla HTML, CSS and JavaScript in one file — no framework, no build step, no dependencies.
Open `index.html` in a browser to run it.

## Features

- Four fixed columns: Backlog, In Progress, Blocked, Done — with live count badges
- Native HTML5 drag and drop, plus a keyboard-accessible "Move" control on every card
- Add Task form with client-side validation and inline error messages
- Filter by project, assignee and priority
- Live summary strip: totals per status and overdue count
- Priority colour-coding, overdue badges, and inline delete confirmation

## Notes

- Board state is held in memory only. Refreshing the page resets it to the seeded demo data.
- Email notifications go through [FormSubmit](https://formsubmit.co). Set `FORMSUBMIT_ENDPOINT`
  at the top of the script to your address; FormSubmit requires a one-time activation click
  before any mail is delivered.
- This is a demo and training tool. It uses a neutral text wordmark and is not affiliated with,
  nor does it imitate, any official UOB system.
