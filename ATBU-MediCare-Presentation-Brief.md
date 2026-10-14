# ATBU MediCare: System Analysis and Presentation Brief

Prepared from the application source in this repository on 15 October 2026. This document is intended as source material for generating presentation notes. It separates verified code behavior from external Supabase services and unverified claims.

## 1. Executive Summary

ATBU MediCare is a browser-based healthcare management application for the Abubakar Tafawa Balewa University community. Its stated project context is Team Tech Fusion's 2026 Interdisciplinary Knowledge and Innovation Summit project at ATBU Bauchi. It models four user roles: patient, doctor, pharmacist, and administrator.

The application brings several clinic workflows into one role-based system:

- Patients create accounts, book doctor appointments, view visit history and prescriptions, submit payment details for verification, and maintain their profile and medical record.
- Doctors view their roster, process appointment requests, consult patients, record advice and medical notes, and prepare prescriptions.
- Administrators manage staff accounts and roster shifts, review payments, inspect appointments and patient records, and monitor pharmacy inventory.
- Pharmacists manage inventory and fulfill prescriptions after payment has been confirmed, with dispensing delegated to a backend procedure that checks stock.

The visible application is implemented as static HTML, CSS, and JavaScript pages. Supabase provides authentication, database access, and RPC/Edge Function operations. The repository does not contain the SQL schema/migrations or the Edge Function source, so the browser app is only one part of the complete running system.

### Short pitch

> ATBU MediCare connects appointment booking, clinical consultations, prescriptions, payment verification, and medicine dispensing in one role-based campus healthcare workflow. Patients, clinicians, administrators, and pharmacists each use the parts of the system relevant to their responsibilities, while Supabase coordinates identity and shared records.

## 2. Project Identity and Scope

- **Product name in the active app:** ATBU MediCare.
- **Institution:** Abubakar Tafawa Balewa University (ATBU), Bauchi, Nigeria.
- **Project team:** Team Tech Fusion.
- **Stated context:** 2026 Interdisciplinary Knowledge and Innovation Summit at ATBU Bauchi.
- **Stated scope:** Patient, doctor, pharmacist, and admin workflows covering appointments, prescriptions, and records.
- **Currency shown in the application:** Nigerian naira (NGN / `₦`).
- **License file:** Apache License 2.0.

### Team listed in the application

The landing page credits Team Tech Fusion with these names and roles:

- Aliyu Auwal Makama: CEO and Founder.
- Mujahid Adamu Aliyu: Database and Cybersecurity Lead.
- Aminat Ahmad Othman: UI/UX Designer.
- Alamin Muhammad: Technical Architect.
- Buhari Shuaibu: Lead Developer.
- Farouq Abdulkadir Umar: Innovation Intern.

This is a healthcare management application prototype/project, not a hospital information system whose regulatory certification, production readiness, or clinical validation is established by the repository. The source demonstrates workflows; it does not prove deployment scale, real-world adoption, or medical outcomes.

## 3. Application Architecture

### Browser application

The root directory contains seven HTML screens:

| File | Responsibility |
|---|---|
| `index.html` | Public landing page, patient sign-up, sign-in, Google sign-in, and forgot-password request |
| `admin.html` | Administration dashboard |
| `doctor.html` | Doctor dashboard |
| `patient.html` | Patient dashboard |
| `pharmacist.html` | Pharmacy dashboard |
| `confirm.html` | Email confirmation callback screen |
| `reset-password.html` | Password recovery callback and password update screen |

Each page contains its own markup and page-specific scripts. There is no app bundler, JavaScript package manifest, or application build pipeline in this repository. The main pages load Supabase's JavaScript client from a CDN and generally use ES modules; the public page and confirmation/recovery pages use the Supabase browser global. Icons are from Phosphor Icons and the Poppins font is loaded from Google Fonts. The patient page also loads `html2canvas` from a CDN to export receipts as PNG images.

### Backend services

The browser clients connect to a Supabase project and use Supabase Auth plus Postgres data APIs. Source code references the following database objects:

