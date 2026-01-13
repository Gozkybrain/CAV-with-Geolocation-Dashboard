# Contact Address Verification (CAV) System

A production-grade **Contact Address Verification (CAV)** system with **role-based workflows**, **multi-step verification**, **invite-only registration**, and **Excel data portability**.

This system is designed to scale across compliance-heavy platforms (ads, fintech, logistics, marketplaces).

---

## Tech Stack

> **Note:** This architecture intentionally **does NOT use Firebase Cloud Functions**.  
All logic runs in **Next.js server routes / server actions** for full control and portability.


## Frontend
- **Next.js (App Router, Fullstack)**
  - Server Actions
  - Route Handlers (`/api/*`)
- **React**
  - User, Agent, Admin dashboards
- **CSS / CSS Modules**
  - Custom styling (no Tailwind)

---

## Authentication & Authorization
- **Firebase Authentication**
  - Email & Password
  - Custom claims for roles:
    - `user`
    - `moderator`
    - `admin`
- **Admin Invite / Registration Code System**
  - Admin-generated codes
  - Stored in Firestore
  - Required during registration

---

## Database & Storage
- **Firebase Firestore**
  - Users
  - Address documents
- **Storage**
  - Cpanel file directory
  - On-site photo proofs
- **Notifications (No Cloud Functions)**
    - NodeMailer (SMTP)
    - Triggered from API routes

---

## Geo-location & Address Verification

### Address → Coordinates (Geocoding)
**Configurable via ENV**

Recommended APIs:
- **Google Maps Geocoding API**
- **Mapbox Geocoding API**
- **OpenCage Geocoder** (cost-effective, dev-friendly)

Recommended NPM packages:
- `@googlemaps/google-maps-services-js`
- `mapbox-sdk`
- `opencage-api-client`


## Distance Calculation (No External API)

**Haversine Formula**

- Pure JavaScript utility function
- No third-party map or distance API required
- Runs in:
  - Next.js API Routes
  - Server Actions

**Recommended Package (Optional)**

- `geolib`

**Example**

```js
import { getDistance } from "geolib";

const distance = getDistance(
  { latitude: userLat, longitude: userLng },
  { latitude: agentLat, longitude: agentLng }
);

const isValid = distance <= 100; // distance in meters
```

---

## Excel Import / Export System
Excel → JSON (Upload Flow)

**Recommended Packages**

- `xlsx` (primary)

- `exceljs` (advanced formatting support)

**Flow**

- User uploads `.xlsx` file

- File is parsed in a Next.js API route

- Converted into JSON

- Stored in Firestore

**Example**

```js
import * as XLSX from "xlsx";

const workbook = XLSX.read(buffer);
const sheet = workbook.Sheets[workbook.SheetNames[0]];
const jsonData = XLSX.utils.sheet_to_json(sheet);
```

## JSON → Excel / CSV (Download Flow)

**Recommended Packages**

- `xlsx` (Excel export)

- `papaparse` (CSV export)

**Flow**

- Fetch filtered documents from Firestore

- Convert JSON to worksheet

- Export as `.xlsx` or `.csv`

**Example**

```js
const worksheet = XLSX.utils.json_to_sheet(data);
const workbook = XLSX.utils.book_new();

XLSX.utils.book_append_sheet(workbook, worksheet, "Verified");
XLSX.writeFile(workbook, "verified-addresses.xlsx");
```

## Supported Export Filters

- Verified documents

- Unverified documents

- Rejected documents

- All documents

---

## Core Concept

CAV ensures that:
- Addresses are **submitted by users**
- **Moderators review** based on jurisdiction
- **Admins give final approval**
- All actions are **auditable, role-aware, and location-constrained**. A document can not be updated by a moderator that is not at the correct location.

---

## Roles & Permissions

### User
- Registers using an **admin-issued registration code**.
- Submits contact address & data as excel file.
- Can export verified data as excel file.
- Cannot verify anything.
- Can view verification status.

### Moderator
- Reviews assigned documents only.
- **Cannot approve if document location ≠ moderator jurisdiction**.
- Can recommend approval or rejection.
- Cannot finalize verification.

### Admin
- Issues registration codes.
- Manages users & moderators.
- Final authority on verification.
- Overrides location restrictions.
- Imports / exports Excel data.
- Assigns moderators to document.

