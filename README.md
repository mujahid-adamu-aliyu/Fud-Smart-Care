# ATBU MediCare

A Supabase-backed healthcare management web application for Abubakar Tafawa Balewa University (ATBU), built by Team Tech Fusion for the 2026 Interdisciplinary Knowledge and Innovation Summit. The app supports patient appointments, prescriptions and payments, alongside staff, roster and pharmacy-stock workflows.

## Application Files

| File | Purpose |
|---|---|
| `index.html` | Public landing page, patient registration, sign-in and role-based redirect |
| `admin.html` | Manage doctors and pharmacists, roster shifts, appointments, payments, patients and stock overview |
| `doctor.html` | Doctor dashboard, roster, appointment queue and history, patient lookup, consultations and prescriptions |
| `patient.html` | Patient dashboard, appointment booking, prescriptions, payments and profile/medical records |
| `pharmacist.html` | Pharmacy stock management and prescription dispensing workflow |
| `confirm.html` | Supabase email confirmation flow |
| `reset-password.html` | Password recovery and password update flow |

## Supporting Files

- `logo.png`, `fud-logo.png`, `og-image.png`: brand and social-preview images. `fud-logo.png` is a legacy-named asset retained in this snapshot.
- `makama.png`, `mujahid.png`, `amina.png`, `alamin.png`, `buhari.png`, `faruq.png`: Team Tech Fusion profile images used by the landing page.
- `wrangler.jsonc`: Cloudflare Wrangler static-assets configuration; it serves the repository root as assets under the worker name `fud-hms`.
- `LICENSE`: Apache License 2.0.

## Technology

- Static HTML, CSS and JavaScript. There is no application build step or package manifest in this repository.
- Supabase Auth and Postgres, accessed by the browser using `@supabase/supabase-js`.
- Phosphor icons and Poppins font are loaded from external CDNs.
- Light and dark themes are stored in browser `localStorage` under ATBU-specific keys.

## Run Locally

Open `index.html` through a local static web server. For example, from this directory:

```sh
python3 -m http.server 8000
```

Then open <http://localhost:8000>. Authentication and data features require the configured Supabase project and network access; the UI is not a standalone offline app.

## Deploy with Cloudflare

The included Wrangler configuration serves the repository root as static assets. With Wrangler installed and authenticated, deploy from this directory using:

```sh
npx wrangler deploy
```

The repository does not include database migrations, Supabase Edge Function source, or a package manifest. Database tables, policies, RPCs, and Edge Functions used by the pages must already be configured in Supabase. Do not place Supabase service-role keys in browser code; the frontend uses the project's anon key.