- Tables or views: `user_roles`, `patients`, `doctors`, `pharmacists`, `admins`, `doctor_slots`, `appointments`, `prescriptions`, `prescription_items`, `payments`, `drug_stock`, and `patient_medical_records`.
- RPC functions: `complete_patient_profile`, `book_appointment`, `generate_payment_reference`, `create_payment_for_prescription`, `cancel_receipt`, `dispense_prescription`, and `link_email_identity`.
- Supabase Edge Functions: `check-and-reset-password`, `create-staff-user`, and `delete-user`.

These names and their call sites are visible in the browser code. The implementation of the functions, database constraints, row-level security policies, triggers, and migrations are not included here. The app therefore assumes the Supabase project has already been configured to match these contracts.

### High-level data relationships implied by page queries

The code suggests the following model; treat this as the client-visible relationship map, not a complete or authoritative database schema:

- An authenticated user has a row in `user_roles` which identifies the role. A profile row with the same user ID is read from the matching role table (`patients`, `doctors`, `pharmacists`, or `admins`).
- A doctor owns roster slots in `doctor_slots`. A slot contains a date, shift, capacity, and booked count.
- An `appointments` row connects a patient to a doctor slot. The client reads fields such as appointment code, assigned time, reason, status, cancellation note, advice, and medical record notes.
- A `prescriptions` row is associated with the appointment, doctor, and patient. `prescription_items` connect the prescription to `drug_stock` and store dosage, frequency, duration, quantity, and price at the time of prescribing.
- `payments` are associated with a patient and prescription. The client reads payment code, amount, status, method, reference, and verification metadata.
- `patient_medical_records` stores patient-entered values including genotype, blood group, allergies, chronic conditions, and emergency contact details.

### Deployment configuration

`wrangler.jsonc` configures Cloudflare Wrangler with worker name `fud-hms`, compatibility date `2026-07-13`, observability enabled, and static assets served from the repository root. This is only deployment configuration; it does not include Supabase schema or Edge Function source. The old `fud-hms` worker name and the `fud-logo.png` filename are legacy identifiers in the ATBU snapshot, not the active product name shown in the app.

## 4. User Roles and Capabilities

### Patient

The patient dashboard contains five main areas:

1. **Dashboard:** Summary counts for appointments, upcoming appointments, active prescriptions, and unpaid payments, plus a recent-appointments view.
2. **Appointments:** Appointment list with status filtering and pagination. Patients can book with a four-step wizard and cancel appointments that are pending or approved.
3. **Prescriptions:** Two subviews: payment/fulfillment transaction status and medical instructions (doctor advice and prescribed items).
4. **Payments:** Payment list with status filtering, submission of payment method/reference for staff verification, receipt viewing, and PNG receipt export.
5. **Profile:** Editable personal/contact/address fields, password changes, and patient medical-record management.

The booking wizard asks the patient to select an active doctor, select an available future shift, enter a reason for the visit, review the details, and confirm. The client uses the `book_appointment` RPC to create the booking and then fetches an appointment code for display. The server-side function is important because capacity and duplicate-booking rules must be enforced reliably on the backend.

Google OAuth is initiated from the landing page with a redirect to `patient.html`. The patient page checks the user's role and signs out non-patient accounts. New Google users without a patient profile are routed through a required profile-completion modal that calls `complete_patient_profile`.

### Doctor

The doctor dashboard contains six areas:

1. **Dashboard:** Counts of upcoming roster shifts, pending requests, upcoming appointments, and historical visits.
2. **My Roster:** Future shifts assigned to the doctor, including date, day/night shift, capacity, and booked count.
3. **Appointments:** Separate pending and approved views. The doctor can approve a pending appointment or decline/cancel it with a patient-visible reason. Approved appointments can be taken into consultation.
4. **History:** Completed, missed, or cancelled visits.
5. **Patients:** Search is scoped to patients with an appointment relationship to the doctor; recent patients are also shown.
6. **Profile:** Read-only profile data and password/security controls.

