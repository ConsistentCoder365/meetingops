# MeetingOps – AI PMO Copilot: PRD

Oct 25, 2026 · @Joy Kukreja

## Overview

MeetingOps turns meeting transcripts into a live project tracker, with a person approving every AI suggestion before it is saved. It targets the PMO work that eats a project coordinator's week: writing up action items, chasing owners, and assembling the weekly status report.

**Problem.** After a meeting, someone rewrites the notes into tasks, updates a tracker by hand, and later rebuilds a status report from that tracker. Items get lost, owners are unclear, and slipping deadlines surface late.

**Product.** Claude extracts action items, owners, deadlines, decisions and risks from a transcript as structured data. A reviewer accepts, edits or rejects each one. Approved items feed a dashboard and an auto-drafted weekly report, and every reviewer correction is logged as evaluation data.

**Why it matters.** It shows AI-human collaboration in a real operations workflow, with measured accuracy rather than a demo. That maps directly onto roles such as PwC's Business AI Engineer Intern (tracking and reporting, data analysis, change communications).

## Goals and non-goals

**Goals**

- Extract action items, owners, deadlines, decisions and risks from a transcript with measurable recall and precision.
- Keep a human in control: nothing reaches the tracker without explicit approval.
- Give a single view of every initiative's status, overdue work and owner workload.
- Cut the time to produce a weekly stakeholder report and change communications.
- Turn reviewer corrections into an eval set that drives prompt and tool improvements.

**Non-goals**

- Live audio capture or speech-to-text; input is a transcript or notes file.
- Replacing a full project management suite (Jira, Asana, MS Project).
- Automatically sending emails or calendar invites without a person confirming.
- Multi-tenant SaaS billing, SSO or enterprise admin features.

## Users and personas

| Persona | Role | Main need | Uses |
| --- | --- | --- | --- |
| Priya, project coordinator | Runs 6–10 meetings a week across teams | Turn meetings into tracked tasks fast, without missing owners or dates | Upload, review queue, scheduling |
| Arjun, engagement manager | Owns a portfolio of initiatives | See what is at risk or late before the weekly review | Dashboard, weekly report |
| Task owner | Assigned action items | Know what they owe and by when | Owner view, status updates |
| Stakeholder | Senior lead outside the team | A short, accurate update and clear notice of decisions affecting them | Weekly report, change communications |

## Core user flows

&#91;embedded content: core flow · 7 steps, 1 human decision\]

Every suggestion stops at the reviewer. Approved items flow into the tracker and reports, and every decision, including rejects, becomes eval data that improves the next extraction.

Two secondary flows sit beside this loop. A manager opens the dashboard, filters to at-risk initiatives and exports the weekly report. A coordinator picks an approved follow-up action and asks the scheduling helper for slots that suit every attendee's time zone.

## Functional requirements

**F1. Meeting ingestion**

- Upload a transcript as .txt, .vtt, .docx or pasted text, tagged to an initiative.
- Parse speakers and timestamps when present; split long transcripts into chunks under the model's context budget.
- Store the raw transcript so every extracted item can link back to its source line.

**F2. AI extraction**

- Claude, via tool calls with a strict JSON schema, returns items of type action, decision or risk.
- Each item has: text, owner (matched to a known person or flagged unknown), due date (absolute, resolved from phrases like "next Friday"), priority, source quote and a confidence score.
- Items with a missing owner or date are flagged for the reviewer, never guessed silently.
- Detect duplicates of open items already in the tracker and propose an update instead of a new item.

**F3. Human review queue**

- Show each suggestion beside its source quote, with Accept, Edit and Reject.
- Bulk-accept items above a confidence threshold the reviewer sets.
- Nothing is written to the tracker until approved.
- Log every decision: original suggestion, final version, field-level diff, reviewer and time.

**F4. Tracker and dashboard**

- Initiatives contain items; each item has status (open, in progress, done, blocked) and history.
- Initiative health is computed as on-track, at-risk or late from overdue and due-soon items, with the rule shown on hover.
- Dashboard shows overdue rate, items due this week, owner workload, and date slippage per initiative over time.
- Filters by initiative, owner, status and date; export to CSV and Excel.

**F5. Weekly status report**

- Auto-draft a one-page report per initiative or portfolio: health, completed, upcoming, risks, decisions.
- Every statement in the draft links to the tracker items it came from.
- Editable before export to PDF, .docx or email text; the system never sends it on its own.

**F6. Change communications**

- For an approved decision that affects other teams, draft a short notice: what changed, why, who is affected, what they need to do.
- The user picks the audience and confirms before copying or sending.

**F7. Scheduling helper**

- Propose follow-up meeting slots across attendees' time zones (for example IST, UK, US Eastern) within working hours.
- Optional Outlook calendar integration through the Microsoft Graph API to read free/busy and create a draft invite.

**F8. Correction logging and evals**

- Turn review logs into labelled eval cases automatically.
- An eval runner scores each prompt or model version against the AMI test set and the correction set; a change ships only if it does not lower recall.

## Data model

PostgreSQL, one schema. The review log is the most important table: it is both the audit trail and the eval dataset.

