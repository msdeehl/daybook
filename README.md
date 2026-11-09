# Daybook

A personal planner in a single HTML file: a month calendar, a daily habit tracker, and a color-coded to-do list with recurring reminders. It has no build step, no dependencies and no server. Open the file in a browser and it runs.

> **Status: prototype.** Login uses **mock, browser-only accounts**. It is fine for personal use and demos, but it is **not secure** and does not sync between devices. See [Security and data](#security-and-data).

---

## Features

**Accounts (mock)**
- Create an account with a username, an optional display name and a passcode.
- Each account's data is stored separately in the browser.
- "Keep me signed in on this device" is optional. Without it, the session ends when the tab closes.
- Sign out from the header.

**Calendar**
- Opens in month view, with previous, next and Today navigation.
- Click any day to add, edit or delete events. Each event has a **title, date, start time, optional end time, location and notes**.
- Event times and titles appear on the calendar. To-dos with due dates also appear on their day.
- Today's date has a spinning monkey (original SVG artwork, drawn in code).

**Daily habit tracker** (left panel)
- Add and delete habits, and check them off each day.
- Checkboxes **reset at midnight Eastern Time** (America/New_York), even if the page is left open. A countdown shows the time until the next reset.
- A progress bar shows how many habits are checked today.
- The **Summary** popup shows, for each habit: today's status, completions in the last 7 days, the current streak and the **all-time total**.

**To-do list** (right panel)
- Each to-do has three separate labels, each with its own colors:
  - **Priority:** High / Medium / Low (red / amber / green)
  - **Importance:** Critical / Important / Minor (purples)
  - **Work required:** Quick / Medium / Big (blues)
- The task's left border shows its priority color. Every label also has text, so color is never the only signal.
- Optional due date. Overdue items are flagged in red.
- Sort by priority, importance, least work first, or due date. Filter to open, done or all.

**Reminders**
- A reminder card lists open to-dos (overdue first) and the rest of today's events.
- It appears when you sign in and repeats on a timer: off, 15 min, 30 min, 1 hour or 2 hours.
- The 🔔 button shows it immediately. The badge and the browser tab title show the number of open to-dos.

**Backup**
- **Export** downloads your data as `daybook-<username>-<date>.json`.
- **Import** restores a backup into the signed-in account, after a confirmation step.

---

## Quick start

1. Download `daybook-local.html`.
2. Open it in a current version of Chrome, Edge, Firefox or Safari.
3. Choose **Create account**, pick a username and passcode, and start adding habits, events and to-dos.

To use it from any device on the web, host the file on any static host and rename it `index.html`. Good options are GitHub Pages, Netlify or Cloudflare Pages. It needs to be served over **https://** or opened as a local file, because passcode hashing uses the browser's Web Crypto API.

> Your data is tied to the **browser and the address** you open the file from. Moving the file, switching browsers or opening it on another device starts with an empty Daybook. Use Export and Import to move your data.

---

## Security and data

| What | How it's handled |
|---|---|
| Passcodes | Hashed with PBKDF2-SHA-256 (150,000 iterations, random 16-byte salt per account). Never stored in plain text. |
| Your data | Saved **unencrypted** in `localStorage`. Anyone using the same browser profile can read it with the browser's developer tools. |
| Sign-in errors | The same message for a wrong username or a wrong passcode, so the form doesn't reveal which usernames exist. |
| Sync | None. Data stays in one browser on one device. |
| Loss | Clearing browser data or using private browsing erases it. **Export regularly.** |

The mock login keeps a casual user out of someone else's Daybook on a shared computer. It is **not** an access-control system. Don't store anything sensitive in it.

### Browser storage keys

| Key | Contents |
|---|---|
| `daybook:v1:users` | Accounts: `{ username: { salt, hash, display, created } }` |
| `daybook:v1:data:<username>` | That account's data (see [Data model](#data-model)) |
| `daybook:v1:session` | The signed-in username, only when "Keep me signed in" is checked. Otherwise it's kept in `sessionStorage`. |

---

## How the code is organized

Everything is in one file: HTML, then CSS in `<style>`, then JavaScript in `<script>`.

| Section (JavaScript) | Purpose |
|---|---|
| Time helpers | Every "today" is computed in **America/New_York** with `Intl.DateTimeFormat`. Dates are stored as `YYYY-MM-DD` strings, and date arithmetic uses UTC so a daylight-saving change can't shift a day. |
| **`MOCK AUTH + LOCAL STORAGE`** block | `MockAuth` (sign up, sign in, session, sign out) and `LocalStore` (load and save one account's data). **This is the only code that knows where data lives.** |
| Sign-in screen | Tab switching, form validation and starting or ending a session. |
| Saving | The UI updates the in-memory state `S` first, then calls `put()` or `del()`. Both save the whole state after a 250 ms pause. Open tabs on the same account stay in sync through the browser's `storage` event. |
| Rendering | `renderHabits`, `renderCalendar`, `renderTodos`, `renderDay` and `renderSummary` rebuild their panel from `S`. |
| Reminders | `syncReminderTimer` and `showReminder`. |
| Event handlers | Forms, checkboxes and deletes. Deletes use a two-click "Confirm?" instead of browser pop-ups. |
| Midnight rollover | Every 20 seconds, checks whether the Eastern Time date has changed and resets the habit view if so. |

### Data model

```js
{
  events:   [{ id, kind: "event", date: "2026-11-12", title, start: "17:00", end: "19:30", location, notes, created }],
  habits:   [{ id, kind: "habit", name, created }],
  todos:    [{ id, kind: "todo", text, priority: "high|med|low", importance: "high|med|low",
               effort: "quick|medium|big", due: "2026-11-15" | "", done, doneAt, created }],
  logs:     { "2026-11-09": { "<habitId>": true, ... } },   // one entry per day: which habits were checked
  settings: { remindEvery: 30 }                              // minutes; 0 = off
}
```

Habit totals, 7-day counts and streaks are **calculated** from `logs` each time they're displayed, never stored. Deleting a habit hides its history from the summary but leaves the log entries in place.

---

## Replacing the mock login with a real service

The app was structured so this swap is limited to one block. To move to a hosted login and database such as Firebase Authentication with Firestore, or Supabase:

1. Replace `MockAuth` with calls to the service's sign-up, sign-in, sign-out and current-user functions. Keep the method names, or update the few places that call them in the sign-in screen code.
2. Replace `LocalStore.load(user)` and `LocalStore.save(user, state)` with reads and writes to the service's database, keyed by the authenticated user ID.
3. Make `load` asynchronous and `await` it in `startSession`.
4. Add database security rules so each user can only read and write their own records.
5. Optionally, replace the whole-state save with per-item writes, which the `put(id, data)` / `del(id)` calls are already shaped for.

Trade-offs to weigh. These were checked in November 2026, so confirm current pricing first:
- **Firebase (Spark plan):** free with no payment method required. Email and Google sign-in, and daily read and write quotas far above what one planner needs. It doesn't pause.
- **Supabase (Free plan):** a Postgres database. Free projects **pause after 7 days without a request** until manually restored, which is a problem for a daily tracker over school breaks.

---

## Known issues

- **Phone layout:** the habits panel stays fixed at the top and the calendar scrolls under it, on screens 760px wide or narrower.
- **Phone layout:** event chips are hidden on small screens, so days with events look empty until tapped.
- **Phone layout:** the monkey covers today's date number on small screens.
- **Tablet layout:** event chips get cut off ("5:00 P…").
- **Phone layout:** the header stays on screen while scrolling and takes about 110px.
- **Reminders only appear while the page is open.** There are no system or push notifications.
- **The reset countdown** can be off by an hour on the two daylight-saving changeover days. The reset itself still happens at the correct midnight.

---

## Browser support

Any current browser with `localStorage`, `Intl.DateTimeFormat` time-zone support and the Web Crypto API (`crypto.subtle`). If storage is blocked (some private-browsing modes) or Web Crypto is unavailable, the app shows a message instead of failing silently.

## Fonts

DM Sans and Fraunces load from Google Fonts. Offline, the app falls back to system fonts and still works.