In the consultation form, the doctor can view the patient's medical record (when permitted), enter advice for the patient, enter an internal medical note, and add one or more prescription items. Each item includes a stocked drug, dosage, frequency, duration, and quantity. The client checks that a drug is in stock and that the requested quantity does not exceed the cached quantity, then asks for confirmation.

On submission, the browser performs a sequence: mark the appointment completed and save the advice/medical note; insert a prescription; insert prescription items; call `create_payment_for_prescription`. Because these are separate client-side requests rather than a single transaction shown in the repository, a failure after an earlier request can leave partial work. The UI reports partial failures, but this is a backend/transaction design point to verify before production use.

### Administrator

The admin dashboard contains eight areas:

1. **Dashboard:** Counts active doctors, active pharmacists, patients, pending payment verifications, and low-stock items.
2. **Doctors:** Browse/filter active and deactivated doctors, activate/deactivate staff, add a doctor, or permanently delete one.
3. **Pharmacists:** Similar lifecycle management for pharmacy staff.
4. **Roster:** Schedule shifts, inspect occupancy, and remove future shifts with no bookings.
5. **Payments:** Filter by payment status, mark submitted payments paid or reject them, and cancel a paid receipt with a reason. The cancel RPC rejects a receipt if its prescription is already dispensed.
6. **Appointments:** Browse/filter appointment records and see patient, date, doctor, shift, and status information.
7. **Patients:** Search patients and open a profile containing contact/demographic details, medical record information, appointment history, and nested prescription details, subject to backend permissions.
8. **Stock:** Read-only overview of pharmacy stock and reorder-threshold status.

Adding staff validates a name, licence pattern (`LIC-...` for doctors and `PH-...` for pharmacists), phone, and `@atbu.edu.ng` email. The page checks email and licence uniqueness across both staff tables, then calls the authenticated `create-staff-user` Edge Function. The function may return a temporary password which is displayed in a success card for sharing. Permanent staff deletion requires typing the staff email exactly and calls `delete-user`; the source warns that deleting a doctor cascades to historical appointment, prescription, and payment records. This is a destructive operation and should be demonstrated only with disposable test data.

Roster shifts are day (`09:00-15:00`) or night (`18:00-22:00`) and have a capacity. Adding a shift requires a future date. The reset-roster action deletes upcoming slots whose `booked_count` is zero; it leaves populated shifts alone.

### Pharmacist

The pharmacist dashboard contains six areas:

1. **Dashboard:** Counts awaiting-payment prescriptions, paid prescriptions ready to dispense, low-stock items, and total dispensed prescriptions; also shows low-stock alerts.
2. **Stock:** Inventory table showing drug, unit/type, quantity, price, and threshold status.
3. **Awaiting:** Active prescriptions whose payment is not paid; dispensing is locked.
4. **Ready:** Active prescriptions whose payment is paid; these can be dispensed.
5. **Dispensed:** Historical completed dispensing records.
6. **Profile:** Read-only pharmacist profile and password/security controls.

Stock management supports adding a drug (name, unit/type, quantity, price, reorder threshold) and updating an existing drug. The update form adds to the existing quantity and can change price and threshold. Low-stock status is calculated when quantity is less than or equal to the reorder threshold.

Dispensing calls `dispense_prescription` rather than directly editing stock and prescription state in the browser. The UI handles server errors for payment not confirmed and insufficient stock. The RPC is the critical place for atomic stock deduction, payment checks, and status transitions; its implementation is not in this repo.

## 5. End-to-End Business Workflows

### A. Patient appointment to completed consultation

1. A patient registers through the public page or signs in using email/password; Google sign-in is also offered for patients.
2. The page checks `user_roles` and loads the matching patient profile.
3. The patient selects an active doctor and a future slot with remaining capacity.
4. The patient enters a reason, reviews the appointment, and confirms it.
5. `book_appointment` creates the appointment and the UI displays its appointment reference.
6. The doctor sees the appointment as pending and approves it or cancels it with a reason.
7. For an approved appointment, the doctor opens a consultation, reviews relevant medical history, and records advice and an internal note.
8. The doctor adds stock-backed prescription items and confirms the summary.
9. The app marks the visit completed, writes the prescription and items, then requests a payment record through `create_payment_for_prescription`.
10. The patient can view the visit, advice, prescription, and payment status; the pharmacist sees the prescription in the appropriate queue.

