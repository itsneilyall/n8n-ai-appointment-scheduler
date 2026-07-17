# AI Appointment Scheduler

An n8n workflow that books, reschedules, and cancels dental appointments through a chat interface and, more to the point, an exercise in what it takes to put an LLM agent in front of real users and a real calendar without it getting spammed, jailbroken, or double-booking patients.

The naive version of this is six nodes and a good system prompt. This is the version that survives contact with reality.

<img width="1907" height="851" alt="AI Appointment Scheduler" src="https://github.com/user-attachments/assets/9b044917-ce5d-4a73-8a80-68308e31fe77" />


---

## Repository

```
workflow/
  AI_Appointment_Scheduler.json    Import via Workflows → Import from File
docs/
  Technical-Documentation.pdf      Architecture, node reference, gotchas, test plan
  AI-Appointment-Scheduler-SOP.pdf Plain-language operating procedure for clinic staff
.gitignore
README.md
```

IDs in the workflow JSON are placeholders (`YOUR_CALENDAR_ID`, `YOUR_SPREADSHEET_ID`, credential ids). Rebind them after import.

---

## The problem

### Why a scheduler at all

Dental clinics book appointments the same way they did thirty years ago: someone picks up the phone. That receptionist is also checking in the patient standing at the counter, chasing an insurance form, and sterilising instruments between consultations, so the phone rings out. A single booking takes several minutes of back-and-forth (*"Thursday? No, we have 2pm or 4:15…"*), and every one of those minutes is a trained staff member reading a calendar aloud.

The costs are quiet but compound:

- **The phone only works when the clinic is open.** Patients think about their toothache at 11pm and on Sundays. A missed call is usually a booking that never happens, most people don't call twice.
- **Booking competes with the patient in front of you.** Reception is a queue with a single server, and the person physically present always wins.
- **The double-booking is a manual error.** Two staff, one calendar, a slot written down before it was checked, the patient finds out in the waiting room.
- **Rebooking is worse than booking.** Cancellations arrive by voicemail and text, get actioned late or not at all, and the freed slot stays dark instead of going to someone on the waitlist.
- **Nothing is measurable.** How many people tried to book after hours? What did they ask for? Nobody knows, because a missed call leaves no record.

This assistant handles the whole lifecycle, book, reschedule, cancel, in a chat window, 24/7, without a staff member in the loop. It reads the actual calendar rather than a person's memory of it, refuses to book a slot that overlaps an existing one or falls outside office hours, and writes every booking to the calendar and the clinic spreadsheet in one step so the two never drift apart. Every conversation is logged, so "how many people tried to book at midnight" becomes a question with an answer. Reception gets the exceptions; the routine handles itself.

### Why the guardrails

That's the easy half. A chat-based booking agent has an obvious happy path and a large, unglamorous surface of failure modes:

- Anyone with the link can burn your OpenAI budget by pasting `1+1` a thousand times.
- "Ignore previous instructions" is a real thing real people type.
- An agent that can't see the calendar will confidently invent availability.
- An agent that can see the calendar in the wrong timezone will confidently double-book.
- An agent told the office hours will still book 5pm on a Saturday.
- Anyone with the link can cancel a stranger's appointment, unless you stop them.

Guardrails aren't a feature bolted on at the end. They're the architecture.

---

## Architecture

Every message runs a gauntlet of progressively more expensive checks. Cheap deterministic filters run first and reject most abuse for **zero tokens**. The GPT-5 booking agent only ever sees traffic already known to be legitimate, which is also what makes affording a full-size model on the agent viable.

