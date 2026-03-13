# 📚 PW Academy Order Form — Complete Workflow Documentation

> **Version:** 4.0 | **Platform:** Google Apps Script Web App | **Audience:** New Team Members & Onboarding

---

## Table of Contents

1. [What Is This App?](#1-what-is-this-app)
2. [Tech Stack & Architecture](#2-tech-stack--architecture)
3. [Data Sources at a Glance](#3-data-sources-at-a-glance)
4. [User Roles](#4-user-roles)
5. [End-to-End Workflow (4 Screens)](#5-end-to-end-workflow-4-screens)
   - [Screen 1 — Login & OTP](#screen-1--login--otp)
   - [Screen 2 — Customer Details](#screen-2--customer-details)
   - [Screen 3 — Item Selection](#screen-3--item-selection)
   - [Screen 4 — Summary & Submit](#screen-4--summary--submit)
6. [Product Series Reference](#6-product-series-reference)
7. [Discount Logic](#7-discount-logic)
8. [Validation Rules (Smart Ordering Guards)](#8-validation-rules-smart-ordering-guards)
9. [Order Submission & Split Orders](#9-order-submission--split-orders)
10. [Duplicate & Idempotency Prevention](#10-duplicate--idempotency-prevention)
11. [Email Confirmation](#11-email-confirmation)
12. [Data Storage Map](#12-data-storage-map)
13. [Super User Reference](#13-super-user-reference)
14. [Key IDs & Configuration Constants](#14-key-ids--configuration-constants)
15. [Troubleshooting Quick Reference](#15-troubleshooting-quick-reference)

---

## 1. What Is This App?

The **PW Academy Order Form** is an internal web application used by the PW Academy sales team to place book orders on behalf of customers (distributors/retailers) for specific schools. It is built entirely on **Google Apps Script** and serves as the front-end order entry system feeding data into Google Sheets and Supabase.

**Who uses it?**
- Field Sales Employees (BDEs/ASMs) — place orders for their assigned customers and schools
- Super Users (Senior Managers / Sales Coordinators) — can place bulk orders for any customer in the system

**What does it do?**
- Authenticates employees via company email + OTP
- Lets users select a customer, a school, and a set of books with quantities
- Applies the correct trade discounts per book series
- Submits the order to a Google Sheet and logs it in a Supabase database
- Sends an HTML confirmation email to the employee and their Reporting Manager

---

## 2. Tech Stack & Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   USER'S BROWSER                        │
│   HTML + Tailwind CSS + Vanilla JS  (Index + Script)    │
└───────────────────────┬─────────────────────────────────┘
                        │  google.script.run (RPC)
                        ▼
┌─────────────────────────────────────────────────────────┐
│              GOOGLE APPS SCRIPT BACKEND                  │
│                    Code.gs                               │
│  • Auth / OTP Logic      • Submission Logic              │
│  • Google Sheet R/W      • Email via MailApp             │
│  • Google Drive Upload   • Supabase REST API calls       │
└──────────┬────────────────────────────┬─────────────────┘
           │                            │
           ▼                            ▼
┌─────────────────────┐     ┌───────────────────────────┐
│   GOOGLE SHEETS     │     │        SUPABASE            │
│                     │     │  (PostgreSQL via REST API) │
│ Source Sheet        │     │                            │
│ • SKU List tab      │     │ • emp_record               │
│  (product catalog)  │     │ • onboarding_form          │
│                     │     │ • order_form_history_data  │
│ Destination Sheet   │     │ • lvt_universe_data        │
│ • Form Responses    │     │   _school_selector         │
│  (order log)        │     └───────────────────────────┘
│                     │
│ Google Drive        │
│ • Order proof files │
└─────────────────────┘
```

**Key Technology Choices:**
| Layer | Technology | Why |
|---|---|---|
| Frontend | HTML / Tailwind CSS / Vanilla JS | Served directly by Apps Script `HtmlService` |
| Backend | Google Apps Script (V8 Runtime) | Runs in Google Cloud, no server needed |
| Database | Supabase (PostgreSQL) | Stores employee records, customers, order history |
| Spreadsheet | Google Sheets | Final order log + product catalog source |
| Auth | OTP via Gmail / MailApp | No external auth service needed |
| File Storage | Google Drive | Order proof uploads |

---

## 3. Data Sources at a Glance

| Source | What It Stores | Used For |
|---|---|---|
| `emp_record` (Supabase) | Employee name, email, RM email, ZM email | Login verification, org hierarchy |
| `onboarding_form` (Supabase) | Customer/distributor data, trade discounts, addresses, GST | Customer dropdown, pre-filling discount & shipping |
| `lvt_universe_data_school_selector` (Supabase) | Schools mapped to each employee | School dropdown |
| `order_form_history_data` (Supabase) | Line-item level order records | Reporting & analytics |
| **SKU List** tab (Source Google Sheet) | Product catalog: SKU Name, Series, Subject, Grade, MRP, No. of Books | Item selection & invoice calculation |
| **Form Responses** tab (Destination Google Sheet) | Submitted orders (78 columns, A–BZ) | Operations / dispatch team |
| Google Drive folder | Uploaded order proof files | Audit trail |

---

## 4. User Roles

### 🔵 Regular Employee
- Must verify their `@pw.live` email via OTP (valid 10 minutes, max 3 attempts)
- Sees only customers belonging to their **Zonal Manager's (ZM)** territory
- Sees only schools assigned to them specifically in `lvt_universe_data_school_selector`
- Must choose a school (or "BULK ORDER" from the dropdown)
- Discounts are **pre-filled** from the customer's onboarding record

### ⭐ Super User
- Bypasses OTP entirely — verified by email matching the `SUPER_USERS` list
- Sees **all** customers in the system (Enabled + Disabled)
- School is **locked** to `SCHOOL NOT KNOWN - BULK ORDER` automatically
- Discounts are **preset** to maximum allowed values
- Gets a gold "Super User Mode" badge in the UI

**Current Super Users:**

| Name | Email |
|---|---|
| Shubham Negi | shubham.negi@pw.live |
| Yash Khandelwal | yash.khandelwal@pw.live |
| Saksham Agarwal | saksham.agarwal@pw.live |
| Ayush Satyam | ayush.satyam@pw.live |
| Ayush Gautam | ayush.gautam@pw.live |
| Ankit Shukla | ankit.shukla2@pw.live |
| Jayashree Bera | jayashree.bera@pw.live |
| Naman Narang | naman.narang@pw.live |
| Imran Rashid | imran.rashid@pw.live |
| Aditya Tiwari | aditya.tiwari@pw.live |

> To add a new Super User, update the `SUPER_USERS` array in **Code.gs** AND the `SUPER_USERS` constant in **Script.js**.

---

## 5. End-to-End Workflow (4 Screens)

The app is a **4-page single-page application** with animated transitions. Below is the complete flow.

```
[Screen 1: Login] ──► [Screen 2: Customer Details] ──► [Screen 3: Item Selection] ──► [Screen 4: Summary & Submit]
```

---

### Screen 1 — Login & OTP

**What happens:**

```
User enters email
       │
       ▼
Is email in SUPER_USERS list?
   ├─ YES ──► Call verifySuperUserAndGetData()
   │           • Validates email exists in emp_record
   │           • Loads ALL customers (Enabled + Disabled)
   │           • Loads product catalog from SKU List sheet
   │           • Skips OTP → goes straight to Screen 2
   │
   └─ NO ──► Call verifyEmailAndSendOTP()
              • Validates email exists in emp_record
              • Generates 6-digit OTP
              • Stores OTP in Script Properties (expires 10 min)
              • Sends HTML email with OTP
              │
              ▼
        User enters OTP → Call verifyOTPAndGetData()
              • Validates OTP (max 3 attempts)
              • On success: loads product catalog, customers for ZM, schools for user
              • Goes to Screen 2
```

**Error States:**
- Email not found → "Email not found in employee list."
- OTP expired → "OTP has expired. Please request a new OTP."
- Too many attempts → "Too many invalid attempts."
- Resend OTP button resets the OTP timer

---

### Screen 2 — Customer Details

**Fields on this screen:**

| Field | Required | Notes |
|---|---|---|
| Customer / Distributor | ✅ | Searchable dropdown; Disabled customers shown greyed-out and cannot be selected |
| School | ✅ | Searchable dropdown from employee's assigned schools; always has "SCHOOL NOT KNOWN - BULK ORDER" option at bottom |
| School Email ID | ✅ | Must contain `@`; used for QBG access, lesson plans & digital content comms |
| PW Academy 2.0 Discount (%) | ✅ | Frontlist books; max 50% |
| PW Academy 1.0 Discount (%) | ✅ | Backlist books; max 50% |
| EU Next Discount (%) | ✅ | EU Next series; min 1%, max 60% |
| Kits Discount (%) | ✅ | Kit series (GST included); max 25% |
| Step By Step Discount (%) | ✅ | Step By Step series; max 50%; pre-filled to 50% |
| Semester Discount (%) | ✅ | Semester series; max 50%; pre-filled from Frontlist discount |
| Order Proof Upload | ✅ | Any file type; uploaded to Google Drive |

**How discounts are pre-filled (Regular Users):**
- Frontlist & Backlist discounts come directly from `onboarding_form.frontlist_discount` and `backlist_discount`
- EU Next → default 50%
- Kits → default 25%
- Step By Step → default 50%
- Semester → same as Frontlist discount value

**Customer Status Behaviour:**
- `Enabled` customers → selectable, shown normally
- `Disabled` customers → shown with a red "Disabled (Docs not Received)" badge, **cannot be selected**

**The "Proceed to Select Items" button** is only enabled when all required fields are valid.

> ⚠️ **If school changes after quantities were already set**, ALL quantities are automatically reset to 0 to prevent mismatched orders.

---

### Screen 3 — Item Selection

**What the user sees:**

Products are displayed in a collapsible accordion grouped by **Series → Subject/Grade**. Each item has **− / quantity input / +** controls.

**Series display order:**
1. 🔵 Frontlist (blue)
2. 🟡 EU Next (amber)
3. 🔴 Backlist (red)
4. 🟢 Kit — GST Included (green)
5. 🟣 Step By Step (indigo)
6. 🩷 Semester (pink)

**Within each series, grouping is:**
- **Frontlist / Backlist** → Bundles → Subjects → Pre-Primary Grades
- **EU Next** → Subjects (using the `EU_NEXT_SUBJECT_MAPPING` lookup table)
- **Kit** → Subjects
- **Step By Step** → Subjects
- **Semester** → Grades (Semester 1, Semester 2, etc.)

**Search:** A live search bar filters items across all series in real time.

**Running Total:** A counter at the top shows total book count selected (quantity × pack size per SKU).

---

### Screen 4 — Summary & Submit

**Recipient & Shipping Details (pre-filled from customer record):**
- Recipient Name (from `owner_name`)
- Recipient Mobile
- Shipping Address, City, State, Pincode

All fields are editable. State uses a searchable dropdown of all 36 Indian states/UTs.

**Required Dispatch Date:**
- Minimum: Today + 2 working days (T+2)
- Maximum: 10th April 2026

**Invoice Breakdown** shows a line-by-line breakdown per series:
```
Total MRP for PW Academy 2.0 (Frontlist):    ₹X,XXX.00
Discount for PW Academy 2.0 (30%):         - ₹XXX.00
...
─────────────────────────────────────────────────────
Total Invoice Value:                         ₹X,XXX.00
```

**On Submit:**
1. Validates all fields
2. Separates cart into **Kit items** and **Non-Kit items**
3. Routes to the correct submission flow (see Section 9)
4. Shows success modal with Order ID(s)
5. Confirmation email sent automatically

---

## 6. Product Series Reference

The system classifies every SKU into exactly one series based on keywords in the `Series` column of the product catalog. Classification is checked in priority order:

| Priority | Series Name | Detection Keywords | UI Color | Discount Field |
|---|---|---|---|---|
| 1 | EU Next | `eu-next` or `eu next` | 🟡 Amber | EU Next Discount |
| 2 | Kit — (GST Included) | `kit` or `gst included` | 🟢 Green | Kits Discount |
| 3 | Step By Step | `step by step` or `step-by-step` | 🟣 Indigo | Step By Step Discount |
| 4 | Semester | `semester` | 🩷 Pink | Semester Discount |
| 5 | Frontlist | `frontlist` or `2.0` | 🔵 Blue | PW Academy 2.0 Discount |
| 6 | Backlist | `backlist` or `1.0` | 🔴 Red | PW Academy 1.0 Discount |
| 7 | Other | (anything else) | Default | PW Academy 2.0 Discount |

> **EU Next subject mapping:** EU Next products use a special subject lookup table (`EU_NEXT_SUBJECT_MAPPING`) instead of the `Subject` column for display grouping. For example, the SKU "Maths Master" maps to subject "Maths", "Arteria" maps to "Art", etc.

---

## 7. Discount Logic

### How Each Series Gets Its Discount

```javascript
series === 'EU Next'              → EU Next Discount %
series === 'Step By Step'         → Step By Step Discount %
series === 'Semester'             → Semester Discount %
series === 'Backlist'             → PW Academy 1.0 Discount %
series === 'Kit - (GST Included)' → Kits Discount %
everything else (Frontlist, etc.) → PW Academy 2.0 Discount %
```

### Invoice Calculation Formula (per line item)

```
Invoice Value = (MRP × Quantity) − [(MRP × Quantity) × (Discount% / 100)]
```

The total invoice value is the sum of all line items across all series.

### Discount Caps

| Series | Max Discount | Min Discount |
|---|---|---|
| PW Academy 2.0 (Frontlist) | 50% | 0% |
| PW Academy 1.0 (Backlist) | 50% | 0% |
| EU Next | 60% | 1% |
| Kits | 25% | 0% |
| Step By Step | 50% | 0% |
| Semester | 50% | 0% |

---

## 8. Validation Rules (Smart Ordering Guards)

When a user selects items on Screen 3, the system applies real-time validation rules to prevent logically incorrect orders. These rules are **only active for specific school orders** — they are **bypassed for Bulk Orders** (`SCHOOL NOT KNOWN - BULK ORDER`).

---

### Rule 1 — EVS ↔ Science / Social Science Mutual Exclusion

**Problem it solves:** Schools either teach EVS *or* separate Science & Social Science — not both — for a given grade.

**Logic:**
- If you select **EVS** for Grade X → Science and Social Science items for Grade X are **disabled**
- If you select **Science or Social Science** for Grade X → EVS items for Grade X are **disabled**
- For Grades 1–2, any Semester series item is also disabled if EVS (or Science/Social Science) is already selected
- For Grades 3–5, Semester series items are disabled if a non-Semester Science or Social Science is selected

---

### Rule 2 — Cross-Series Subject Blocking

**Problem it solves:** The same subject (e.g., Maths, Grade 3) should not be ordered from two different series (e.g., both Frontlist and Semester).

**Logic:**
- Once you select a SKU for `Subject X` + `Grade Y` from `Series A`, all other SKUs for `Subject X` + `Grade Y` from `Series B, C, D...` are **disabled**

---

### Rule 3 — Semester Bidirectional Blocking

**Problem it solves:** Schools that use Semester series have different curricula from Frontlist/Backlist — they should not be mixed.

**Logic (for Grades 1–5 only):**
- If you select a **Semester** item for Grade X → Non-Semester items for "blocked subjects" in Grade X are **disabled**
- If you select a **non-Semester** item for a "blocked subject" in Grade X → Semester items for Grade X are **disabled**

**Blocked subjects by grade:**
- Grades 1–2: English Literature, Maths, EVS, GK, Science, Social Science
- Grades 3–5: English Literature, Maths, Science, Social Science, GK, EVS

---

### Rule 4 — Same-Grade Quantity Consistency (Non-Semester)

**Problem it solves:** For a given grade (e.g., Grade 3), all subjects ordered must have the same quantity (you can't order 30 Maths books and 20 Science books for the same class).

**Logic:**
- The **first SKU** you set a quantity for in a grade "establishes" the quantity for that grade
- All subsequent SKUs in the same grade are **auto-filled** to that established quantity when first selected
- Attempting to set a higher quantity on a non-source SKU shows a **tooltip error**: *"Order quantity of all subjects of a class must be the same"*
- Decreasing a quantity → that SKU goes to 0 directly
- Changing quantity on the **source SKU** updates all other selected SKUs in that grade

---

### Semester-Specific Rule — Textbook ↔ Workbook Sync

**Problem it solves:** For Semester series, Textbooks and Workbooks are paired. Workbook quantity cannot exceed Textbook quantity.

**Logic:**
- Changing **Textbook** quantity → **Workbook** for the same Grade + Semester is **auto-synced** to the same quantity
- Attempting to set Workbook quantity **above** Textbook quantity → tooltip error: *"You cannot increase the quantity for the Workbook beyond the Textbook quantity for the same semester and grade."*
- Decreasing Textbook to 0 → Workbook also set to 0 automatically

---

### Visual Indicators

| State | Appearance |
|---|---|
| Available SKU | Normal white card |
| Disabled SKU | 50% opacity, grey background, controls disabled |
| Hover on disabled SKU | Red "Blocked by validation rule" tooltip badge appears |
| Rule active banner | Amber banner at top of item list |

---

## 9. Order Submission & Split Orders

### Normal Flow (No Kit Items)

```
finalSubmitButton clicked
        │
        ▼
Validate all fields
        │
        ▼
All items are non-Kit
        │
        ▼
Call submitSampleSubmission({ isKitOrder: false, items: [...] })
        │
        ▼
Backend:
  1. Acquire script lock (prevents concurrent submissions)
  2. Idempotency check (reject if requestId seen before)
  3. Recent submission check (same customer+school within 30s?)
  4. Generate Order ID: K8OD00001, K8OD00002, ...
  5. Upload proof file to Google Drive
  6. Write row to "Form Responses" sheet (78 columns, A–BZ)
  7. Insert line-item records to Supabase order_form_history_data
  8. Send confirmation email to employee + CC Reporting Manager
  9. Mark customer+school as "recently submitted"
  10. Release lock
        │
        ▼
Show success modal with Order ID
```

### Split Order Flow (Mixed Kit + Non-Kit Items)

When a cart contains **both** Kit and non-Kit items, the system automatically **splits it into two separate orders**:

```
Submit clicked (cart has Kit + Non-Kit items)
        │
        ▼
Step 1: Submit Normal Order (non-Kit items)
        → Gets Order ID: K8OD00042
        │
        ▼ (on success)
Step 2: Submit Kit Order (Kit items)
        → Gets Order ID: K8OD00042_KIT
        │
        ▼
Show "Split Order" success modal:
  "📦 Normal Order ID: K8OD00042"
  "🎁 Kit Order ID:    K8OD00042_KIT"
  "You will receive 2 confirmation emails."
```

> **Note:** History records in Supabase are only written once (during the Normal Order submission) for all combined items. The Kit order submission sets `skipHistoryInsert: true`.

### Order ID Format

```
K8OD00001        → Normal order
K8OD00001_KIT    → Kit portion of a split order
```

IDs are sequential, generated by scanning the max existing ID in the sheet.

---

## 10. Duplicate & Idempotency Prevention

The system has **three layers** of duplicate prevention:

| Layer | Mechanism | Window |
|---|---|---|
| **Request ID** | Each submit generates a UUID `requestId`; backend checks Script Properties and rejects if seen before | 5 minutes |
| **Recent Submission** | Backend checks if same `customerCode + schoolName` was submitted recently | 30 seconds |
| **Script Lock** | `LockService.getScriptLock()` ensures only one submission runs at a time | 45 seconds max |

If a duplicate is detected, the user sees an error with the original Order ID:
> *"Order was recently submitted (ID: K8OD00042). Please wait 30 seconds before submitting again."*

---

## 11. Email Confirmation

A confirmation email is sent automatically after every successful submission.

**Recipients:**
- `To:` Employee's email
- `CC:` Reporting Manager's email

**Subject line examples:**
- `Order Placed Successfully - Order ID: K8OD00042`
- `[KIT ORDER] Order Placed Successfully - Order ID: K8OD00042_KIT`
- `[SUPER USER ORDER] Order Placed Successfully - Order ID: K8OD00042`

**Email content includes:**
- PW Academy logo
- Kit badge (if applicable)
- Order status tracker (Placed → ZM Approval → Shipped → Delivered)
- Invoice value + discount breakdown
- Shipping address
- Full order summary table (Book Name | Quantity)
- Customer details (Trade Name, Customer Code, GST, School)
- Contact info for Sales Coordinator (Jayashree Bera)

---

## 12. Data Storage Map

### Google Sheet — "Form Responses" Tab (78 Columns A–BZ)

| Column | Header | Data |
|---|---|---|
| A | Timestamp | Date/time of submission |
| B | Submission ID | K8OD##### or K8OD#####_KIT |
| C | Employee Email ID | Logged-in user's email |
| D | RM Email ID | Reporting Manager's email |
| E | ZM Email ID | Zonal Manager's email |
| F | Employee Name | Employee's name |
| G | Company Trade Name | Customer's trade name |
| H | Customer Code | Customer code |
| I | Customer Code Trade City | Combined customer identifier |
| J | Recipient Name | Recipient at delivery address |
| K | Recipient Mobile | Delivery contact number |
| L | Customer Email | Customer's firm email |
| M | School Name | Selected school (or BULK ORDER) |
| N | Shipping Address | Full delivery address |
| O | Shipping City | — |
| P | Shipping State | — |
| Q | Shipping Pincode | — |
| R–U | Business Address | Business address fields |
| V | PW Academy 2.0 Discount | e.g., "30%" |
| W | PW Academy 1.0 Discount | e.g., "20%" |
| X | Proof Upload | Google Drive link |
| Y | SKU Info | All items: `SKU Name $ qty // SKU Name $ qty` |
| AA | Required Dispatch Date | MM-DD-YYYY format |
| BO | EU Next Discount | e.g., "50%" |
| BP | Firm GST Number | Customer's GST |
| BQ | School Email ID | Entered school email |
| BV | Kits Discount | e.g., "25%" (kit orders only) |
| BY | Step By Step Discount | e.g., "50%" |
| BZ | Semester Discount | e.g., "30%" |

### Supabase — `order_form_history_data` Table

One row per SKU line item. Key fields:

| Field | Description |
|---|---|
| `submission_id` | Links to the Google Sheet order ID |
| `order_date` | ISO timestamp |
| `employee_email / name` | Who placed the order |
| `rm_email / zm_email` | Org hierarchy |
| `customer_code / trade_name` | Customer info |
| `school_name / school_email_id` | School info |
| `sku_name / sku_quantity` | What was ordered |
| `sku_series` | Classified series (Frontlist, EU Next, etc.) |
| `sku_subject / sku_grade / sku_mrp` | Product metadata |
| `frontlist_discount` | Applied % |
| `backlist_discount` | Applied % |
| `eu_next_discount` | Applied % |
| `kit_discount` | Applied % |
| `step_by_step_discount` | Applied % |
| `semester_discount` | Applied % |

---

## 13. Super User Reference

| Feature | Regular User | Super User |
|---|---|---|
| Authentication | OTP (10-min expiry, 3 attempts) | Email match only, no OTP |
| Customer visibility | ZM's territory only | All Enabled + Disabled customers |
| School selection | From assigned school list | Locked to "BULK ORDER" |
| Default discounts | From customer's onboarding record | Max values (FL:50%, BL:50%, EU:60%, Kit:25%, SBS:50%, Sem:50%) |
| Email subject tag | (none) | `[SUPER USER ORDER]` prefix |
| Order ID | K8OD##### | K8OD##### (same format) |
| Badge in UI | None | Gold "⭐ Super User Mode" badge |

---

## 14. Key IDs & Configuration Constants

> These are in `Code.gs`. Do not change without coordinating with the tech team.

| Constant | Value / Purpose |
|---|---|
| `SUPABASE_URL` | `https://kfkcohosbpaeuzxuohrm.supabase.co` |
| `SOURCE_SHEET_ID` | Google Sheet containing the **SKU List** product catalog |
| `SAMPLE_SHEET_TAB_NAME` | `SKU List` (tab name in the source sheet) |
| `DESTINATION_SHEET_ID` | Google Sheet where **Form Responses** are written |
| `SUBMISSIONS_TAB_NAME` | `Form Responses` (tab name in destination sheet) |
| `UPLOAD_FOLDER_ID` | Google Drive folder ID for order proof uploads |

---

## 15. Troubleshooting Quick Reference

| Symptom | Likely Cause | What to Check |
|---|---|---|
| "Email not found in employee list" | Email not in `emp_record` Supabase table | Ask admin to add the employee record |
| No customers showing after login | ZM email missing in `emp_record` or no customers mapped to that ZM | Check `zm_email_id` in `onboarding_form` |
| No schools showing in dropdown | Employee not mapped in `lvt_universe_data_school_selector` | Check `employee_email` + `status=Active` in that table |
| "Disabled (Docs not Received)" on customer | `customer_status = Disabled` in `onboarding_form` | Customer cannot be selected; escalate to onboarding team |
| Items greyed out on Screen 3 | Validation Rules 1–3 active | Check which subject/series is already selected for that grade |
| "Order quantity of all subjects must be the same" | Rule 4: trying to set different qty for same grade | Reduce all same-grade items to 0 first, then re-enter |
| "This request has already been processed" | Duplicate submission within 5-minute window | Wait and retry if genuinely new order |
| Order not in Google Sheet | Script lock or server error | Check Apps Script execution logs (Stackdriver) |
| OTP not received | Email delivery delay or wrong email | Wait 2 minutes; use Resend OTP button |
| "Could not obtain lock" | Concurrent submissions from same script | Retry after 30 seconds |

---

## Appendix — Order Lifecycle

```
Employee fills form ──► [Order Placed] ──► ZM Approval ──► Shipped ──► Delivered
         │                    │
         │              Email sent to
         │              employee + RM
         │
         ├── Google Sheet "Form Responses" updated (operations team)
         └── Supabase order_form_history_data updated (analytics)
```

The confirmation email shows the **order tracking stages** visually. The email states:
> *"You will receive the next update when your order is approved by your ZM."*

---

*Last updated: March 2026 | App Version: 4.0 | Maintained by: Sales Tech Team*