### B. Prescription payment and dispensing

1. Doctor consultation creates an active prescription and payment record.
2. The patient sees the invoice/payment amount and submits one of the listed methods: card, bank transfer, or cash at counter. The app generates or requests a payment reference.
3. Submitting a payment changes its status to `pending_verification`; it does not charge a card or process a bank transfer.
4. An administrator verifies the submitted payment by marking it paid or rejecting it back to unpaid.
5. The pharmacist sees an active prescription as awaiting until its payment is paid.
6. The pharmacist dispatches it through `dispense_prescription`.
7. The backend procedure should confirm payment and stock, deduct inventory, and mark it dispensed. Exact server logic must be verified from the missing Supabase function/schema source.
8. The patient sees fulfillment state; paid-but-not-dispensed is distinct from dispensed.

### C. Account lifecycle

1. Patient sign-up submits profile values as Auth user metadata and requests a confirmation email to `confirm.html`.
2. Email confirmation supports Supabase OTP/PKCE token parameters and a legacy access/refresh-token hash flow. It asks the user to click Confirm, verifies the token, signs out, and shows success or failure.
3. Staff accounts are created by an admin through the server-side `create-staff-user` function.
4. Staff records may have `must_change_password`; the doctor and pharmacist pages gate the dashboard until a new password is set and the flag is cleared.
5. The public forgot-password form calls `check-and-reset-password`, redirecting to `reset-password.html`.
6. The reset page exchanges PKCE codes or reads a recovery session, requires a password of at least eight characters, calls `updateUser`, attempts `link_email_identity` for OAuth-only accounts, signs out, and returns to the public page.

## 6. Key Statuses and Rules Visible in the Client

### Appointment statuses

The client explicitly handles `pending`, `approved`, `completed`, `missed`, and `cancelled`.

- Patient cancellation is offered only for pending or approved appointments.
- Doctors approve pending appointments, cancel with a required note, and complete an appointment through the consultation flow.
- Doctor history includes completed, missed, and cancelled appointments.
- The database must enforce status/ownership rules; front-end checks are not an authorization boundary.

### Payment statuses

The client explicitly handles `unpaid`, `pending_verification`, and `paid`.

- `unpaid`: patient can submit payment details.
- `pending_verification`: admin decides whether to mark paid or reject.
- `paid`: prescription is eligible for pharmacist fulfillment unless already dispensed.
- An admin can cancel a receipt with a reason, but the RPC refuses cancellation after dispensing.

### Prescription statuses

The client relies chiefly on `active` and `dispensed` for pharmacy workflow. Display state combines prescription status and payment status, so a paid prescription is not represented as dispensed until a pharmacist completes the dispensing step.

### Inventory rule

For a drug, the low-stock alert condition is `quantity <= reorder_threshold`. The doctor checks availability and quantity before submission; the dispensing RPC must check again because inventory can change between the doctor's check and pharmacy fulfillment.

## 7. Technical Design and User Experience

- **Navigation:** Each dashboard has role-specific tabs, a mobile drawer, and a desktop left-side navigation rail.
- **Responsive layout:** Desktop and mobile navigation/layout modes differ; mobile workflows use bottom-sheet-style modals.
- **Feedback:** Loading skeletons, refresh overlays, inline errors, toast messages, confirmation dialogs, and empty states are implemented in the pages.
- **Forms:** Custom select/calendar/date-of-birth controls are used in patient and staff flows. Doctor prescription entry uses native selects for dosage units, frequency, and duration.
- **Theme:** Light/dark CSS variables; the `atbu-theme` preference is stored in localStorage for the dashboards and auth pages.
- **Accessibility efforts:** Buttons and fields include labels/ARIA attributes in many places; native selects are used in prescription entry for keyboard/screen-reader support. This should not be described as a completed accessibility audit.
- **Visual assets:** `logo.png`, `og-image.png`, legacy-named `fud-logo.png`, and Team Tech Fusion portrait assets. The active pages render ATBU MediCare branding; the old asset filename is retained in this snapshot.