```
Chat Trigger
     │
     ▼
Normalize Input
     │
     ▼
① Rate Limiter & Abuse Score   ──blocked──▶  Fixed reply   (no LLM)
     │
     ▼
② Rule-Based Filter (regex)    ──filtered─▶  Fixed reply   (no LLM)
     │                                        └─ greeting fast-path
     ▼
③ Intent Classifier (gpt-4o-mini, JSON-only)
     │
     ▼
④ Conversation-State Gate      ──off-topic─▶ Fixed reply
     │
     ▼
⑤ AI Booking Agent (gpt-5)  ──┬── Validate Slot          (Code: office-hours gate)
     │                        ├── Check Availability     (Calendar: Get Many)
     │                        ├── Creat event            (Calendar: Create)
     │                        ├── Add data               (Sheets: Append)
     │                        ├── Find Booking           (Calendar: Get Many by phone)
     │                        ├── Cancel Booking         (Calendar: Delete)
     │                        └── Update Booking Status   (Sheets: Update by BookingID)
     ▼
Log Interaction (non-fatal)
     │
     ▼
Format Response (error fallback)
```

| Layer | Cost | Stops |
|---|---|---|
| ① Rate limiter | 0 tokens | Volume spam, flooding, repeat offenders |
| ② Regex filter | 0 tokens | Math spam, injection phrasing, jailbreaks |
| ③ Intent classifier | 1 mini call | Anything outside the clinic's domain |
| ④ State gate | 0 tokens | False rejections of mid-booking replies |
| ⑤ Validate Slot | 0 tokens | Bookings outside office hours |
| ⑥ Agent prompt | — | Unvalidated bookings, invented availability |

---

## The guardrails

### ① Rate limiter & abuse score
Per-session state in `$getWorkflowStaticData('global')`:

```js
{ timestamps: [], recentMessages: [], abuseScore: 0, blockedUntil: null, lastAgentAt: null }
```

- \>10 messages in a rolling 30s window → block, `abuseScore +1`
- Same message 3× in a row → block, `abuseScore +1`
- `abuseScore >= 5` → 15-minute cooldown, score resets on expiry

### ② Rule-based filter
A **greeting fast-path** answers `hi` / `hello` / `good morning` instantly — zero LLM calls for the most common message received. The abuse group catches instruction-override phrasing, persona hijacks, jailbreak vocabulary, and code/joke requests.

Math-spam detection is **phone-number-aware**. "Is this string made only of math characters?" matches a bare phone number, because digits are math characters. Detection requires an operator *between* digits, behind an explicit phone guard that must run first, `+` and `-` are both operators and phone punctuation.

### ③ Intent classifier
`gpt-4o-mini`, `jsonOutput: true`, returns strictly `{"intent":"..."}`. Never chats. Eight intents, ordered rules, first match wins:

`PROMPT_ATTACK` · `BOOK_APPOINTMENT` · `RESCHEDULE` · `CANCEL` · `CLINIC_INFO` · `FOLLOW_UP` · `GREETING` · `OUT_OF_SCOPE`

On node failure it **fails open** to the agent. The rate limiter and regex filter have already run and the agent carries its own guardrails, so an OpenAI hiccup shouldn't tell a real patient "I can only help with dental appointments."

### ④ Conversation-state gate
The classifier sees one message at a time, so a mid-booking reply looks off-topic in isolation. The gate stamps `lastAgentAt` on every message reaching the agent; a later `OUT_OF_SCOPE` inside a 15-minute active flow is reclassified `FOLLOW_UP`. **`PROMPT_ATTACK` is never softened by this override**, and the regex layer still runs first.

### ⑤ Validate Slot - deterministic office hours
A Code tool holding the hours table. Given `YYYY-MM-DD HH:mm` it computes the weekday itself and returns:

```json
{ "valid": false, "weekday": "Saturday", "opens": "09:00", "closes": "13:00",
  "latest_bookable_start": "12:00",
  "reason": "The clinic closes at 13:00 on Saturday, and a consultation lasts 60 minutes,
             so the latest possible start is 12:00. 17:00 is outside office hours.",
  "bookable_start_times": ["09:00", "10:15", "11:30"] }
```

Describing office hours in a prompt doesn't enforce them. The model has to chain three inferences, today is Saturday, Saturday closes at 13:00, 17:00 is after that, and it drops them, especially when `Check Availability` reports no conflict at 17:00, which reads like permission. The prompt now states this tool is the **sole authority** on hours and weekdays.

