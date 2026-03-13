# School Selector Web Application

> Complete Technical & User Documentation

**Version:** 1.0 | **Last Updated:** March 2026 | **Platform:** Google Apps Script + Supabase

---

## Table of Contents

- [1. Introduction](#1-introduction)
- [2. Technology Stack](#2-technology-stack)
- [3. System Architecture](#3-system-architecture)
- [4. Complete User Workflow](#4-complete-user-workflow)
- [5. Screen-by-Screen Guide](#5-screen-by-screen-guide)
- [6. Backend Functions Reference](#6-backend-functions-reference)
- [7. Database Schema](#7-database-schema)
- [8. Configuration Reference](#8-configuration-reference)
- [9. Data Flow Diagrams](#9-data-flow-diagrams)
- [10. Troubleshooting Guide](#10-troubleshooting-guide)
- [11. FAQ](#11-faq)
- [12. Glossary](#12-glossary)

---

## 1. Introduction

### 1.1 What is the School Selector App?

The **School Selector** is an internal web application designed for company employees to manage their school portfolios.

**Key Features:**

| Feature | Description |
|---------|-------------|
| School Selection | Choose schools from a master database for your territory |
| Add New Schools | Register schools that don't exist in the system |
| Edit School Data | Update information like strength, fees, and revenue |
| Track History | Maintain records of all interactions and changes |
| Secure Access | OTP-based authentication without passwords |

### 1.2 Who Should Use This App?

| User Role | Purpose | Access Level |
|-----------|---------|--------------|
| Field Employees | Select and manage schools in assigned territories | Full Access |
| Regional Managers | Review team submissions via database | Read Access (Supabase) |
| IT Administrators | Maintain backend configuration | Admin Access |

### 1.3 Key Benefits

- No password management - OTP-based secure login
- Smart pre-selection - Previous choices are remembered
- Mobile-friendly - Works on any device
- Offline-ready structure - Single HTML file architecture
- Real-time sync - Data saved to cloud database
- Audit trail - Complete history maintained

---

## 2. Technology Stack

### 2.1 Architecture Overview

┌─────────────────────────────────────────────────────────────┐ │ FRONTEND LAYER │ │ HTML5 + CSS3 + Vanilla JavaScript │ │ (Single Page Application) │ └─────────────────────────────────────────────────────────────┘ │ │ google.script.run (RPC) ▼ ┌─────────────────────────────────────────────────────────────┐ │ BACKEND LAYER │ │ Google Apps Script (Code.gs) │ │ ┌──────────────┬──────────────┬──────────────────┐ │ │ │ Auth Service │ Data Service │ Submission Service│ │ │ └──────────────┴──────────────┴──────────────────┘ │ └─────────────────────────────────────────────────────────────┘ │ ┌─────────────────┼─────────────────┐ ▼ ▼ ▼ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ │ Google Sheets │ │ Supabase │ │ Google Mail │ │ (Master Data) │ │ (User Data) │ │ (OTP Send) │ └─────────────────┘ └─────────────────┘ └─────────────────┘



### 2.2 Technology Details

| Layer | Technology | Purpose |
|-------|------------|---------|
| Frontend | HTML5 | Structure & Content |
| Styling | CSS3 (Embedded) | Responsive Design |
| Client Logic | JavaScript (ES6+ Vanilla) | User Interactions |
| Backend | Google Apps Script (V8) | Server-side Processing |
| Primary Database | Supabase (PostgreSQL) | User Submissions & History |
| Master Data | Google Sheets | Read-only School Lists |
| Authentication | Google CacheService | OTP Storage (5 min TTL) |
| Email Service | Google MailApp | OTP Delivery |

### 2.3 External Services

| Service | URL | Purpose |
|---------|-----|---------|
| Supabase | `https://kfkcohosbpaeuzxuohrm.supabase.co` | Cloud Database |
| Google Sheets | Multiple Sheet IDs | Master School Data |

---

## 3. System Architecture

### 3.1 File Structure

School-Selector-App/ │ ├── Code.gs # Backend (Google Apps Script) │ ├── Configuration # API keys, Sheet IDs, URLs │ ├── Web App Entry # doGet() function │ ├── Authentication # OTP generation & validation │ ├── Data Retrieval # Fetch schools & history │ └── Data Submission # Save to Supabase │ └── Index.html # Frontend (Single HTML File) ├── <style> # Embedded CSS ├── HTML Screens # 9 different screens └── <script> # Embedded JavaScript



### 3.2 Screen Architecture

The application uses a **single-page architecture** with 9 screens:

| Screen ID | Purpose |
|-----------|---------|
| `screen-login` | Email entry for authentication |
| `screen-otp` | OTP verification |
| `screen-state` | State selection |
| `screen-districts` | District selection |
| `screen-schools` | School selection |
| `screen-add-schools` | Add new schools (optional) |
| `screen-edit-review` | Edit school details |
| `screen-review` | Final review before submission |
| `success-modal` | Confirmation modal |

### 3.3 Data Storage Strategy

| Data Type | Storage Location | Access Pattern |
|-----------|------------------|----------------|
| Employee Emails | Google Sheets | Read-only (Verification) |
| Master School Lists | Google Sheets (36 sheets) | Read-only (Per State) |
| User Selections | Supabase | Read/Write |
| User-Added Schools | Supabase | Read/Write |
| School Edit History | Supabase | Read/Write |
| OTP Codes | Google CacheService | Write once, Read once, Auto-expire |

---

## 4. Complete User Workflow

### 4.1 High-Level Flow

LOGIN → OTP VERIFY → STATE SELECT → DISTRICT SELECT → SCHOOL SELECT │ ┌──────────────────────────────────────────────────────┘ ▼ ADD NEW SCHOOLS (Optional) → EDIT & REVIEW → FINAL REVIEW → SUCCESS!



### 4.2 Detailed Workflow Steps

| Step | Screen | User Action | System Response |
|------|--------|-------------|-----------------|
| 1 | Login | Enter email | Validate email → Send OTP |
| 2 | OTP | Enter 6-digit code | Verify OTP → Grant access |
| 3 | State | Select state | Load master schools + user history |
| 4 | Districts | Check districts | Pre-select from history |
| 5 | Schools | Check schools | Pre-select active schools |
| 6 | Add Schools | (Optional) Add new | Collect new school data |
| 7 | Edit Review | Fill/edit details | Pre-fill from history |
| 8 | Final Review | Confirm selections | Display summary |
| 9 | Submit | Click Submit | Save all data to Supabase |

---

## 5. Screen-by-Screen Guide

### Screen 1: Login

**Screen ID:** `screen-login`

**Purpose:** Collect user's email and initiate authentication

**User Actions:**
1. Enter registered company email
2. Click "Send OTP"

**Backend Process:**
sendVerificationOtp(email) ├── verifyUser(email) // Check employee records │ └── Returns: {success: true/false} ├── Generate 6-digit OTP // Math.random() ├── Store in cache (5 min) // CacheService └── Send via email // MailApp.sendEmail()



**Validation:**
- Email must exist in Employee Records Google Sheet
- Email format must be valid

---

### Screen 2: OTP Verification

**Screen ID:** `screen-otp`

**Purpose:** Verify the user's identity via One-Time Password

**User Actions:**
1. Check email for 6-digit OTP
2. Enter OTP in input field
3. Click "Verify & Proceed"

**Backend Process:**
validateOtp(email, otp) ├── Retrieve stored OTP from cache ├── Compare with user input ├── If match: Remove from cache, return success ├── If expired: Return "OTP has expired" └── If wrong: Return "Incorrect OTP"



**OTP Scenarios:**

| Scenario | Message | Action |
|----------|---------|--------|
| OTP Correct | Success | Proceed to State Selection |
| OTP Wrong | "Incorrect OTP. Please try again." | Stay on screen |
| OTP Expired | "OTP has expired. Please request a new one." | Go back to login |

---

### Screen 3: State Selection

**Screen ID:** `screen-state`

**Purpose:** Select the state/territory for school management

**User Actions:**
1. Select state from dropdown (36 options)
2. Click "Continue"

**Available States (36 Total):**

| Region | States |
|--------|--------|
| North | Delhi, Haryana, Himachal Pradesh, Jammu & Kashmir, Ladakh, Punjab, Uttarakhand, Uttar Pradesh, Chandigarh |
| South | Andhra Pradesh, Karnataka, Kerala, Tamil Nadu, Telangana, Puducherry |
| East | Bihar, Jharkhand, Odisha, West Bengal, Sikkim |
| West | Gujarat, Maharashtra, Rajasthan, Goa, Dadra & Nagar Haveli and Daman & Diu |
| Northeast | Arunachal Pradesh, Assam, Manipur, Meghalaya, Mizoram, Nagaland, Tripura |
| Central | Chhattisgarh, Madhya Pradesh |
| Islands | Andaman & Nicobar Islands |

**Backend Process:**
getInitialData(state, email) ├── getMasterSchoolList(state) // From Google Sheets ├── getFormResponseDataFromSupabase(state, email) │ ├── privateNewSchools[] // User-added schools │ └── userHistory[] // All previous records └── Return combined data to frontend



---

### Screen 4: District Selection

**Screen ID:** `screen-districts`

**Purpose:** Select districts within the chosen state

**Features:**

| Feature | Icon | Description |
|---------|------|-------------|
| Lock Toggle | 🔓 / ✏️ | Lock/unlock district selection |
| Search | 🔍 | Filter districts by name |
| Pre-selection | ✓ | Districts with Active schools are pre-checked |
| Badge Display | 🏷️ | Shows selected districts at bottom |

**Pre-selection Logic:**
```javascript
// Districts are pre-selected if user has Active schools there
activeDistricts = userHistory
    .filter(h => h.state === selectedState && h.status === 'Active')
    .map(h => h.district);
Screen 5: School Selection
Screen ID: screen-schools
Purpose: Select specific schools within chosen districts
School Display Information:
Element	Description
School Name	Official name from master list
District Badge	Shows which district (outline style)
Status Badge	"Active" (filled) or "Available" (outline)
Private Tag	"(Added by you)" for user-added schools

Export as CSV
Composite List Building:
javascript


// Frontend combines master list + user's private schools
compositeSchoolList = [
    ...masterList.map(s => ({ ...s, isPrivate: false })),
    ...privateNewSchools.map(s => ({ ...s, isPrivate: true }))
];
// Duplicates removed by school key: "name||district"
Screen 6: Add Missing Schools
Screen ID: screen-add-schools
Purpose: Allow users to add schools not in the master database
Input Fields:
Field	Type	Validation	Example
District	Dropdown	Required, from selected districts	"Central Delhi"
School Name	Text	Required	"ABC International School"
Address	Text	Required	"123, Main Street, Sector 5"
Pincode	Text	Required, exactly 6 digits	"110001"

Export as CSV
Data Storage Format:
javascript


// New schools are stored with combined name
combinedName = `${schoolName}, ${address}, ${pincode}`;
// Example: "ABC International School, 123 Main Street, 110001"
Screen 7: Edit & Review
Screen ID: screen-edit-review
Purpose: Edit detailed information for all selected schools
Editable Fields:
Field	Input Type	Description	Example Values
Strength K(8)	Number	Students up to Class 8	500, 1000, 2500
Monthly Fee	Number	Fee in ₹ (INR)	1500, 5000, 15000
User School	Dropdown	Is this a user school?	Yes, No
Revenue	Number	Revenue amount in ₹	50000, 100000
Employee's Past Relationship	Dropdown	Previous connection	See options below

Export as CSV
Relationship Options:
Sold books
No existing relationship
Previous company relation
Pre-fill Logic:
javascript


// If school exists in userHistory, pre-fill values
historyEntry = userHistory.find(h => 
    h.name === school.name && h.district === school.district
);
if (historyEntry) {
    strengthField.value = historyEntry.strength;
    feeField.value = historyEntry.fee;
    // ... etc
}
Screen 8: Final Review
Screen ID: screen-review
Purpose: Final confirmation before submission
Displayed Information:
Email address
Selected state (as badge)
List of selected districts with count
List of selected schools with count
List of new schools to add (if any)
Submission Requirements:
Requirement	Description
Confirmation 1	Must confirm information accuracy
Confirmation 2	Must acknowledge system update
Submit Button	Enabled only when both boxes checked

Export as CSV
Screen 9: Success Modal
Purpose: Confirm successful submission
User Action: Click "Done" to reset and start over
6. Backend Functions Reference
6.1 Authentication Functions
sendVerificationOtp(email)
Validates email and sends OTP to user.
javascript


/**
 * @param {string} email - User's email address
 * @returns {Object} {success: boolean, message?: string}
 */
function sendVerificationOtp(email) {
    // 1. Verify user exists in Employee Records
    // 2. Generate 6-digit random OTP
    // 3. Store in cache with 5-minute expiry
    // 4. Send email via MailApp
}
validateOtp(email, otp)
Verifies OTP entered by user.
javascript


/**
 * @param {string} email - User's email address
 * @param {string} otp - 6-digit OTP from user
 * @returns {Object} {success: boolean, message?: string}
 */
function validateOtp(email, otp) {
    // 1. Retrieve stored OTP from cache
    // 2. Compare with user input
    // 3. If match: clear cache, return success
    // 4. If mismatch: return appropriate error
}
verifyUser(email)
Checks if email exists in Employee Records.
javascript


/**
 * @param {string} email - Email to verify
 * @returns {Object} {success: boolean, message?: string}
 */
function verifyUser(email) {
    // Opens Employee Records sheet
    // Checks column A for email match
}
6.2 Data Retrieval Functions
getStates()
Returns list of all available states.
javascript


/**
 * @returns {Array<string>} Array of state names (36 states)
 */
function getStates() {
    return Object.keys(STATE_SHEET_MAP);
}
getInitialData(state, userEmail)
Fetches all data needed for a state.
javascript


/**
 * @param {string} state - Selected state name
 * @param {string} userEmail - User's email
 * @returns {Object} {masterList, privateNewSchools, userHistory}
 */
function getInitialData(state, userEmail) {
    // 1. Get master school list from Google Sheets
    // 2. Get user history from Supabase
    // 3. Combine and return
}
getMasterSchoolList(state)
Gets schools from state-specific Google Sheet.
javascript


/**
 * @param {string} state - State name
 * @returns {Array<Object>} [{district, name}, ...]
 */
function getMasterSchoolList(state) {
    // Opens sheet by STATE_SHEET_MAP[state]
    // Reads columns F (district) and K (school name)
    // Returns array of school objects
}
getFormResponseDataFromSupabase(state, userEmail)
Retrieves user's historical data from Supabase.
javascript


/**
 * @param {string} state - State name
 * @param {string} userEmail - User's email
 * @returns {Object} {privateNewSchools, userHistory}
 */
function getFormResponseDataFromSupabase(state, userEmail) {
    // Queries Supabase for all user records
    // Separates private/new schools from history
    // Returns both arrays
}
6.3 Data Submission Functions
submitUserData(submissionData)
Main submission handler.
javascript


/**
 * @param {Object} submissionData
 *   - email: string
 *   - state: string
 *   - confirmedDistricts: Array<string>
 *   - selectedSchools: Array<{name, district}>
 *   - newSchools: Array<{district, name, address, pincode}>
 *   - editedData: Array<{key, strength, fee, userSchool, revenue, relationship}>
 * @returns {Object} {success: boolean, message?: string}
 */
function submitUserData(submissionData) {
    // 1. Fetch existing user records
    // 2. Update existing records (Active/Inactive)
    // 3. Insert new records for newly selected schools
    // 4. Insert records for user-added schools
}
updateSchoolRecord(email, state, district, schoolName, updates)
Updates an existing record in Supabase.
javascript


/**
 * @param {string} email
 * @param {string} state
 * @param {string} district
 * @param {string} schoolName
 * @param {Object} updates - Fields to update
 */
function updateSchoolRecord(email, state, district, schoolName, updates) {
    // PATCH request to Supabase
    // Updates matching record
}
insertSchoolRecord(recordData)
Inserts a new record into Supabase.
javascript


/**
 * @param {Object} recordData
 *   - employee_email
 *   - type: "Old" | "New"
 *   - state, district, school_name
 *   - status: "Active"
 *   - strength_k8, monthly_fee, user_school, revenue, employee_past_relationship
 */
function insertSchoolRecord(recordData) {
    // POST request to Supabase
    // Creates new record
}
7. Database Schema
7.1 Supabase Table
Table Name: lvt_universe_data_school_selector
Column	Type	Nullable	Description
employee_email	TEXT	NO	User's email address
type	TEXT	NO	"Old" (master list) or "New" (user added)
state	TEXT	NO	State name
district	TEXT	NO	District name
school_name	TEXT	NO	School name (or combined for new schools)
status	TEXT	NO	"Active" or "Inactive"
strength_k8	INTEGER	YES	Student count up to Class 8
monthly_fee	NUMERIC	YES	Monthly fee amount
user_school	TEXT	YES	"Yes" or "No"
revenue	NUMERIC	YES	Revenue amount
employee_past_relationship	TEXT	YES	Relationship type

Export as CSV
7.2 Sample Data
employee_email	type	state	district	school_name	status
user@company.com	Old	DELHI	Central Delhi	Delhi Public School	Active
user@company.com	New	DELHI	North Delhi	ABC Academy, 123 St, 110001	Active
user@company.com	Old	DELHI	East Delhi	St. Mary's School	Inactive

Export as CSV
7.3 Google Sheets Structure
Employee Records Sheet
Sheet ID: 1Im_b4htxa-4tmI9Gl_C4Ney-nw85pey08HyQcGERut8
Tab Name: EMP Record
Data: Column A contains authorized employee emails
Master School Sheets (Per State)
Data Range: F2:K
Column F (Index 1): District
Column K (Index 5): School Name
8. Configuration Reference
8.1 Supabase Configuration
javascript


const SUPABASE_URL = 'https://kfkcohosbpaeuzxuohrm.supabase.co';
const SUPABASE_API_KEY = '[YOUR_API_KEY]';
const SUPABASE_TABLE = 'lvt_universe_data_school_selector';
8.2 Google Sheets Configuration
javascript


// Employee Records
const EMP_RECORD_SHEET_ID = '1Im_b4htxa-4tmI9Gl_C4Ney-nw85pey08HyQcGERut8';
const EMP_RECORD_TAB_NAME = 'EMP Record';
8.3 State to Sheet ID Mapping
javascript


const STATE_SHEET_MAP = {
    'ANDAMAN AND NICOBAR ISLANDS': '1uGdAdB-FDJ03mcE4R5lNNR31lyh1Xwsoj0oDwKDVbRU',
    'ANDHRA PRADESH': '1CJ47zQ2Xt4mEVPFKDl4R5ANp8O4IQSzb8750MmiTEAI',
    'ARUNACHAL PRADESH': '1HOQ8lbXRTEukaMO8qdZip4a4qXZrY2Xf_lPhqIE-4VU',
    'ASSAM': '1iuaexfcVRu_oxZ4up755mr2tmIlFIHyQj1EDISECIf0',
    'BIHAR': '18GbgajKDmihuJuAOUd6PmbkT_QcfqDyihR3D04yhV6s',
    'CHANDIGARH': '183ANAtHPOuRShiYf-9c9pNKYjVs2JY5f0uA03cR4Mlw',
    'CHHATTISGARH': '1P6b_-eHEiRI_nYlb28Fmm4LRbWQaOXqxm__C8bI7bu4',
    'DADRA AND NAGAR HAVELI AND DAMAN AND DIU': '1HswuLDUVGFIS5qk4XjCIcSNdCVX_LPBypTF4MrPXBuo',
    'DELHI': '1e6RijZX5-pRTGpxltOXsQ37MtaE_QxQaN1vH_5OwbPU',
    'GOA': '16DpQnbGrWu-ljcDtUAQWWn8YD5WWeFmfK2obUvtnWd0',
    'GUJARAT': '196InEY7zMZE6ueQG4Cs4VqLhP8k0V984EVUcTo8uoqM',
    'HARYANA': '1hIUpL6I0229Qvh59l-R3xntPiAGkM7Pd-Um3nJmTlAU',
    'HIMACHAL PRADESH': '1Q4yTiz5nAut-lJO3pXkxzYNDc4ObUci_ghUDehrmkwY',
    'JAMMU AND KASHMIR': '1fFy7yNUMRDndjSBrwO_-0Gu-Tg_sLYF2iyiOONmpEto',
    'JHARKHAND': '1KzeJ35xRv3FSgXaiRBVRkUuVJPtNzO6o4ym9BLT02Ro',
    'KARNATAKA': '1XMgnaUwm975LFnHvPpAVbBWGnC0CKNhOo_NK1m4iu0w',
    'KERALA': '1j_S5LQ0utq5btaymHbGNGAPOLEaW02R9Mc6MsgW0Fak',
    'LADAKH': '1c63544MdqyNQY3-O49OvEJCtq3PpBr5GeSjVhZohu-g',
    'MADHYA PRADESH': '1TwFNlzGZcCUMpcnLTDcp6QXMizhhS1-16wS4w3trcTM',
    'MAHARASHTRA': '1VRH4oER2pjGG8ybK_8dqrwMt5pejxdc_YKAQzCFGKiI',
    'MANIPUR': '1rkV4Jjn8m_e665AhIES0rDwRfH1jSxC3xS6RLUhzUOQ',
    'MEGHALAYA': '1mBklqP9XTdPM9s01IDUN5ZeZgdJwX7Lxq0W26v1FkbY',
    'MIZORAM': '1ykL0kx6S2SqI4C6blAEZ1401C9Cy1MeaHjThl1lukBs',
    'NAGALAND': '1pYjIYvHPUdptYw0shsFNJ0i-5pnD2FLPgmgTAmxVQFI',
    'ODISHA': '1xMiLDCPXUeHkK4JavXU7sAF97198N3w5IZqmI7BgzG8',
    'PUDUCHERRY': '1S-0rg7gIztzW_unvfSqGeUObiBFNNs-Y66M38Rs1oE4',
    'PUNJAB': '1Xa4Lb795Co5rWpYsFeAbrfjxNncLx2KbhLMfxcGDuV0',
    'RAJASTHAN': '1B6RHhxyXf51ehySuUbWZdmqYHPVvCd45yTQqrHF0li4',
    'SIKKIM': '1_dCWc1irOaAz5jXdlhqv-FkDSL8MSBdIbzv1sdHdJMw',
    'TAMIL NADU': '15xUDjfva2zItY6NxhhIF8BaLK5rDEWQa2rHg9v5LMKI',
    'TELANGANA': '14Mekm52YRly26zJxz26Pz-Zpoxye6afZu06zMYxwKpg',
    'TRIPURA': '1Slekf-bJNwbUXfLYLGbcBkc4ByTNIH2-Bx3PM7cUWyo',
    'UTTAR PRADESH': '1Im_b4htxa-4tmI9Gl_C4Ney-nw85pey08HyQcGERut8',
    'UTTARAKHAND': '1bnIPk_L2KprNgsWnsnprOTmeZIYN-By3E37B7PPpFUE',
    'WEST BENGAL': '16I6ASg_KxCzJ8ffkkwO5A-Dcjvw3-rUr7v07LmYQNJE'
};
9. Data Flow Diagrams
9.1 Authentication Flow


User                    Frontend                Backend                 External
 │                         │                       │                       │
 │  Enter Email            │                       │                       │
 │────────────────────────>│                       │                       │
 │                         │  sendVerificationOtp()│                       │
 │                         │──────────────────────>│                       │
 │                         │                       │  verifyUser()         │
 │                         │                       │──────────────────────>│ Google Sheets
 │                         │                       │<──────────────────────│
 │                         │                       │  Generate OTP         │
 │                         │                       │  Store in Cache       │
 │                         │                       │──────────────────────>│ MailApp
 │                         │<──────────────────────│                       │
 │  "OTP Sent"             │                       │                       │
 │<────────────────────────│                       │                       │
 │                         │                       │                       │
 │  Enter OTP              │                       │                       │
 │────────────────────────>│                       │                       │
 │                         │  validateOtp()        │                       │
 │                         │──────────────────────>│                       │
 │                         │                       │  Check Cache          │
 │                         │                       │  Compare OTP          │
 │                         │<──────────────────────│                       │
 │  "Success"              │                       │                       │
 │<────────────────────────│                       │                       │
9.2 Data Loading Flow


User                    Frontend                Backend                 External
 │                         │                       │                       │
 │  Select State           │                       │                       │
 │────────────────────────>│                       │                       │
 │                         │  getInitialData()     │                       │
 │                         │──────────────────────>│                       │
 │                         │                       │                       │
 │                         │                       │  getMasterSchoolList()│
 │                         │                       │──────────────────────>│ Google Sheets
 │                         │                       │<──────────────────────│
 │                         │                       │                       │
 │                         │                       │  getFormResponse...() │
 │                         │                       │──────────────────────>│ Supabase
 │                         │                       │<──────────────────────│
 │                         │                       │                       │
 │                         │<──────────────────────│                       │
 │                         │  Build Composite List │                       │
 │                         │  Pre-select Districts │                       │
 │                         │  Pre-select Schools   │                       │
 │  Display Data           │                       │                       │
 │<────────────────────────│                       │                       │
9.3 Submission Flow


User                    Frontend                Backend                 Supabase
 │                         │                       │                       │
 │  Click Submit           │                       │                       │
 │────────────────────────>│                       │                       │
 │                         │  submitUserData()     │                       │
 │                         │──────────────────────>│                       │
 │                         │                       │                       │
 │                         │                       │  getExistingRecords() │
 │                         │                       │──────────────────────>│
 │                         │                       │<──────────────────────│
 │                         │                       │                       │
 │                         │                       │  For each existing:   │
 │                         │                       │  - If selected:       │
 │                         │                       │    Update -> Active   │
 │                         │                       │  - If deselected:     │
 │                         │                       │    Update -> Inactive │
 │                         │                       │──────────────────────>│
 │                         │                       │                       │
 │                         │                       │  For new selections:  │
 │                         │                       │  insertSchoolRecord() │
 │                         │                       │──────────────────────>│
 │                         │                       │                       │
 │                         │                       │  For new schools:     │
 │                         │                       │  insertSchoolRecord() │
 │                         │                       │──────────────────────>│
 │                         │                       │                       │
 │                         │<──────────────────────│                       │
 │  Show Success Modal     │                       │                       │
 │<────────────────────────│                       │                       │
10. Troubleshooting Guide
10.1 Common Issues & Solutions
Issue	Possible Cause	Solution
"Email not found"	Email not in Employee Records	Add email to column A of EMP Record sheet
OTP not received	Email in spam / delay	Check spam folder; wait 2 mins; resend
"OTP has expired"	More than 5 minutes passed	Go back and request new OTP
Schools not loading	Google Sheet access issue	Verify Sheet ID in STATE_SHEET_MAP
Submission failed	Supabase connection error	Check API key; verify table exists
Blank dropdowns	Sheet data format issue	Verify data is in columns F and K
"Network error"	Internet connectivity	Check connection; retry

Export as CSV
10.2 Error Logging
To view server-side logs:
Open Google Apps Script editor
Go to View → Executions
Click on failed execution
View error details and stack trace
Log locations:
javascript


// Backend errors logged using:
Logger.log('Error message: ' + e.toString());
// Frontend errors shown via: alert('An error occurred: ' + error.message);



### 10.3 Debug Checklist

- [ ] Is the user's email in the Employee Records sheet?
- [ ] Is the correct Google Sheet ID mapped for the state?
- [ ] Is the Supabase API key valid and not expired?
- [ ] Does the Supabase table exist with correct schema?
- [ ] Is the internet connection stable?
- [ ] Are there any browser console errors?
- [ ] Is the Google Apps Script deployed correctly?

---

## 11. FAQ

### General Questions

**Q: Can I use this app on mobile?**

A: Yes! The app is designed mobile-first with responsive CSS.

**Q: How long is the OTP valid?**

A: OTPs expire after 5 minutes.

**Q: Can I change my state selection after proceeding?**

A: Yes, use the "Back" button to return to the state selection screen.

**Q: What happens if I deselect a school I previously selected?**

A: Its status changes to "Inactive" in the database. The record is preserved.

### Technical Questions

**Q: Where is my data stored?**

A: User selections and history are stored in Supabase (cloud PostgreSQL). Master school lists are in Google Sheets.

**Q: Can I add a school that already exists?**

A: No, duplicates are prevented by the composite key (school name + district).

**Q: What format is the pincode?**

A: Exactly 6 numeric digits (e.g., 110001).

**Q: How do I add a new employee?**

A: Add their email to the Employee Records Google Sheet, column A.

---

## 12. Glossary

| Term | Definition |
|------|------------|
| **OTP** | One-Time Password - 6-digit code sent via email for verification |
| **Master List** | Official list of schools from Google Sheets (read-only) |
| **Private/New School** | School added by user that doesn't exist in master list |
| **Composite List** | Combined list of master schools + user-added private schools |
| **Active Status** | School is currently selected by the user |
| **Inactive Status** | School was previously selected but is now deselected |
| **User History** | All previous submissions by the user stored in Supabase |
| **School Key** | Unique identifier: `"school_name\|\|district"` |
| **Cache** | Temporary storage for OTP (auto-expires in 5 minutes) |
| **Supabase** | Cloud database service using PostgreSQL |
| **Google Apps Script** | Server-side JavaScript platform by Google |
| **SPA** | Single Page Application - all screens in one HTML file |
| **RPC** | Remote Procedure Call - `google.script.run` calls |



## Document Information

| Field | Value |
|-------|-------|
| Document Title | School Selector Web Application Documentation |
| Version | 1.0 |
| Created | March 2026 |
| Purpose | New Team Member Onboarding & Reference |
| Maintainer | Development Team |



**End of Documentation**

*For questions or updates, contact the Development Team.*