## 8. Security, Privacy, and Operational Caveats

These points are important to describe accurately and to verify before any real patient-data use:

1. **The browser uses a Supabase anon/public key.** It is embedded in the HTML pages. That key is intended for browser use only when Supabase row-level security (RLS) and grants correctly restrict each role. The repository does not contain the RLS policies or schema, so their correctness cannot be assessed from this checkout. Never expose a Supabase service-role key in browser code or a presentation.
2. **Some authorization checks are client-side.** Dashboard pages check the session and role before showing content, but backend RLS/functions must independently enforce authorization. A browser redirect is not a security control by itself.
3. **Sensitive data exists in scope.** Patient demographics, contact details, medical history, prescriptions, and payment metadata are accessed. Use synthetic accounts and data in the presentation; do not project real patient information or credentials.
4. **No payment processor integration is shown.** The patient selects a payment method and submits a reference for manual verification. The repo does not show card acquiring, bank-transfer reconciliation, or settlement integration.
5. **Critical backend code is absent.** SQL schema/migrations, RLS policies, triggers, and three Edge Functions are not included. RPC contracts are called by name, but their implementation is unavailable here.
6. **Consultation writes are multiple browser requests.** Appointment completion, prescription creation, prescription-item insertion, and payment creation happen sequentially. A failure partway through may leave partial data unless the backend has additional safeguards. The code reports some partial failures; it does not show a single transaction wrapping the workflow.
7. **Staff deletion is intentionally destructive.** The admin UI warns that deleting a doctor removes linked history via cascading deletion. Avoid demonstrating this against non-test records.
8. **Static homepage metrics are not verified.** The landing page displays `2,400+ Patients Served`, `15+ Qualified Doctors`, and `24/7 Medical Support` as fixed HTML copy; they are not queried from Supabase. Present these as placeholder/marketing copy unless separately validated.
9. **Brand/deployment identifiers are mixed.** `wrangler.jsonc` names the worker `fud-hms`; `fud-logo.png` remains in the tree; `confirm.html` sends users to a hard-coded Vercel URL (`https://atbu-medi-care.vercel.app/index.html`). This can create callback or deployment mismatch if hosting elsewhere. Verify Supabase redirect allowlists and hosting URLs before a live deployment.
10. **External dependencies require network access.** Supabase JS, Phosphor icons, Poppins, and `html2canvas` are loaded from external CDNs. Offline operation and CDN-independent deployment are not demonstrated.
11. **No automated test suite or build is included.** The project has no package manifest, test scripts, or CI configuration in this repository. Do not claim automated tests or production validation without separate evidence.

## 9. Files and Assets in This Checkout

- Application pages: `index.html`, `admin.html`, `doctor.html`, `patient.html`, `pharmacist.html`, `confirm.html`, `reset-password.html`.
- Deployment and metadata: `wrangler.jsonc`, `.gitignore`, `LICENSE`, `README.md`.
- Images: `logo.png`, `fud-logo.png`, `og-image.png`, `makama.png`, `mujahid.png`, `amina.png`, `alamin.png`, `buhari.png`, `faruq.png`.
- Not present: SQL schema/migrations, Edge Function source, `package.json`, local vendored JS/CSS dependencies, automated tests.

## 10. Suggested Presentation Plan

### 7-10 minute outline