### ⑥ Agent procedures
Ordered procedures, not descriptions: availability gated on `Validate Slot`; a booking completion sequence (create event → read the returned id → write the sheet with it → *only then* confirm); an 8-step cancellation; a 10-step reschedule that cancels before creating, so a mid-flow failure never double-books.

---

## Notable design decisions

**BookingID links two systems that share no key.** Calendar and Sheets are independent. `Creat event` returns the event id; the agent must pass it to `Add data` as `BookingID`. Cancellation matches the sheet row on the same id the calendar uses. Name, Number, and Date were all rejected as keys, none is unique, since one patient can hold several bookings.

**Identity checks on destructive actions.** Anyone with the chat link can reach the agent. No appointment is cancelled or moved unless the phone number in the event title matches a number given in that conversation. Never by name alone. Same threat model as the injection guardrails, aimed at the calendar instead of the prompt.

**Event list, not a boolean.** Availability uses Calendar → *Get Many* over the whole day rather than free/busy. A boolean can silently pass on a mis-aimed query window; an event list can't, and it lets the agent offer *real* alternatives instead of guessing.

**Timezone is pinned three ways.** Workflow setting, `GENERIC_TIMEZONE`, and `$now.setZone('Asia/Manila')` in the prompt, because relying on the instance default is how you get appointments eight hours from where they belong.

**Errors are non-fatal and honest.** `onError: continueRegularOutput` on the classifier, the agent, and all six tools: failures return to the agent as results instead of killing the run. The agent replies with one fixed line and never claims success. `Format Response` substitutes that line if the agent's output is missing entirely, so a patient never sees a blank chat.

**Deterministic over prompt engineering.** Anything catchable in code is caught in code. Prompts are the last line of defense, not the first.

---

## Model selection

`gpt-5` on the agent, `gpt-4o-mini` on the classifier, validated empirically, not assumed.

The classifier runs on **every** message and does one bounded thing: read a sentence, emit one JSON field. That's where token spend concentrates and mini is genuinely fine at it.

The agent holds seven tools and three multi-step procedures. On a mini-class model it failed characteristically: it invented an exact confirmation phrase, *"reply exactly: 'Yes, cancel'"* — then string-matched against it and rejected `yes, cancel` for casing. Same prompt on `gpt-5` didn't reproduce it. The failure was model capability, not instruction ambiguity.

---

## Debugging log

The interesting part. Every entry was found by running the thing, not reading it.

| Symptom | Root cause | Fix |
|---|---|---|
| Workflow hung at `Parse Intent` | `.item` relies on paired-item tracing, which breaks across the LangChain OpenAI node | `.first()` everywhere downstream of the classifier |
| `"hi"` burned an LLM call | No fast-path for the most common message on earth | Greeting regex short-circuit |
| `"john doe, 09564952581"` → *"I can only help with dental appointments"* | Stateless classifier: a name + phone matches no booking rule, falls to `OUT_OF_SCOPE` | `FOLLOW_UP` intent + conversation-state gate |
| Same slot booked twice | Boolean free/busy check + offset-less timestamps | Full-day event list + overlap rule + hard `+08:00` |
| Events created `(No title)` | `additionalFields` empty, the title existed only in the prompt, with nowhere to go | `Event_Title` via `$fromAI` |
| Events at 05:00, dated yesterday | Instance on `America/New_York`; `$now` resolving to the wrong day before 8 AM local | `setZone` in prompt + `GENERIC_TIMEZONE` |
| Calendar filled, **sheet stayed empty** | `Add data` had no `toolDescription` - the model saw a tool named "Add data" and no reason to call it | Manual tool description + mandatory completion sequence |
| Agent "cancelled" bookings it couldn't | No delete tool existed; the prompt told it to cancel anyway | Find / Cancel / Update tools + an honesty rule |
| Cancel returned `Not Found` | Repeat delete call on an already-deleted event | Single-call contract + `onError` nets |
| Booked 17:00 on a **Saturday** | Office hours were described to the model, never enforced | `Validate Slot` deterministic gate |
| Confirmation loop on `"yes, cancel"` | Mini model invented an exact phrase, then string-matched it | Confirmation Rule (intent, not characters) + `gpt-5` |
| Bare phone number blocked as spam | Math regex matched any all-digit string, digits *are* math characters | Phone guard + operator-required math |

