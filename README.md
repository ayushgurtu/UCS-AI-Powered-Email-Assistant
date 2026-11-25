# HWB UCS AI-Powered Email Assistant

AI-Flow (n8n) workflows that help the University Counselling Services (UCS) front desk answer email sent to the shared mailbox. The automation classifies each new email, searches past UCS replies for precedents, applies current operating hours and the consult roster, and prepares a reply or a forward for a front-desk officer to review.

> **The automation never sends an email.** It only creates drafts. Every draft must be reviewed by a person before it is sent. It is not a crisis-response service and does not monitor email for emergencies.

## Contents

- [What it does](#what-it-does)
- [Workflows in this folder](#workflows-in-this-folder)
- [How it works](#how-it-works)
- [Classification](#classification)
- [Draft routing](#draft-routing)
- [Outputs](#outputs)
- [Exception handling](#exception-handling)
- [Deployment](#deployment)
- [Business user guide](#business-user-guide)
- [Making simple changes](#making-simple-changes)
- [Known limitations](#known-limitations)

## What it does

- Reads new, unread emails without attachments from the UCS mailbox
- Skips automated mail: out-of-office replies, bounces, mailing lists, and blocked senders
- Recovers the real question from forwarded emails and collects earlier messages in the thread
- Decides whether an email needs a clinician (consult); otherwise classifies it into one of eight intents
- Searches past UCS replies for the same sender role and intent
- Applies current operating hours, closures, and the consult roster from Dropbox files
- Answers several questions in one email separately, then combines them into one draft
- Prepares a draft in UCS house style and records every draft in an Excel log on OneDrive
- Keeps the library of past replies current and removes records older than two years

## Workflows in this folder

| File | Starts on | Role |
|---|---|---|
| `WF-2 Inbound Draft Generator (Forward-Aware, Batched, Query Rewriting & Rephrasing).json` | Outlook check every minute, or manual test | Main workflow. Prepares and classifies each email, routes it to a branch, writes the draft log. |
| `WF-2a Sub-Intent Draft Branch (Sub-workflow).json` | Called by WF-2 | Drafts single-question emails for the appointment and general information intents. |
| `WF-2b Multi-Question Draft Branch (Sub-workflow).json` | Called by WF-2 | Answers each question separately and combines the answers into one draft. |
| `WF-1 Historical Sent-Email Indexer (Scheduled Trigger - Parent Orchestrator).json` | Schedule | Finds sent replies since the last run in 15-day windows and exports an Excel log per window. |
| `WF-1b · Reply Processor (Extract → Classify → Embed).json` | Called by WF-1 | Pairs each sent reply with its question, cleans and classifies the pair, adds it to the library. |
| `WF-RETENTION - Purge records greater than 2 yrs.json` | Schedule | Deletes library records whose reply is more than two years old. |
| `Error Workflow/WF-ERR - UCS Email Assistant Error Handler.json` | Automatically on failure | Sends a formatted Outlook alert when a linked workflow fails. |

Not included here: **WF-Deletion Azure AI Search Index Deletion**, a manual, administrator-only workflow that deletes the whole library index after a typed-name confirmation. Connection tests, retrieval tests, the Excel merge utility, prompt tests, and earlier WF-1, WF-1a, and WF-2 versions are not part of the operating flow.

Credentials have been removed from all workflow files. See [Deployment](#deployment).

## How it works

### Drafting a new email (WF-2, WF-2a, WF-2b)

| Step | What happens |
|---|---|
| 1. Email pickup | Unread Inbox emails without attachments are collected. Blocked senders and emails with `ucs@nus.edu.sg` in CC are removed. |
| 2. Daily log folder | Finds or creates a dated *AI Draft Email Logs* folder on OneDrive. |
| 3. Automated-mail check | Headers, subject, and sender are checked. Auto-replies, bounces, read receipts, calendar responses, and bulk mail are skipped. |
| 4. Thread history | Up to five recent messages from the same conversation are collected from Inbox and Sent Items. |
| 5. Question extraction | Signatures and notices are removed. For a forwarded email, AI decides whether the question is in the covering note or the quoted message. |
| 6. Consult decision | A clinician named in the email is matched against the UCS roster. AI decides whether the front desk can close the matter or a clinician must take it. |
| 7. Intent classification | If no consult is needed, AI assigns one of eight intents. Sender role comes from the email domain. |
| 8. Operating hours | `UCS_OperatingHours.xlsx` is read from Dropbox and resolved for today and the current Monday–Friday week. |
| 9. Question splitting | Longer emails in five of the intents are checked for separate topics. Emails with more than one topic go to WF-2b. |
| 10. Search query | A short thread reply or a very long email is restated as one clear question for searching. The draft still responds to the original email. |
| 11. Precedent search | Up to six past question-and-reply pairs with the same sender role and intent are retrieved. More recent replies get a ranking boost. |
| 12. Draft and alignment | AI writes a reply or a colleague handover note. Some replies are aligned to approved standard wording. |
| 13. Formatting | Code, not AI, adds the greeting, classification line, signature, and confidentiality notice, and inserts weekly hours where required. |
| 14. Outlook draft and log | The Outlook draft step runs when enabled. A row for each email is written to the Excel draft log. |

### Updating the library of past replies (WF-1, WF-1b)

| Step | What happens |
|---|---|
| 1. Index check | WF-1 confirms the Azure AI Search index exists and creates it if missing. |
| 2. Time windows | The last run time is read from the *Indexer State* data table. The period up to now is split into 15-day windows. |
| 3. Sent replies | Sent Items without attachments are collected for each window and passed to WF-1b. |
| 4. Pairing | Each reply is matched with the newest earlier inbound message in the same conversation. |
| 5. Cleaning and labelling | AI cleans both emails and applies the same consult decision and intent classification used for new mail. |
| 6. Indexing | Paired records are added to the library. Replies with no matching question are logged but not indexed. |
| 7. Run log | An Excel file for the window is uploaded to a dated *Email Ingestion Export* folder on OneDrive, and the last run time is updated. |

### Inputs

| Item | Expected value |
|---|---|
| Live input | Unread emails in the UCS Inbox, without attachments |
| Test input | Unread emails in the `Inbox/AI Email Testing` folder, through the manual trigger |
| Library source | Sent Items replies paired with the inbound question in the same conversation |
| `UCS_OperatingHours.xlsx` (Dropbox) | Sheets: *Holiday Dates*, *Office Closure Dates*, *Soft Closure Periods*, *General Operating Hours* |
| `UCS_ConsultSchedule.xlsx` (Dropbox) | One sheet per month (e.g. `Oct 2026`, `March 2026`) with Date, Day, Consult Name, Consult Email, Covering Consult Name, Covering Consult Email, Remarks |
| Blocked senders | Set in the Outlook query and *Filter - Exclude Own Mailbox* nodes |
| Library retention | Replies from the last two years |

## Classification

Every email gets a sender role and one intent. The consult decision runs first, so an email marked as a consult never reaches the intent classifier.

| Intent | Typical email | Draft path |
|---|---|---|
| `consult_required` | A named clinician's session, a message for the counsellor, an open case, clinician-owned work, a change of counsellor, or a clinical or referral issue | Forward to consult of the day |
| `appointment` | Booking, rescheduling, cancelling, counselling applications, logistics of a booked session | WF-2a scenarios |
| `assessment_request` | Fitness to Resume Studies request, documents, or status | WF-2a |
| `service_clarification` | What UCS offers, eligibility, fees, confidentiality, Intellect services | WF-2a |
| `operational_info` | Location, hours, contacts, forms, routing to another service | WF-2a |
| `staff_referral` | A staff member or third party writing about another named student | WF-2a |
| `non_actionable` | Marketing, mass mail, recruitment solicitation, bare thanks | WF-2a |
| `volunteer_internship` | Seeking volunteering, an internship, a practicum, or a placement | Forward to colleague |
| `partnership` | An external organisation proposing a collaboration, talk, event, or sponsorship | Forward to colleague |

| Sender role | Email domain |
|---|---|
| student | `u.nus.edu`, `u.yale-nus.edu.sg`, `u.duke.nus.edu` |
| visitor | `visitor.nus.edu.sg`, `partner.nus.edu.sg` |
| staff | `nus.edu.sg` (other than the student and visitor domains) |
| external | Any other domain |
| unknown | No usable address |

## Draft routing

Within WF-2a, each single-question email is first checked for Zoom. Other emails get one of eight scenarios. The scenario classifier also receives the sender role, which helps it spot staff referrals. When the classifier is unsure or fails, the email goes to the normal reply path, not to a forward.

| Scenario | When used | Draft produced |
|---|---|---|
| Zoom | The request concerns a Zoom link, ID, passcode, or joining problem (an online FTS assessment does not count) | Forward with handover note |
| Reschedule | An existing or proposed slot should move | "Intake assessment" mentioned: reply aligned to standard wording. Otherwise: forward to a colleague. |
| Booking | A new slot or walk-in is requested, or the sender asks about the booking portal | Staff sender: forward with a fixed note to `hwb@nus.edu.sg`. Others: if the sender has already used the portal or form, a reply answers the portal question without standard wording; otherwise a reply aligned to standard wording. |
| Application | Applying for counselling | Reply aligned to standard wording. For exchange students, the Student Portal link is removed. |
| FTS application | A Fitness to Stay assessment | Not booked: aligned reply. Booked: reply that promises no slot and always includes the standard sentences on earlier slots from cancellations. Completed: reply telling the sender to wait for the outcome, or answering what happens next. |
| Intellect credits | Credit balance, used credits, or paid Intellect registration | Reply aligned to standard wording |
| Tiering appeals | Routed elsewhere and asking for 1:1 UCS counselling | Reply aligned to standard wording |
| Referrals | Staff, a faculty, or a department refers a student for counselling support | Brief reply thanking the sender and confirming UCS will follow up. WF-2a only. |
| General | Cancellations, slot confirmations, ADHD or ASD enquiries, and anything else | Reply based on precedents. ADHD and ASD assessment enquiries get two fixed paragraphs explaining that assessments form part of counselling. |

**Multi-question emails.** WF-2b uses the same scenario paths for each question, without the Zoom check and without the Referrals scenario. Only staff booking and counselling reschedule questions count as routed to a colleague. The result is always one draft: if any question is routed to a colleague, the whole email becomes a forward using only the routed answers, and the first routed question decides the path. Otherwise the answers are merged into one continuous reply.

**Forwarded emails.** When a colleague forwards a student's email and the question is in the quoted message, the appointment reply branch can re-address the draft to the original enquirer, but only for student, visitor, or external addresses. A yellow note asks the officer to check the recipient.

**Consult forwards.** The recipient comes from today's roster. If today has no rostered consult, the next rostered date is used. A covering consult takes priority over the listed consult on the same row.

### What a draft looks like

| Element | Description |
|---|---|
| Subject | Replies use the normal reply subject. Forwards start with `FW:` after older `RE:` and `FW:` prefixes are removed. |
| Classification line | `[NUS Restricted]` at the top |
| Review note | A yellow note on forwarded-mail replies prompting a recipient check |
| Greeting | The sender's name, or `[Name]` if none is found. Forwards use *Dear Colleague*; consult forwards use the consult's name. |
| Body | A reply or handover note. Links are copied only from precedents, thread history, or approved wording. |
| Operating hours | Reschedule, booking, and tiering replies carry the current week's hours, inserted by code |
| Quoted original | When Outlook draft creation is enabled, the original email appears below the draft through Microsoft Graph |
| Signature and notice | Fixed UCS contact block, Lifeline number, and red confidentiality notice |

## Outputs

| Output | Content |
|---|---|
| AI Draft Email Logs (OneDrive, Excel) | One row per email: conversation ID, question subject, sender, received time, question text, reply subject, reply text, HTML body, sender role, intent, precedents, search query, rewrite mode, whether the query was rewritten, search filter, and precedent count, IDs, and scores |
| Email Ingestion Export (OneDrive, Excel) | One file per 15-day window with conversation ID, question and reply details, sender role, and intent. Long values are cut at 32,000 characters. |
| Azure AI Search index | The library of past question-and-reply pairs, searched by meaning and keywords and filtered by sender role and intent |
| Indexer State (data table) | The last successful run time used by WF-1 |

Each log run creates a new timestamped Excel file. The logs contain student email content and must be accessed and shared in line with UCS confidentiality practice.

## Exception handling

A failed AI step leads to a safe default wherever possible.

| Situation | Behaviour |
|---|---|
| Automated or bulk email | Skipped; no draft is created |
| Intent result is unusable | Falls back to `non_actionable` |
| Scenario result is unusable | Goes to the general reply path |
| Zoom or exchange check fails | Defaults to no |
| FTS stage is unclear | Defaults to not booked |
| Booking stage is unclear | Treated as a new booking request; gets the standard booking reply |
| Search returns no precedents | A brief holding reply is drafted from the email and thread |
| AI rewrite is empty or too long | The original email text is used for the search |
| Primary AI model is unavailable | The configured fallback model is used |
| Sender name cannot be identified | The greeting uses `[Name]` |
| No consult is rostered | The draft has no To: line; the officer must add the recipient |
| Library index is missing during purge | WF-RETENTION stops before deleting anything and reports an error |
| A linked workflow fails | WF-ERR sends an Outlook alert with the error details |

## Deployment

The folder includes a Docker Compose setup for a self-hosted n8n instance. n8n stores its data (workflows, credentials, executions, data tables) in a SQLite database in the `n8n_data` volume.

| File | Purpose |
|---|---|
| `docker-compose.yaml` | n8n service, plus an opt-in `n8n-import` service that loads the workflows |
| `.env` | Secrets and overrides. Holds `N8N_ENCRYPTION_KEY`. Not committed to git. |
| `.gitignore` | Keeps `.env`, local data, and credential exports out of version control |

### 1. Configure `.env`

`N8N_ENCRYPTION_KEY` encrypts every stored credential. Set it once, back it up, and never change it — changing or losing it makes existing credentials unreadable. To generate one:

```
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Optional overrides (defaults in brackets): `N8N_VERSION` [`latest`], `WEBHOOK_URL` [`http://localhost:5678/`], `N8N_HOST`, `N8N_PROTOCOL`, `TIMEZONE` [`Asia/Singapore`].

### 2. Start n8n and import the workflows

```
docker compose up -d
docker compose --profile import run --rm n8n-import
```

The import loads the error handler first, then the six main workflows, keeping their workflow IDs so sub-workflow calls and the error-workflow setting resolve. Imported workflows are inactive.

> Re-running the import overwrites workflows with the same ID, including any edits made in the editor. Only run it for first-time setup or to reset to the files in this folder.

### 3. Create credentials

Credentials are not included in the workflow files. Create these in n8n, then select them on every node that shows a credential warning (including **Send Alert Email** in WF-ERR):

| Credential type | Used for |
|---|---|
| Microsoft Outlook OAuth2 | Reading the mailbox, creating drafts, error alerts |
| Microsoft OneDrive OAuth2 | Draft logs and ingestion exports |
| Dropbox OAuth2 | `UCS_OperatingHours.xlsx` and `UCS_ConsultSchedule.xlsx` |
| Azure AI Search | The precedent library |
| OpenAI | Chat models and `text-embedding-3-large` embeddings |

For the OAuth2 credentials, register `<WEBHOOK_URL>rest/oauth2-credential/callback` (by default `http://localhost:5678/rest/oauth2-credential/callback`) as the redirect URI in the Azure and Dropbox app registrations.

### 4. Create variables and the data table

The workflows read these n8n Variables (`$vars`):

| Variable | Used in | Purpose |
|---|---|---|
| `AZURE_SEARCH_ENDPOINT` | WF-1, WF-1b, WF-2, WF-2a, WF-2b, WF-RETENTION | Azure AI Search service URL |
| `AZURE_SEARCH_INDEX` | WF-1, WF-1b, WF-2, WF-2a, WF-2b, WF-RETENTION | Library index name |
| `AZURE_SEARCH_API_VERSION` | WF-1, WF-1b, WF-2, WF-2a, WF-2b, WF-RETENTION | Azure AI Search REST API version |
| `global_ai_know_url` | WF-1b, WF-2, WF-2a, WF-2b | Base URL of the OpenAI-compatible gateway used for embeddings (`/completions/oai/embeddings`) |
| `PARTNERSHIP_FORWARD_TO` | WF-2 | Recipient for partnership forwards |
| `VOLUNTEER_FORWARD_TO` | WF-2 | Recipient for volunteer and internship forwards |
| `CONSULT_FORWARD_TO_FALLBACK` | WF-2 | Last-resort consult recipient when the roster has no entry |
| `ZOOM_FORWARD_TO` | WF-2a | Recipient for Zoom forwards |

> Variables are a licensed feature on self-hosted n8n. Check that your instance's plan includes them before going live; without them, `$vars` expressions resolve to nothing.

Also create the **Indexer State** data table used by WF-1 to store the last successful run time.

### 5. Activate

Test WF-2 with the manual trigger and the `Inbox/AI Email Testing` folder first. Then activate WF-ERR, WF-2, WF-1, and WF-RETENTION. WF-1b, WF-2a, and WF-2b are sub-workflows and are called automatically.

WF-1 and WF-RETENTION use the n8n default schedule (daily at midnight, in the `TIMEZONE` set in `.env`). Adjust their Schedule Trigger nodes if a different time is needed.

## Business user guide

1. **Keep the Dropbox files current.** Update holidays, closures, reduced hours, and the consult roster before they take effect. Drafts use these files on every run.
2. **Review every draft.** Check the recipient, facts, dates, links, and tone. Delete any yellow note and replace any `[Name]` placeholder before sending.
3. **Handle what the automation skips.** Emails with attachments, automated mail, and blocked senders receive no draft and need normal front-desk handling.
4. **Review the Excel draft log.** Use the daily log to see how each email was classified, what was drafted, and which precedents were used. Report recurring errors to the workflow owner.

### Responsibilities

1. Review, correct, and send or discard each draft.
2. Confirm recipients on all forwards and redirected replies.
3. Keep `UCS_OperatingHours.xlsx` and `UCS_ConsultSchedule.xlsx` accurate.
4. Approve the standard reply wording and supply any missing links.
5. Confirm who should receive each type of forward.
6. Tell the workflow owner when clinicians join or leave, or blocked senders change.
7. Report misclassified emails and poor drafts with examples.
8. Respond to or escalate any WF-ERR alert.

## Making simple changes

Before any change, duplicate or export the workflow as a backup. After the change, run one test with the manual trigger and the `Inbox/AI Email Testing` folder. Only make changes that are clear and business-approved. For Code node changes, use AI-Chat to generate the corrected code and paste it back into the node.

### Standard reply wording

Make the same change in **both WF-2a and WF-2b** so single- and multi-question drafts stay consistent.

| Change | Node |
|---|---|
| Intake assessment reschedule reply | Align Intake Reply (LLM) |
| Booking reply | Align Booking Reply (LLM) |
| Counselling application reply | Align Application Reply (LLM) |
| Tiering appeal reply | Align Tiering Appeals Reply (LLM) |
| Fitness to Stay reply | Align Tiering FTS Assessment (LLM) |
| Intellect credits reply | Align Tiering Intellect Credits (LLM) |
| ADHD and ASD assessment paragraphs | Draft Reply (LLM REST) (General) |
| Earlier-slot sentences for booked FTS assessments | Draft Reply (LLM REST) (FTS Booked) |

### How AI writes or classifies

| Change | Node |
|---|---|
| Tone or rules for a scenario draft | Draft Reply (LLM REST) node for that scenario, in WF-2a and WF-2b |
| Scenario definitions | Classify Scenario (LLM), in WF-2a and WF-2b |
| Portal question or new booking | Classify Booking Stage (LLM), in WF-2a and WF-2b |
| FTS stage definitions | Classify FTS Stage (LLM), in WF-2a and WF-2b |
| Intent definitions | Classify Intent (LLM), in WF-2 and WF-1b |
| Consult rules | Classify Consult (LLM), in WF-2 and WF-1b |
| Multi-question merging | Collate Replies (LLM) or Collate Forward (LLM), in WF-2b |

Intent and consult labels are stored with past replies in the library. Changing a definition affects which precedents match, so discuss these changes with the workflow owner first.

### Forward recipients

| Forward type | Where to change it |
|---|---|
| Booking from staff | `FORWARD_TO` in Build Outlook Forward HTML (Booking Staff), in WF-2a and WF-2b |
| Reschedule counselling | `FORWARD_TO` in Build Outlook Forward HTML (Reschedule Counselling), in WF-2a and WF-2b |
| Zoom | Build Outlook Forward HTML (Zoom), in WF-2a (reads `ZOOM_FORWARD_TO`) |
| Partnership | Variable `PARTNERSHIP_FORWARD_TO` |
| Volunteer or internship | Variable `VOLUNTEER_FORWARD_TO` |
| Consult of the day | `UCS_ConsultSchedule.xlsx` in Dropbox (no workflow change) |

To change a `FORWARD_TO` value: open the Code node, press Ctrl+F, search for `FORWARD_TO`, and replace only the value inside the quotes.

```js
// Before
const FORWARD_TO = 'hwb@nus.edu.sg'.trim();
// After
const FORWARD_TO = 'new.mailbox@nus.edu.sg'.trim();
```

### Operating hours and consult roster

Maintained in Dropbox; no workflow change needed. Keep file names, sheet names, and column headings unchanged.

| Change | Where |
|---|---|
| Public holiday | *Holiday Dates* sheet, `UCS_OperatingHours.xlsx` |
| Office closure | *Office Closure Dates* sheet, `UCS_OperatingHours.xlsx` |
| Reduced hours for a period | *Soft Closure Periods* sheet, `UCS_OperatingHours.xlsx` |
| Normal weekly hours and last walk-in | *General Operating Hours* sheet, `UCS_OperatingHours.xlsx` |
| Consult or covering consult | Month sheet, `UCS_ConsultSchedule.xlsx` |

Add each new month's roster sheet before that month starts, using the existing name pattern, because the workflow looks the sheet up by name.

### Blocked senders and clinician list

- **Blocked senders:** add the address in the Outlook Trigger (Inbox), Get Inbox Messages, and Filter - Exclude Own Mailbox nodes in WF-2, and in Outlook - Get Inbox Messages and Filter - Exclude Own Mailbox in WF-1b. Copy the format of an existing address exactly.
- **Clinician list:** update the numbered list in Extract Clinician (LLM) in WF-2, including the short names people use, and make the matching change in WF-1b.

Do not use these steps for routing logic, search filters, the index, merge nodes, or code that matches records. Ask the workflow owner to make those changes.

## Known limitations

- Emails with attachments are not read or drafted.
- Sender role comes from the email domain. For a forwarded email, it reflects the person who forwarded it.
- Classification uses the current email text and subject line, not the wider thread.
- Operating hours in drafts come from the Dropbox file, not from past replies.
- Internal closure event names are shown only as "an event". Holiday names are shown.
- Library records older than two years are removed when WF-RETENTION runs.

### Possible future enhancements

- Reading attachments so those emails can also be drafted
- Giving the classifiers the wider thread as context
- Moving FTS and Intellect routing higher up if those emails are classified outside appointment
- A feedback loop from officer edits to the library
- Regular quality reporting using the evaluation dataset and draft log
