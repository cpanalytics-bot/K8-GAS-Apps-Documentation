# 📚 School Selector App — Complete Workflow Documentation

> **For new team members:** This document explains everything you need to know about how the School Selector web application works — from a user's perspective and from a technical perspective. No prior knowledge assumed.

---

## Table of Contents

1. [What Is This App?](#1-what-is-this-app)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [Data Sources & Where Things Live](#3-data-sources--where-things-live)
4. [The Full User Journey (Step by Step)](#4-the-full-user-journey-step-by-step)
5. [Screen-by-Screen Breakdown](#5-screen-by-screen-breakdown)
6. [Backend Logic Deep Dive](#6-backend-logic-deep-dive)
7. [Database: What Gets Stored in Supabase](#7-database-what-gets-stored-in-supabase)
8. [Key Business Rules](#8-key-business-rules)
9. [How to Add a New Employee](#9-how-to-add-a-new-employee)
10. [Troubleshooting Common Issues](#10-troubleshooting-common-issues)
11. [Tech Stack Cheatsheet](#11-tech-stack-cheatsheet)

---

## 1. What Is This App?

The **School Selector App** is an internal web tool used by field employees (sales or relationship managers) to **record which schools in their territory they are actively managing or have relationships with**.

Each employee logs in, selects their state and districts, checks off schools they work with, optionally adds schools that are missing from the master list, fills in school-level data (fees, student strength, revenue), and submits. This data is stored in Supabase and used for reporting.

**In one sentence:** It's a structured data-collection form that ensures every employee's school portfolio is captured accurately in a central database.

---

## 2. System Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        USER'S BROWSER                            │
│                   (Index.html — the frontend)                    │
│   7 Screens rendered in sequence, state held in JS variables     │
└────────────────────────┬─────────────────────────────────────────┘
                         │  google.script.run (RPC calls)
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│              GOOGLE APPS SCRIPT BACKEND (Code.gs)                │
│   Serves the HTML page, handles auth, reads Sheets, writes       │
│   to Supabase via REST API                                        │
└──────────┬───────────────────────────────────┬───────────────────┘
           │                                   │
           ▼                                   ▼
┌──────────────────────┐           ┌───────────────────────────────┐
│   GOOGLE SHEETS      │           │         SUPABASE              │
│                      │           │   (PostgreSQL database)        │
│  • EMP Record Sheet  │           │                               │
│    (who can log in)  │           │  Table:                       │
│                      │           │  lvt_universe_data_           │
│  • 35 State Sheets   │           │  school_selector              │
│    (master school    │           │                               │
│     lists)           │           │  Stores each employee's       │
│                      │           │  school selections & data     │
└──────────────────────┘           └───────────────────────────────┘
```

### Key Point

The app has **two storage systems working together:**

- **Google Sheets** = read-only source of truth for *who can log in* and *what schools exist*
- **Supabase** = the live database where *employee submissions* are stored and updated

---

## 3. Data Sources & Where Things Live

### 3.1 Employee Record Sheet

| What | Details |
|------|---------|
| **Purpose** | Whitelist of authorized users |
| **Sheet ID** | `1Im_b4htxa-4tmI9Gl_C4Ney-nw85pey08HyQcGERut8` |
| **Tab Name** | `EMP Record` |
| **Key Column** | Column A (A2 onwards) — employee email addresses |
| **Used for** | Checking if a login email is authorized before sending OTP |

> ⚠️ If an employee can't log in, first check that their email is in Column A of this sheet.

### 3.2 State Master School Lists

There is **one Google Sheet per Indian state/UT** (35 total). Each sheet contains the master list of schools for that state.

| Column in Sheet | Data |
|----------------|------|
| Column F | (used internally) |
| Column G | **District name** |
| Column K | **School name** |

The app reads columns `F2:K` from the first tab of each state's sheet.

### 3.3 Supabase Table: `lvt_universe_data_school_selector`

This is the central database. Every employee submission creates or updates rows here.

| Column | Description |
|--------|-------------|
| `employee_email` | The logged-in employee's email |
| `state` | Selected state name |
| `district` | District of the school |
| `school_name` | School name (for new schools: `Name, Address, Pincode`) |
| `type` | `Old` (from master list) or `New` (added by employee) |
| `status` | `Active` (currently selected) or `Inactive` (deselected) |
| `strength_k8` | Student strength up to Class 8 |
| `monthly_fee` | Monthly fee per student |
| `user_school` | Whether it's the employee's own school (`Yes`/`No`) |
| `revenue` | Revenue figure |
| `employee_past_relationship` | Nature of past relationship with the school |

---

## 4. The Full User Journey (Step by Step)

Below is the complete end-to-end flow a user goes through every time they use the app.

```
[1] Enter Email
      │
      ▼
[2] Backend checks email against EMP Record Sheet
      │ ✗ Not found → Show error, stop
      │ ✓ Found ↓
      ▼
[3] Generate 6-digit OTP → Store in cache (5 min TTL) → Email to user
      │
      ▼
[4] User enters OTP → Backend validates against cache
      │ ✗ Wrong/expired → Show error
      │ ✓ Correct ↓
      ▼
[5] Load state dropdown (from STATE_SHEET_MAP keys)
      │
      ▼
[6] User selects state → Backend fetches:
      │   • Master school list from that state's Google Sheet
      │   • All existing Supabase records for this user (all states)
      ▼
[7] User selects districts (pre-ticked based on prior Active records)
      │
      ▼
[8] User selects schools (pre-ticked based on prior Active records)
      │
      ▼
[9] User indicates if any schools are missing from the list
      │   (If Yes → fills in a table with district, name, address, pincode)
      ▼
[10] Edit & Review screen — user can update:
      │   Strength K8 | Monthly Fee | User School | Revenue | Past Relationship
      ▼
[11] Final Review screen — summary of all selections + confirmation checkboxes
      │
      ▼
[12] Submit → Backend upserts data to Supabase:
      │   • Updates existing records (status, field values)
      │   • Marks deselected schools as Inactive
      │   • Inserts new records for newly selected schools
      │   • Inserts new school records for employee-added schools
      ▼
[13] Success modal shown → App resets on "Done"
```

---

## 5. Screen-by-Screen Breakdown

### Screen 1 — Login

**What the user sees:** A single email input field and a "Send OTP" button.

**What happens behind the scenes:**
1. `sendVerificationOtp(email)` is called on the backend.
2. Backend calls `verifyUser(email)` — opens the EMP Record Sheet and checks if the email exists in column A.
3. If found: generates a random 6-digit OTP, stores it in Google Apps Script's `CacheService` (expires in 5 minutes), and emails it to the user via `MailApp`.
4. If not found: returns an error message displayed as an alert.

---

### Screen 2 — OTP Verification

**What the user sees:** A field to enter the 6-digit OTP they received by email.

**What happens behind the scenes:**
1. `validateOtp(email, otp)` is called.
2. Backend retrieves the stored OTP from cache using the email as the key.
3. If it matches: cache entry is deleted (one-time use), and the user proceeds.
4. If expired or wrong: error message is shown.

---

### Screen 3 — State Selection

**What the user sees:** A dropdown listing all 35 Indian states and UTs.

**What happens behind the scenes:**
1. `getStates()` returns the keys of `STATE_SHEET_MAP` (the full list of states).
2. Once the user picks a state and clicks Continue, `getInitialData(state, email)` is called.
3. This fetches:
   - **Master school list** from the relevant state Google Sheet (columns G & K)
   - **All Supabase records** for this employee (across all states) — used to pre-populate selections on later screens

---

### Screen 4 — District Selection

**What the user sees:** A searchable checkbox list of all districts in the chosen state. Districts where the user had **Active** schools are pre-checked.

**Key feature — Lock/Unlock Toggle (🔓/✏️):**
- When **unlocked** (default): user can check/uncheck districts freely.
- When **locked** (click 🔓 to lock): all district checkboxes are disabled — prevents accidental changes.
- The lock icon shows 🔓 (unlocked) or ✏️ (locked/edit mode).

**Behind the scenes:**
- Districts are derived from the master school list (unique values from the district column).
- Pre-checked districts come from `userHistory` filtered to the current state where `status === 'Active'`.

---

### Screen 5 — School Selection

**What the user sees:** A searchable checkbox list of all schools in the selected districts. Schools the user had previously marked as Active are pre-checked.

**School badges:**
- **Active** (dark badge) — this school was in the user's last submission as active.
- **Available** (outline badge) — in the master list but not previously active for this user.
- **(Added by you)** — a school this employee previously added manually (a "private new school").

**Behind the scenes:**
- The composite school list = master list + any schools this employee previously added as `type: 'New'` for this state (called `privateNewSchools`).
- Pre-checked schools come from `userHistory` where `status === 'Active'`.

---

### Screen 6 — Add Missing Schools

**What the user sees:** A yes/no radio button asking if there are schools not in the master list.

- If **No**: proceed directly.
- If **Yes**: a table appears where the user can add rows with: District, School Name, Address, and Pincode (must be 6 digits).

**Validation:** Pincode is validated client-side with a regex (`/^[0-9]{6}$/`). Invalid pincodes block submission.

**Behind the scenes:** New schools are stored in `appState.newSchools`. Their final name in the database becomes `"SchoolName, Address, Pincode"` (concatenated).

---

### Screen 7 — Edit & Review

**What the user sees:** A table listing every selected school (both from master list and newly added), with editable fields:

| Field | Input Type | Options |
|-------|-----------|---------|
| Strength K(8) | Number input | Free numeric entry |
| Monthly Fee | Number input | Free numeric entry |
| User School | Dropdown | Yes / No |
| Revenue | Number input | Free numeric entry |
| Employee's Past Relationship | Dropdown | Sold books / No existing relationship / Previous company relation |

**Pre-population:** If the user has previously submitted data for a school, those values are pre-filled from `userHistory`.

---

### Screen 8 — Final Review & Submit

**What the user sees:** A summary of everything — email, state, districts, schools, new schools added — and two confirmation checkboxes. The Submit button is disabled until both are checked.

**On submit:** The backend `submitUserData()` function is called with the complete submission payload.

---

### Success Modal

Shown after a successful submission. Clicking "Done" reloads the page, resetting the entire app.

---

## 6. Backend Logic Deep Dive

### `submitUserData()` — The Core Submit Function

This is the most complex function. Here's exactly what it does:

```
INPUT: { email, state, confirmedDistricts, selectedSchools, newSchools, editedData }

STEP 1: Build a Set of all selected school keys (format: "schoolName||district")
        — includes both checkbox-selected schools AND newly added schools

STEP 2: Fetch all existing Supabase records for this user+state combination

STEP 3: Loop through existing records:
        ├── If school IS in selected set:
        │     → PATCH the record: set status = 'Active', apply any edits
        └── If school is NOT in selected set:
              → Only if the district is in confirmedDistricts AND current status is 'Active':
                  → PATCH: set status = 'Inactive'

STEP 4: Loop through selected schools from checkboxes:
        └── If NOT already in existing records → INSERT new record (type: 'Old', status: 'Active')

STEP 5: Loop through newly added schools:
        └── If NOT already in existing records → INSERT new record (type: 'New', status: 'Active')

OUTPUT: { success: true } or { success: false, message: "..." }
```

### Why "confirmedDistricts" Matters

A school is only marked **Inactive** if:
1. It was previously Active, **AND**
2. The user actually visited that district's screen (i.e., the district is in `confirmedDistricts`)

This prevents accidentally deactivating schools in districts the user never intended to modify.

### Supabase API Calls

The backend communicates with Supabase using REST:

| Operation | HTTP Method | When Used |
|-----------|-------------|-----------|
| Fetch user's records | `GET` with `?employee_email=eq.X` | On state selection |
| Fetch user's records for state | `GET` with `?employee_email=eq.X&state=eq.Y` | Before submission |
| Update existing record | `PATCH` with query filters | During submission |
| Insert new record | `POST` | During submission |

---

## 7. Database: What Gets Stored in Supabase

### Record Types

| `type` value | Meaning |
|-------------|---------|
| `Old` | School came from the master Google Sheet list |
| `New` | School was manually added by the employee |

> Note: There is legacy code that maps `'Existing'` to `'Old'` when reading records. This handles older records in the database.

### Status Values

| `status` value | Meaning |
|---------------|---------|
| `Active` | Employee currently works with / manages this school |
| `Inactive` | Employee deselected this school in a later submission |

### Example Row

```
employee_email: "john.doe@company.com"
state:          "MAHARASHTRA"
district:       "PUNE"
school_name:    "Sunrise International School"
type:           "Old"
status:         "Active"
strength_k8:    450
monthly_fee:    1200
user_school:    "No"
revenue:        540000
employee_past_relationship: "Sold books"
```

---

## 8. Key Business Rules

1. **Access Control:** Only emails listed in the EMP Record Sheet (Column A) can log in. There is no self-registration.

2. **OTP Expiry:** OTPs expire in exactly 5 minutes and are single-use (deleted from cache upon successful validation).

3. **Pre-population:** The app always loads the user's last known state from Supabase and pre-fills their selections — they don't start fresh each time.

4. **Soft Deletion:** Schools are never deleted from Supabase. They are only marked `Inactive`. This preserves historical data.

5. **New School Naming:** Employee-added schools get their name stored as `"Name, Address, Pincode"` — this is the composite key used to match them in future sessions.

6. **District Confirmation Logic:** If a user selects District X but that district is NOT in the districts they confirmed during their current session, any schools in District X will **not** be deactivated, even if unchecked. This is a safety mechanism.

7. **Blank Field Preservation:** On the Edit & Review screen, if a user submits a blank field, it does **not** overwrite existing data. Blank values are passed as `null` and the PATCH only updates non-null fields.

---

## 9. How to Add a New Employee

To give a new employee access to the app:

1. Open the **EMP Record Google Sheet** (ID: `1Im_b4htxa-4tmI9Gl_C4Ney-nw85pey08HyQcGERut8`)
2. Go to the tab named **"EMP Record"**
3. Add the employee's work email address to the next empty row in **Column A**
4. Save the sheet

That's it. The employee can now log in using that email and will receive an OTP.

> No code changes or deployments are needed to add new users.

---

## 10. Troubleshooting Common Issues

### "Email not found" error at login

- Check that the email is spelled correctly (case-insensitive check is used but verify anyway).
- Open the EMP Record Sheet and confirm the email is in Column A starting from row 2.

### OTP never arrives / "Could not send OTP"

- The Google Apps Script account must have Gmail sending permissions. Check that `MailApp` is authorized.
- OTPs expire in 5 minutes — ask the user to request a fresh one if it's been a while.

### State loads but shows no districts or schools

- The state's Google Sheet may be missing data or the Sheet ID in `STATE_SHEET_MAP` may be wrong.
- Check that the relevant sheet has data in columns F through K starting from row 2.

### Submission fails

- Check Apps Script execution logs in the Google Apps Script editor (View → Logs).
- The Supabase API key or URL in the script may have changed — verify `SUPABASE_URL` and `SUPABASE_API_KEY` constants.
- The Supabase table name `lvt_universe_data_school_selector` must match exactly.

### A school I added previously doesn't show up

- Employee-added schools for a state only appear as "Added by you" if their `type === 'New'` and `state` matches the currently selected state in Supabase.

---

## 11. Tech Stack Cheatsheet

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | HTML + CSS + Vanilla JavaScript | Single-page multi-screen UI |
| Backend | Google Apps Script (`.gs` file) | Server logic, auth, data fetching |
| Hosting | Google Apps Script Web App | Serves the HTML file via `doGet()` |
| Auth | Email OTP via `MailApp` + `CacheService` | Stateless, time-limited login |
| Master Data | Google Sheets (35 state sheets + 1 EMP sheet) | Read-only reference data |
| Live Database | Supabase (PostgreSQL via REST API) | Stores all employee submissions |
| Communication | `google.script.run` (GAS client-server RPC) | Frontend calls backend functions |

---

## Appendix: State → Sheet ID Mapping

The app covers all 35 Indian states and union territories. Each maps to a dedicated Google Sheet ID in the `STATE_SHEET_MAP` constant in `Code.gs`. To add a new state or change a sheet, update this map in the script and redeploy.

Currently covered: Andaman and Nicobar Islands, Andhra Pradesh, Arunachal Pradesh, Assam, Bihar, Chandigarh, Chhattisgarh, Dadra and Nagar Haveli and Daman and Diu, Delhi, Goa, Gujarat, Haryana, Himachal Pradesh, Jammu and Kashmir, Jharkhand, Karnataka, Kerala, Ladakh, Madhya Pradesh, Maharashtra, Manipur, Meghalaya, Mizoram, Nagaland, Odisha, Puducherry, Punjab, Rajasthan, Sikkim, Tamil Nadu, Telangana, Tripura, Uttar Pradesh, Uttarakhand, West Bengal.

---

*Last updated: March 2026 | Maintained by the Technology / Operations team*