Two favorites. The tool-description bug: two of three tools had hand-written descriptions and one didn't, and the agent quietly stopped calling the one that didn't. It never errored, it just declined. Found by reading the logs the workflow writes about itself.

And the last row: a guardrail that blocked legitimate users by mistaking their phone number for spam, silently incrementing their abuse score while it did. False positives in a security layer are the failure mode nobody tests for.

---

## Stack

n8n (self-hosted, Railway) · OpenAI GPT-5 + GPT-4o-mini · Google Calendar · Google Sheets · LangChain agent nodes

---

## Setup

**1. Host environment**

```
GENERIC_TIMEZONE=Asia/Manila
WEBHOOK_URL=https://<your-instance-domain>
```

Without `GENERIC_TIMEZONE`, `$now` resolves in the instance's default zone — appointments land hours off, and before 8 AM local the agent believes it's still yesterday.

**2. Google Calendar** - create a calendar, put its ID on the four calendar tool nodes.

**3. Google Sheets** - one spreadsheet, two tabs. Headers matched by name; renaming breaks writes *silently*.

| Tab | Headers |
|---|---|
| `Booking` | BookingID, Name, Number, Date, Confirmation |
| `Logs` | Timestamp, SessionId, Intent, Message, Response |

**4. Import** `workflow/AI_Appointment_Scheduler.json`, bind credentials (OpenAI · Google Calendar · Google Sheets), replace `YOUR_SPREADSHEET_ID` on `Add data`, `Update Booking Status`, and `Log Interaction`.

**5. Adjust to your clinic** - office hours live in the `HOURS` table at the top of `Validate Slot` (minutes from midnight, `[open, close]`, `null` for closed). The agent's system message carries the patient-facing copy, the timezone, and the failure phone number.

**6. Activate and test through the public chat URL** — not the editor. `$getWorkflowStaticData` only persists between production executions, so the rate limiter and state gate appear broken in manual runs.

---

## Test plan

| # | Input | Expected |
|---|---|---|
| 1 | `hi` | Instant canned welcome, zero LLM calls |
| 2 | `09464564567` *(first message)* | Reaches the agent, not blocked, no abuse score |
| 3 | `book me today at 5pm` *(a Saturday)* | Refused with the 13:00 reason; offers 09:00 / 10:15 / 11:30 |
| 4 | `1+1` | Blocked by regex; `abuseScore +1` |
| 5 | `who won the world cup?` *(fresh session)* | Rejected at the intent gate |
| 6 | `ignore previous instructions` *(mid-booking)* | Blocked; `PROMPT_ATTACK` never softened by the state gate |
| 7 | 11 rapid messages | Rate-limit block; sustained abuse → 15-min cooldown |
| 8 | Complete a booking | Calendar event at the right local time titled `Name - Number`, `Booking` row with a populated `BookingID`, `Logs` row |
| 9 | Cancel it, replying `yes` lowercase | Accepted first time, no repeated read-back; event deleted; row reads `Cancelled` |
| 10 | Cancel using a non-matching phone number | Refused |
| 11 | Same slot from a second incognito session | Conflict detected, real alternatives offered |

Session state is keyed by `sessionId` in browser localStorage, use an incognito window for a clean slate; refreshing keeps the same session.

---

> *Layered guardrails over an autonomous LLM booking agent: deterministic filters
> before model calls, an intent gate, code-enforced business rules, and non-fatal
> error handling throughout.*