1. **Problem and goal (45 seconds):** Explain the need for a single flow linking appointment scheduling, clinical records, prescription, payment verification, and dispensing for a university healthcare setting.
2. **Product and users (45 seconds):** Introduce ATBU MediCare and the four roles. Emphasize that each role has a separate dashboard and permissions are intended to be enforced by Supabase.
3. **Architecture (60 seconds):** Show the static HTML role pages connected to Supabase Auth/Postgres/RPC/Edge Functions, and Cloudflare static asset configuration. Clarify that the database and function implementation is a separately configured backend, not included in this repository.
4. **Patient journey (2 minutes):** Demonstrate sign-in, appointment booking wizard, prescription/medical instructions, payment submission, and receipt view. Say that payment is submitted for manual verification, not charged by the app.
5. **Clinician and operations journey (2-3 minutes):** Show doctor pending/approved appointment handling, consultation and prescription preparation; then admin verification and pharmacist dispensing/stock.
6. **Safety and operational controls (60 seconds):** Explain role checks, staff deactivation, roster capacity, low-stock alerts, payment gates, and the server-side dispensing procedure. Make clear that RLS and RPC implementation require backend validation.
7. **Close (30 seconds):** Summarize how one appointment progresses from request to clinical work to payment verification and medicine fulfillment; mention the next steps needed for a production deployment.

### Safe demo sequence

- Prepare test accounts for all four roles and one synthetic patient profile.
- Before presenting, ensure a doctor has a future shift with free capacity, pharmacy stock includes the proposed medicine, and one payment is available for verification.
- Start on the landing page, then show patient appointment booking.
- Sign in as doctor; approve the appointment and complete a consultation with a simple test prescription.
- Sign in as admin; verify the test payment.
- Sign in as pharmacist; show the prescription leaving Awaiting and appearing Ready, then dispense it and show stock/status change.
- Return to patient to show prescription/payment/receipt views.
- Avoid showing actual email addresses, temporary passwords, real health data, API keys, or destructive deletion actions.
- If accounts or backend data are not pre-seeded, use screenshots or a prepared walkthrough rather than relying on live signup, OAuth, or email delivery during the presentation.

### Suggested opening statement

> ATBU MediCare is a role-based campus healthcare workflow prototype. It brings patients, doctors, administrators, and pharmacists into one process: book a visit, provide care, issue a prescription, verify payment, and dispense medicine while tracking inventory.

### Likely questions and concise answers

**Does the app process payments online?**
No payment gateway is included in this repository. Patients submit method/reference details, and an administrator verifies the payment. The data model and UI support payment status and receipts.

**How is a prescription prevented from being dispensed before payment?**
The UI separates prescriptions into Awaiting and Ready queues based on payment status. Actual enforcement and stock deduction are delegated to `dispense_prescription`; its backend implementation and RLS policies need to be inspected separately.

**Where is the database schema?**
It is not included in this repository. The pages expect the listed Supabase tables, RPCs, triggers, policies, and Edge Functions to already exist.

**Can patients self-register as staff?**
No. Patient self-registration is implemented on the public page. Doctors and pharmacists are provisioned by an administrator through the `create-staff-user` Edge Function.

**What happens when a doctor completes a visit?**
The page saves advice and an internal medical note, marks the appointment completed, creates a prescription and its items, and requests payment creation. Those are separate requests in the current client implementation.

**What prevents a doctor or patient from reading another person's records?**
The client scopes some queries to the current user or doctor's appointment relationships, and dashboard code checks roles. The authoritative control must be Supabase RLS and backend authorization, which are not included here and must be validated.

**Is the app ready for production?**
The repository demonstrates the user-facing workflows but does not establish production readiness. Before real patient data, validate RLS, RPC atomicity, Edge Function authorization, audit logging, backups, consent/privacy requirements, deployment callbacks, and operational support.

## 11. Brief for Another AI

Use this document as the factual source for generating presentation notes. Produce speaker notes in clear, natural language for a 7-10 minute presentation, plus a short slide outline and a 60-second summary. Explain the patient-to-pharmacy journey chronologically. Do not claim that payments are processed by a gateway, that the Supabase schema/functions are included, that the displayed homepage counts are live/verified, or that the system is production-certified. Keep technical caveats understandable to a non-specialist and do not reproduce API keys, credentials, or patient data.
