# Workflow Documentation

Node‑by‑node breakdown of all 5 workflows. Sheet/Drive IDs are the ones baked into the source JSON (see [Before you publish](README.md#before-you-publish-this-repo)).

- **Student Database** spreadsheet: `1FNf6IkEBrZ6oAJ4ZJUf11-9jHeabXoTz2BF3Z9Sr3uc`, tab `database`
- **Student Records** spreadsheet: `1wdh6ZDeGH_RZCspQRazyUTF-IULlTBXmFviJJBr9_lM`, tabs `asheet` and `assignment 1`

---

## 1. Student Registration & Enrollment

`Student Registration & Enrollment.json` · internal name `Student Registration & Enrollment`

**Purpose:** Onboard a new student from a public Google Form — validating there's no existing registration, issuing a Student ID, recording them in the master roster, welcoming them by email, and provisioning a personal Drive folder for future submissions.

**Trigger:** `On form submission` — form titled *"Student Registration Form"*.

**Form fields:** First Name, Last Name, Email Address, Mobile Number, Date of Birth, Gender (radio: Male / Female / Non‑binary / Prefer not to say), Course (dropdown: BCA / B.Tech / MBA / MCA / BBA / M.Tech), Parent/Guardian Name, Parent Contact Number. All required.

**Flow:**

1. **On form submission** — captures the raw form fields.
2. **Edit Fields** — normalizes the payload: `fullName` (first + last, trimmed), `email` (trimmed + lower‑cased), `mobile`, `dateOfBirth`, `gender`, `course`, `parentName`, `parentContact`, `registrationDate` (date part of the form's `submittedAt`), and `status = "Registered"`.
3. **Sheet** — looks up *Student Database* by `Email` to check for an existing registration.
4. **If** (`Email !== undefined`):
   - **True** (already registered) → **Send a message**: *"Registration Already Exists"* email.
   - **False** (new student) → **Code in JavaScript**.
5. **Code in JavaScript** — generates the Student ID: `Mlit-<COURSE>-<mobile digits>`, where `<mobile digits>` is the first two digits + last two digits of the mobile number (e.g. `Mlit-BCA-9834`).
6. **Edit Fields1** — carries the generated ID forward as `studentld` *(sic — lowercase "L", a naming typo; used consistently downstream so it's functionally fine)*.
7. **Append row in sheet** — appends the new row to *Student Database*: `Student ID, Full Name, Email, Mobile, Course, Registration Date`.
8. Two parallel branches off the append:
   - **Create folder** (Google Drive) — creates `<Student ID> - <First Name>` inside the parent Drive folder (`1_s3FYk1RRxgytP3gCu0o9XEVHR_IqdHh` in the source workflow) → **Update row in sheet** writes the new folder's ID back into the `driveID` column, matched on `Student ID`.
   - **Send a message1** — *"Welcome to MoonLit University – Registration Successful"* email, including the new Student ID.

**Reads:** Student Database (duplicate check).
**Writes:** Student Database (new row, then `driveID` back‑fill); Google Drive (new folder).
**Credentials:** Google Sheets OAuth2, Gmail OAuth2, Google Drive OAuth2.

---

## 2. Assignment Pipeline

`Assignment PipeLine.json` · internal name `Assignment PipeLine`

**Purpose:** Accept a Biology assignment submission, verify and de‑duplicate it, file it to Drive, grade it with an LLM against a fixed rubric, and email the result — the most involved workflow in the suite (22 nodes).

**Trigger:** `On form submission` — form titled *"Assignment Submission - 1"*.

**Form fields:** Student ID, Name, Registered Email, plus 5 free‑text questions (Photosynthesis, DNA structure/function, human circulatory system, natural selection/Darwin, mitosis). All required.

**Flow:**

1. **On form submission**.
2. **Get row(s) in sheet** — looks up *Student Database* by `Registered Email` **OR** `Student ID` (`combineFilters: OR`).
3. **If** (`Email !== undefined`):
   - **False** (not a registered student) → **Send a message**: *"Assignment Submission Failed – Student Verification Required"*.
   - **True** (verified) → **Get row(s) in sheet1**.
4. **Get row(s) in sheet1** — looks up *Student Records → assignment 1* by `Email`, to check for a prior submission.
5. **If1** (`Email !== undefined` evaluated as **false**, i.e. *no* existing submission):
   - **False branch** (a submission already exists) → **Send a message1**: *"Assignment Submission Rejected – Assignment Already Submitted"*.
   - **True branch** (first submission) → fans out to **Edit Fields** *and* **Basic LLM Chain** in parallel.
6. **Edit Fields** — renders the raw Q&A into a formatted plain‑text submission report (`assigmentfile`).
7. **Convert to File** → **assignment** (Google Drive) — uploads the text as `assignment 1 - <Student ID>` into the student's personal folder (looked up via `driveID` on the Student Database row).
8. **Append or update row in sheet** — appends to *assignment 1*: `Student ID, Name, Email, Assignment Link (Drive webViewLink)`, with `Percentage` and `Result Report` left blank, matched on `Student ID`.
9. In parallel, **Basic LLM Chain** (Google Gemini, via `Google Gemini Chat Model`, with `Structured Output Parser` enforcing a JSON schema and Groq wired in as the parser's auto‑fix model) grades the 5 answers — see [ARCHITECTURE.md](ARCHITECTURE.md#3-key-flow-assignment-pipeline-with-ai-grading) for the full rubric and schema.
10. **Basic LLM Chain** output fans out to **Send a message2** *and* **Edit Fields1**:
    - **Send a message2** — *"Assignment 1 Evaluation Completed – MoonLit University"* email with the marks/feedback per question. ⚠️ *Currently reads the grade via `.output.properties.<field>.type` — see [Known Issues](README.md#known-issues--recommendations).*
    - **Edit Fields1** — renders the same result into a formatted evaluation‑report text (`assigmentresult`), same expression issue.
11. **Convert to File1** → **assignment - report** (Google Drive) — uploads the evaluation report as `assignment 1 - <Student ID> - report`, into the same student folder.
12. **Append or update row in sheet1** — updates the *assignment 1* row: `Result Report` (Drive link) and `Percentage`, matched on `Student ID`.
13. **Append or update row in sheet2** — mirrors `Percentage` onto the matching row in *Student Database* (column `assignment 1`).

**Unused node:** `Message a model` (a direct `googleGemini` node targeting `gemini-3-flash-preview`) exists in the canvas with no connections in or out — not part of the executing graph.

**Reads:** Student Database (verification); Student Records → assignment 1 (duplicate check).
**Writes:** Google Drive (2 files per submission); Student Records → assignment 1 (2 writes); Student Database (percentage mirror).
**Credentials:** Google Sheets OAuth2, Gmail OAuth2, Google Drive OAuth2, Google Gemini (PaLM) API, Groq API.

---

## 3. Attendance Sheet Updater

`Attendance Sheet Updater.json` · internal name `Attendance Sheet Updater`

**Purpose:** Every morning, make sure each student has an (initially blank) attendance row for the day, so there's something for attendance to be marked against later.

**Triggers:** `Schedule Trigger` (daily at 08:00) or `Manual Trigger`.

**Flow:**

1. **Get row(s) in sheet** — reads every row from *Student Database*.
2. **Code in JavaScript** — for each student, adds `Date` (today, `YYYY-MM-DD`) and `AttendanceID` (`"<date>_<Student ID>"`).
3. **Loop Over Items** (Split In Batches) — processes one student at a time.
4. **Get row(s) in sheet1** — looks up *Student Records → asheet* filtered by `Date` **and** `Student ID` (`alwaysOutputData: true`, so a miss doesn't stop the run).
5. **If** (`row_number !== undefined`):
   - **True** (a row for today already exists) → loop back, skip.
   - **False** (no row yet) → **Append or update row in sheet**.
6. **Append or update row in sheet** — appends to *asheet*: `Attendance ID, Date, Student ID, Full Name, Course, email`, matched on `Attendance ID`. The `Attendance` column is intentionally left unset.
7. Loops back to process the next student.

**Reads:** Student Database (full roster); Student Records → asheet (existence check).
**Writes:** Student Records → asheet (one blank row per student per day).
**Credentials:** Google Sheets OAuth2.

---

## 4. AttendaceWarn *(Attendance Warning)*

`AttendaceWarn.json` · internal name `AttendaceWarn` *(sic — cosmetic typo, missing an "n")*

**Purpose:** At the end of the day, notify every student who was marked absent.

**Triggers:** `Schedule Trigger` (daily at 16:00) or `Manual Trigger`.

**Flow:**

1. **Get row(s) in sheet** — reads *Student Records → asheet* filtered to `Date = today`.
2. **Loop Over Items** (Split In Batches).
3. **If** — `Attendance` equals `"a"` **or** `"A"` (loose/case‑insensitive OR).
   - **True** → **Send a message**.
   - **False** → loop back, skip.
4. **Send a message** — *"Attendance Alert: Absent Marked for Today"* email with Student ID, Course, Date, and instructions to contact faculty if it's an error.
5. Loops back to the next row.

**Reads:** Student Records → asheet (today's rows).
**Writes:** none (email only).
**Credentials:** Google Sheets OAuth2, Gmail OAuth2.

**Note:** nothing in this suite of 5 workflows *sets* the `Attendance` value to `P`/`A` — it's seeded blank by workflow 3, and this workflow only reads it. Marking attendance itself happens outside these files.

---

## 5. CertificationSystem

`CertificationSystem.json` · internal name `CertificationSystem`

**Purpose:** Decide pass/fail per graded assignment and send either a certificate or a "not completed" notice.

**Trigger:** `Manual Trigger` only (no schedule).

**Flow:**

1. **Get row(s) in sheet** — reads every row from *Student Records → assignment 1*.
2. **If** — `Percentage > 0.4`.
   - **True** (pass) → **Get row(s) in sheet1**.
   - **False** (fail) → **Send a message1**.
3. **Get row(s) in sheet1** — looks up *Student Database* by `Email`, to pull the student's `Full Name`.
4. **Send a message** — HTML "Certificate of Completion" email (styled certificate template), addressed to the looked‑up `Full Name`, for `Course`.
5. **Send a message1** — HTML "Course Result – Not Successfully Completed" notice, addressed to `Name`/`Email` straight from the *assignment 1* row.

**Reads:** Student Records → assignment 1; Student Database (name lookup on pass).
**Writes:** none (email only).
**Credentials:** Google Sheets OAuth2, Gmail OAuth2.

**Note:** see [Known Issues #1–3](README.md#known-issues--recommendations) — the `Percentage > 0.4` threshold assumes a 0–1 fraction, but the Assignment Pipeline's rubric produces a 0–100 value, and (separately) that value isn't currently being written correctly upstream.

---

## Credentials reference

| Credential | Type | Used in |
|---|---|---|
| Google Sheets account | `googleSheetsOAuth2Api` | All 5 workflows |
| Gmail account | `gmailOAuth2` | All except Attendance Sheet Updater |
| Google Drive account | `googleDriveOAuth2Api` | Registration, Assignment Pipeline |
| Google Gemini (PaLM) API | `googlePalmApi` | Assignment Pipeline (grading) |
| Groq account | `groqApi` | Assignment Pipeline (structured‑output auto‑fix) |

Every credential is referenced by the original author's internal ID, so each will need to be re‑mapped to your own after import (see [Getting started](README.md#getting-started) in the README).