---

## Invite-Code Based Registration

### Why?
- Prevents random sign ups
- Enables controlled onboarding
- Useful for internal tools & compliance systems

### Flow
1. Admin generates a **registration code**
2. Code is shared with intended user
3. User must supply code during sign up
4. Code is validated before account creation

---

### 📦 Registration Code Schema

```js
{
  _id: ObjectId,
  code: String,
  role: "user" | "moderator",
  fullName: String,
  email: String,
  phoneNumber: String,
  organization: String,
  createdAt: Date
}
```

# CAV Roadmap

## Data Ingestion (User → System)

### Excel Import
- A **user uploads an Excel (.xlsx) file**
- The file contains:
  - Personal user data
  - Contact address data
- The system:
  1. Parses the Excel file
  2. Converts rows into structured JSON
  3. Stores the data in **Firebase (Firestore)**

### Key Rules
- No verification happens at this stage
- Imported data is marked as `unverified`
- Each row becomes a **verification document**

---

### Sample Excel Columns
```txt
fullName | email | phone | address | city | state | country
```

## Data Storage (Firebase)

**Document State After Import**

- Stored as JSON in Firestore
- Linked to the uploading user
- Status: `pending_assignment`

**At this point:**

- Documents are read-only
- No moderator access yet

---

## Moderator Assignment (Admin → Moderator)

**Admin Responsibilities**

- Admin views all imported documents
- Admin assigns documents to moderators
- Assignment is location-aware

**Assignment Rules**

- Each document has an address (state/location)
- Moderator must have matching jurisdiction
- Admin can reassign or override if needed

**After assignment:**

- Document status → `assigned_to_moderator`

---

## Physical Address Verification (Moderator)

**Moderator Workflow**

- Moderator travels to the actual physical address
- Confirms:
  - Address exists
  - Address is reachable
  - Matches submitted data
  - Makes other findings

**Location Enforcement (Critical)**

- Moderator’s current GPS location is checked
- If moderator is not within the document’s location:
  - Upload is blocked
  - Verification fails immediately
  - No data is saved

**Proof Upload**

- Moderator uploads:
  - Photo evidence of the address
  - Optional notes

**Verification Outcome**

- **Success:**
  - Verification data is saved
  - Status → `moderator_verified`
- **Failure (if applicable):**
  - Address does not exist
  - Address is unreachable
  - Submitted data is vague or incorrect
  - Status → `verification_failed`
  - Moderator can add notes explaining the reason
---

## Final Approval (Admin)

**Admin Review**

- Admin reviews:
  - Submitted address data
  - Moderator’s proof image
  - Moderator notes
  - Location metadata

**Admin Decision**

- Admin has final authority
- Admin can:
  - Approve → `verified`
  - Reject → `rejected`

> ⚠️ Moderator approval alone is never final

---

## Data Export (User)

**Export Capabilities**

- User can export documents at any time
- Export formats:
  - Excel (`.xlsx`)

**Export Filters**

- Verified documents
- Unverified documents
- Rejected documents
- All documents

### Core Principles

- Bulk first (Excel → JSON)

- Physical verification is mandatory

- Location mismatch = hard fail

- Admin approval is final

- Data must remain exportable


## Architecture Summary

- **No Firebase Cloud Functions**
  - All business logic is handled within the application layer for full control and portability.

- **Execution Layer**
  - **Next.js API Routes**
    - Data processing
    - Validation
    - File handling
    - Role-based actions
  - **Next.js Server Actions**
    - Secure form submissions
    - Authenticated mutations
    - Server-side enforcement

---

## Modular CAV Workflow

1. **Upload → Parse → Store**
   - User uploads Excel documents
   - Data is converted to JSON
   - Records are stored in Firestore

2. **Assign → Verify → Geo-check**
   - Admin assigns documents to field agents
   - Agent verifies on-site using GPS + photo proof
   - Distance validation enforced (Haversine logic)

3. **Review → Approve → Export**
   - Admin reviews verification data
   - Final approval or rejection
   - Verified data available for export (Excel / CSV)

---

> This architecture ensures a **traceable**, **secure**, and **scalable** Contact Address Verification system without reliance on backend serverless functions.