| Table | Key columns | Notes |
| --- | --- | --- |
| `initiatives` | id, name, owner\_id, start\_date, target\_date, health | health is derived, cached nightly |
| `people` | id, name, email, timezone, aliases\[\] | aliases help match "Priya" or "PK" in transcripts |
| `meetings` | id, initiative\_id, title, held\_at, transcript\_text, source\_format | raw transcript kept for source links |
| `suggestions` | id, meeting\_id, type, text, owner\_id, due\_date, priority, source\_quote, confidence, model\_version, state | state: pending, accepted, edited, rejected |
| `items` | id, initiative\_id, suggestion\_id, type, text, owner\_id, due\_date, status, created\_at | only approved suggestions become items |
| `item_history` | id, item\_id, field, old\_value, new\_value, changed\_at | powers the slippage chart |
| `review_log` | id, suggestion\_id, reviewer\_id, action, before\_json, after\_json, diff\_fields\[\], seconds\_to\_review | feeds the eval set |
| `reports` | id, initiative\_id, period\_start, draft\_md, final\_md, prep\_seconds | prep\_seconds measures report time |
| `eval_runs` | id, prompt\_version, model\_version, dataset, precision, recall, f1, run\_at | one row per eval run |

## System architecture and tech stack

&#91;embedded content: architecture · web app, 5 backend modules, 4 dependencies\]

The web app only talks to the backend. The backend owns every call to Claude, Postgres and Microsoft Graph, and the eval runner runs in CI against the same database.

| Layer | Choice | Why |
| --- | --- | --- |
| Frontend | Next.js, TypeScript, Tailwind CSS | matches the CV stack; fast review-queue UI |
| Backend | Python, FastAPI, Pydantic | tool schemas double as Pydantic models |
| LLM | Claude API with tool use | structured extraction with typed outputs |
| Database | PostgreSQL | relational items, history and audit log |
| Calendar | Microsoft Graph API (optional) | Outlook free/busy and draft invites |
| Packaging | Docker Compose, GitHub Actions | one-command setup; eval gate in CI |

## Evaluation plan and success metrics

Each metric below fills one `[X]` on the CV, so measure them exactly this way and record the run in `eval_runs`.

**Dataset.** The public [AMI Meeting Corpus](https://groups.inf.ed.ac.uk/ami/corpus/) includes human-written abstractive summaries with "actions" and "decisions" sections. Use those as the reference labels. Hold out a fixed test split of 20–30 meetings and never tune prompts on it.

**Matching rule.** An extracted item counts as a match when it refers to the same task as a reference action. Score matches with an LLM judge, then hand-check a random 20% of the judge's decisions and report the agreement rate.

| Metric | How it is measured | CV placeholder |
| --- | --- | --- |
| Action-item recall | matched reference actions ÷ all reference actions, on the AMI test split | "\[X\]% recall on the AMI Meeting Corpus" |
| Action-item precision | matched extracted items ÷ all extracted items | not on CV; report alongside recall |
| Accepted unedited | suggestions accepted with no field changed ÷ all reviewed suggestions, from `review_log` | "\[X\]% of items accepted by reviewers unedited" |
| Report prep time, before | minutes to write a weekly report by hand from the tracker, timed over at least 5 reports | "from \[X\]" |
| Report prep time, after | minutes from opening the auto-draft to a final report, from `reports.prep_seconds` | "to \[X\] min" |
| Owner and date accuracy | share of matched items whose owner and due date are correct | supporting metric |

**Who reviews.** The accepted-unedited metric needs reviewers other than you. Recruit 3–5 people (classmates, DebSoc or Debugging Society members) and have each review at least 2 meetings.

## Non-functional requirements

| Area | Requirement |
| --- | --- |
| Latency | Suggestions for a 60-minute transcript ready in under 60 seconds; dashboard pages load in under 1 second |
| Cost | Track tokens per meeting in the DB; cache the system prompt to keep a meeting's extraction cost low |
| Reliability | Every tool-call response is validated against the schema; one retry on invalid output, then the chunk is flagged for review |
| Privacy | Transcripts stay in your own Postgres; personal data is never put into logs; one-click deletion of a meeting and its items |
| Auditability | Every tracker item traces back to a suggestion, a reviewer decision and a source quote |
| Security | Role-based access (coordinator, manager, viewer); API keys held server-side only |
| Reproducibility | Prompt and model version stored on every suggestion and eval run; the eval runs in CI |
| Accessibility | Keyboard-only review queue (A accept, E edit, R reject); WCAG AA contrast |

## Risks and open questions

**Risks**

- **AMI labels are loose.** Reference actions are short summaries, not exact task lists, so recall depends on the matching rule. Mitigation: publish the rule and the judge-agreement rate with the result.
- **Owner resolution.** Transcripts use first names, nicknames or "you". Mitigation: alias table plus a forced "unknown owner" flag rather than a guess.
- **Relative dates.** "End of next week" depends on the meeting date. Mitigation: resolve against `meetings.held_at` and show the resolved date in review.
- **Too few reviewers.** The accepted-unedited number is weak if only you review. Mitigation: recruit 3–5 outside reviewers before quoting it.
- **Report-time baseline.** Before/after timing is easy to bias. Mitigation: time the manual baseline first, on different weeks from the assisted runs.

**Open questions**

- [ ] Ship the Outlook integration, or keep scheduling as an in-app suggestion only?
- [ ] Which AMI meetings form the held-out test split?
- [ ] Use Claude as the match judge, or a cheaper model checked against the hand-labelled 20%?
- [ ] Deploy a public demo with sample data, or keep it a local Docker setup with a video walkthrough?
