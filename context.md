# NileBridge Scholar — Project Context

> Single source of truth for the project: what we are building, why, how, and in what order.
> Keep this file updated. If code and this file disagree, fix one of them the same day.
> Anyone (human or AI assistant) starting work on the repo must read this file first.

---

## 1. Project in one paragraph

**NileBridge Scholar** is a bilingual (English/Arabic) web platform that helps international students, with a first focus on Sudanese students, discover Rwandan universities, apply to them, pay the application fee, and track their application. Admins manage universities, programs, applications, students, payments, FAQs and country information from a dashboard. Card payments (Visa/Mastercard) go through **Flutterwave** (test mode for now). Students in Sudan who prefer **Bank of Khartoum** or **Western Union** follow a guided manual payment flow, where proof of payment is **optional** in the app because many students send receipts on WhatsApp.

It is the rebuild of an earlier project called **Marina** (Next.js + Prisma + Tailwind, deployed on Vercel). The new version is a proper Java backend with a modern Next.js frontend.

**Name idea:** the Nile links Sudan and Rwanda (Nile waters rise in the Great Lakes region). *NileBridge* = a bridge between students and Rwandan education.

## 2. Goals

### 2.1 Primary goals
1. **Final year project:** a complete, demonstrable, well-documented system that satisfies academic requirements (analysis, design, implementation, testing, report).
2. **Job search:** a portfolio piece that shows I can build and ship production-style Java/Spring Boot software, to land Java engineering roles in Rwanda.
3. **Real-world usefulness:** the system should be usable by a real agency or university office if someone becomes interested later.

### 2.2 Guiding principle: simple but very effective
- Prefer boring, well-understood solutions over clever ones.
- Fewer features, done properly (validated, tested, secured, documented), beat many half-finished ones.
- Every feature must answer: *does this help a student apply, or help an admin process applications?* If not, it is out of scope.
- Cost target: **as close to $0/month as possible.**

### 2.3 What "success" looks like
- A recruiter can open a live demo link, register a student, submit an application, make a test card payment, and see an admin approve a manual payment, all within 5 minutes, without reading code.
- A reviewer can clone the repo, run `docker compose up`, and have everything working.
- The README, architecture diagram, API docs, and tests make the engineering quality obvious.

## 3. Non-goals (explicitly out of scope for v1)
- Real-money payments (Flutterwave stays in **test mode**; going live is a config change for later).
- Automated WhatsApp integration (the WhatsApp Business API); we only use `wa.me` deep links.
- Native mobile apps. The site is responsive and that is enough.
- Multi-tenancy, microservices, message brokers, Kubernetes.
- AI recommendation or chatbot features.
- Live chat. WhatsApp links cover communication.
- Languages beyond English and Arabic at launch (French is a possible later addition).

## 4. Users and roles

| Role | Who | What they can do |
|---|---|---|
| `VISITOR` | Anyone, not logged in | Browse universities, programs, country guides, FAQs; switch language |
| `STUDENT` | Registered, email-verified | Manage profile, create/submit applications, upload documents, pay, track status |
| `PAYMENT_REVIEWER` | Staff who verify manual payments | View and approve/reject manual payments only |
| `ADMIN` | Full staff | Everything: content, applications, students, payments, users, audit log |

Notes:
- Who approves manual payments is still undecided, so the reviewer role exists now and can be assigned later without code changes.
- Roles are stored on the user (single role per user is enough for v1).
- The very first admin is created from environment variables on first startup (see §14), never from hard-coded credentials.

## 5. Tech stack

| Layer | Choice | Notes |
|---|---|---|
| Backend | **Java 21 (LTS)** + **Spring Boot** (latest stable at project start; pin the version) | Web, Validation, Data JPA, Security, Mail, Actuator |
| Database | **PostgreSQL** | Schema managed only by **Flyway** migrations |
| Frontend | **Next.js (App Router) + TypeScript + Tailwind CSS** | Server components for SEO; `next-intl` for EN/AR |
| Payments | **Flutterwave** (test mode) | Hosted checkout + webhook + server-side verification |
| File storage | **Cloudflare R2** (S3-compatible) | Accessed through the AWS SDK for Java v2 behind a storage interface |
| Email | **Gmail SMTP (app password)** or **Brevo** free tier | No custom domain required; see §11 |
| Auth | **JWT** (short-lived access token + refresh token) | Stored in httpOnly cookies through the Next.js server layer (see §9) |
| API docs | **springdoc-openapi** (Swagger UI) | Available in dev and demo |
| Testing | JUnit 5, Mockito, **Testcontainers (PostgreSQL)**, Playwright (a few e2e) | |
| Build | Maven (or Gradle, pick one and stay with it) | |
| Containers | Docker + Docker Compose | One command to run everything locally |
| CI | GitHub Actions | Build + test on every push/PR |

### Library shortlist (add only when needed)
- MapStruct (optional, for DTO mapping) or hand-written mappers; Lombok only if used consistently.
- Bucket4j (rate limiting), jjwt or Spring's built-in resource server support for JWT.
- AWS SDK v2 `s3` module for R2.
- Apache Tika (or `Files.probeContentType` plus magic-byte checks) for upload validation.
- Zalando Problem or Spring's native `ProblemDetail` for error responses.

**Rule:** do not add a dependency without a clear reason written in the PR/commit message.

## 6. Architecture

### 6.1 Style
A **modular monolith**: one Spring Boot application, organized by feature (not by technical layer), plus a separate Next.js frontend. No microservices.

```
 Browser ──► Next.js (SSR, i18n, SEO) ──► Spring Boot REST API ──► PostgreSQL
                  │                              │
                  │                              ├──► Cloudflare R2 (documents)
                  │                              ├──► Flutterwave (checkout, verify)
                  │                              └──► SMTP (email codes, notifications)
                  └── Flutterwave webhook ───────► Spring Boot (public webhook endpoint)
```

### 6.2 Backend package layout (by feature)

```
com.nilebridge.scholar
├── NileBridgeApplication.java
├── common/            # base entities, exceptions, error handling, utils, pagination
├── config/            # security, CORS, OpenAPI, storage, mail, properties classes
├── auth/              # register, login, refresh, logout, email verification, password reset
├── user/              # User entity, roles, admin user management
├── student/           # StudentProfile
├── university/        # University, Program
├── application/       # Application, status workflow, status history
├── document/          # ApplicationDocument, upload/download
├── payment/
│   ├── core/          # Payment entity, status, references, shared logic
│   ├── flutterwave/   # client, webhook, verification
│   └── manual/        # manual payment flow, proofs, review queue
├── content/           # FAQ, CountryInfo (bilingual content)
├── notification/      # email sending, templates
├── audit/             # AuditLog entity, recording service
└── storage/           # FileStorageService interface + R2 and local implementations
```

Each feature package contains: `*Controller`, `*Service`, `*Repository`, `domain` (entities/enums), `dto` (request/response records).

### 6.3 Layer rules
- Controllers: HTTP only (validation, mapping, status codes). No business logic.
- Services: business rules and transactions (`@Transactional`). No HTTP types.
- Repositories: Spring Data JPA; custom queries only when needed.
- Entities never leave the service layer; the API exposes **DTO records** only.
- One feature must not reach into another feature's repository; call its service.

### 6.4 Frontend structure (Next.js)

```
frontend/
├── app/
│   └── [locale]/                    # en | ar
│       ├── (public)/                # home, universities, programs, countries, faq, how-to-apply, contact
│       ├── (auth)/                  # register, verify-email, login, forgot-password
│       ├── (student)/               # dashboard, profile, applications, payments
│       ├── (admin)/admin/           # dashboard, applications, students, universities, faq, country-info, payments, audit-log, users
│       └── api/                     # route handlers acting as a thin proxy to the Spring API
├── components/
├── lib/                             # api client, auth helpers, seo helpers, formatters
├── messages/                        # en.json, ar.json
└── public/
```

## 7. Domain model

### 7.1 Entities and relationships (summary)
- `User` 1—1 `StudentProfile` (only for students)
- `University` 1—N `Program`
- `StudentProfile` 1—N `Application`; each `Application` targets one `Program` (and therefore one `University`)
- `Application` 1—N `ApplicationDocument`
- `Application` 1—N `ApplicationStatusHistory`
- `Application` 1—N `Payment` (usually one successful payment)
- `Payment` 0..1—N `PaymentProof` (manual payments only, optional)
- `AuditLog` records sensitive actions by any user
- `Faq` and `CountryInfo`: bilingual public content, managed by admins
- `VerificationCode`: email verification and password reset codes
- `RefreshToken`: hashed refresh tokens for revocation

### 7.2 Bilingual content strategy
Keep it simple: **separate columns** (`name_en`, `name_ar`, `description_en`, `description_ar`) for content entities instead of translation tables. There are only two languages and few content types. If a third language is added later, migrate to translation tables then.

### 7.3 Table sketches (final DDL lives in Flyway migrations)

```sql
users(
  id UUID PK, email VARCHAR UNIQUE NOT NULL, password_hash VARCHAR NOT NULL,
  role VARCHAR NOT NULL,                     -- STUDENT | PAYMENT_REVIEWER | ADMIN
  email_verified BOOLEAN NOT NULL DEFAULT false,
  enabled BOOLEAN NOT NULL DEFAULT true,
  must_change_password BOOLEAN NOT NULL DEFAULT false,
  preferred_locale VARCHAR(5) NOT NULL DEFAULT 'en',
  created_at, updated_at
)

student_profiles(
  id UUID PK, user_id UUID UNIQUE FK, first_name, last_name,
  date_of_birth DATE, nationality VARCHAR, country_of_residence VARCHAR,
  phone VARCHAR, whatsapp VARCHAR, passport_number VARCHAR,
  highest_education VARCHAR, created_at, updated_at
)

universities(
  id UUID PK, slug VARCHAR UNIQUE, name_en, name_ar, city, website,
  description_en TEXT, description_ar TEXT, logo_key VARCHAR,
  published BOOLEAN, created_at, updated_at
)

programs(
  id UUID PK, university_id FK, slug VARCHAR UNIQUE, name_en, name_ar,
  level VARCHAR,                             -- DIPLOMA | BACHELOR | MASTER | PHD
  duration_months INT, tuition_amount NUMERIC(12,2), tuition_currency CHAR(3),
  language_of_instruction VARCHAR, intake_months VARCHAR,
  description_en TEXT, description_ar TEXT, published BOOLEAN, created_at, updated_at
)

applications(
  id UUID PK, reference VARCHAR UNIQUE,      -- NBS-2026-000421
  student_id FK, program_id FK, intake VARCHAR,
  status VARCHAR NOT NULL, submitted_at, decided_at,
  admin_notes TEXT, created_at, updated_at
)

application_status_history(
  id UUID PK, application_id FK, from_status, to_status,
  changed_by FK users, note TEXT, changed_at
)

application_documents(
  id UUID PK, application_id FK,
  type VARCHAR,                              -- PASSPORT | TRANSCRIPT | CERTIFICATE | PHOTO | OTHER
  original_filename VARCHAR, storage_key VARCHAR UNIQUE, content_type VARCHAR,
  size_bytes BIGINT, uploaded_at
)

payments(
  id UUID PK, reference VARCHAR UNIQUE,      -- NBP-2026-000123 (also Flutterwave tx_ref)
  application_id FK, method VARCHAR,         -- CARD_FLUTTERWAVE | BANK_OF_KHARTOUM | WESTERN_UNION
  amount NUMERIC(12,2), currency CHAR(3), status VARCHAR NOT NULL,
  provider_transaction_id VARCHAR,           -- Flutterwave transaction id
  provider_payload JSONB,                    -- last verified provider response (for audit)
  sender_name VARCHAR, transfer_reference VARCHAR,  -- manual: bank reference or WU MTCN
  student_note TEXT, reviewed_by FK users, reviewed_at, review_note TEXT,
  created_at, updated_at
)

payment_proofs(
  id UUID PK, payment_id FK, storage_key VARCHAR, original_filename, content_type,
  size_bytes, source VARCHAR,                -- UPLOADED_BY_STUDENT | ATTACHED_BY_ADMIN (e.g., received on WhatsApp)
  uploaded_by FK users, uploaded_at
)

faqs(id UUID PK, question_en, question_ar, answer_en TEXT, answer_ar TEXT,
     category VARCHAR, sort_order INT, published BOOLEAN, created_at, updated_at)

country_info(id UUID PK, slug VARCHAR UNIQUE, country_code CHAR(2),
     title_en, title_ar, body_en TEXT, body_ar TEXT,
     published BOOLEAN, created_at, updated_at)

verification_codes(id UUID PK, user_id FK, purpose VARCHAR,   -- EMAIL_VERIFY | PASSWORD_RESET
     code_hash VARCHAR, expires_at, attempts INT, used_at, created_at)

refresh_tokens(id UUID PK, user_id FK, token_hash VARCHAR, expires_at, revoked_at, created_at)

audit_logs(id UUID PK, actor_id FK NULL, actor_role VARCHAR, action VARCHAR,
     entity_type VARCHAR, entity_id VARCHAR, details JSONB, ip VARCHAR, created_at)
```

Conventions: UUID primary keys; `created_at`/`updated_at` on every table; money as `NUMERIC(12,2)` plus an ISO currency code; foreign-key indexes on every FK; unique constraints for slugs and references.

## 8. Business workflows

### 8.1 Application status workflow

```
DRAFT ──submit──► SUBMITTED ──admin picks up──► UNDER_REVIEW
                                                  │
                  ┌───────────────────────────────┼───────────────────────┐
                  ▼                               ▼                       ▼
             NEEDS_INFO ──student updates──► UNDER_REVIEW            ACCEPTED / REJECTED
                                                                          │
                                                                    (student) WITHDRAWN possible
                                                                    before decision
```

Rules:
- Only `DRAFT` and `NEEDS_INFO` applications are editable by the student.
- Submitting requires: complete profile, required documents (passport, transcript, photo), and the application fee settled (see §8.3) or a manual payment submitted, configurable via a setting.
- Every transition is validated in one place (a small state-machine class) and written to `application_status_history` and the audit log.
- Students are emailed on every status change.

### 8.2 Payment methods

| Method | Who it's for | Confirmation |
|---|---|---|
| `CARD_FLUTTERWAVE` | Anyone with Visa/Mastercard | Automatic: webhook + server-side verification |
| `BANK_OF_KHARTOUM` | Students in Sudan | Manual: admin/reviewer approves |
| `WESTERN_UNION` | Students in Sudan (and elsewhere) | Manual: admin/reviewer approves |

What the fee is for: the **application fee** per application (amount stored in configuration/settings, in USD). Whether to add tuition deposits later is an open question (§19).

### 8.3 Payment status workflow

Card (Flutterwave):
```
INITIATED ──student redirected──► PENDING ──webhook + verify OK──► SUCCESSFUL
                                      └──────failed/cancelled/expired──► FAILED
```

Manual (Bank of Khartoum / Western Union):
```
AWAITING_TRANSFER ──student says "I paid" (proof optional)──► UNDER_REVIEW ──► APPROVED
        │                                                           └──────► REJECTED (with reason)
        └──timeout/cancel──► CANCELLED
```

Key rule for manual payments: **proof upload is optional in the app.**
- The student can tick "I have paid" and optionally enter the sender name and the transfer reference or MTCN.
- The student can upload a receipt in the app, **or** tap "Send receipt on WhatsApp", which opens `https://wa.me/<number>?text=<prefilled message containing the payment reference>`.
- The reviewer can **attach proof on the student's behalf** (`source = ATTACHED_BY_ADMIN`, with a note like "received via WhatsApp") so the trail stays complete.
- A payment can be approved with no proof in the system, but the reviewer must write a verification note (for example, "Confirmed via bank statement") and the amount actually received. Both go in the audit log.

### 8.4 References
- Application: `NBS-<year>-<6-digit sequence>`, e.g. `NBS-2026-000421`.
- Payment: `NBP-<year>-<6-digit sequence>`. This also serves as the Flutterwave `tx_ref`.
- Generated from database sequences, never random guesses, and shown to the student so they can quote it on WhatsApp.

## 9. Authentication and authorization

### 9.1 Registration and email verification (no custom domain)
1. Student registers (email, password, name, preferred language).
2. API creates the user with `email_verified = false` and sends a **6-digit code** (valid 15 minutes).
3. Code stored **hashed**, max 5 attempts, resend limited (e.g., 1 per minute, 5 per hour).
4. After verification the student can log in and use the app. Unverified users cannot submit applications.

Same mechanism powers **password reset**.

### 9.2 Sessions
- Passwords hashed with BCrypt (or Argon2); minimum length and basic strength validation.
- **Access token**: JWT, short-lived (e.g., 15 minutes). **Refresh token**: opaque random value, stored hashed in DB, rotated on use, revocable.
- Next.js acts as a thin **backend-for-frontend (BFF)**: its server route handlers call the Spring API and keep the tokens in **httpOnly, Secure, SameSite=Lax cookies** on the frontend's own origin. This avoids CORS and third-party cookie problems when the frontend and API sit on different hosts, and keeps tokens out of JavaScript.
- The Spring API remains stateless and also accepts `Authorization: Bearer` so Swagger and tests work.

### 9.3 Authorization
- Role-based rules on endpoints (`@PreAuthorize` or security config).
- Ownership rules in services: a student can only read/modify **their own** profile, applications, documents and payments (check `student_id`, not just the role).
- Admin-only: content management, status changes, user management, audit log.
- Reviewer + admin: manual payment queue.

### 9.4 Hardening checklist
- Rate limit login, registration, code verification, resend code (Bucket4j, per IP and per email).
- Generic error messages on login and reset (no account enumeration).
- Validate all input with Bean Validation; reject unknown fields where sensible.
- Security headers on the frontend (CSP, X-Content-Type-Options, Referrer-Policy, frame protection).
- CORS limited to the known frontend origin(s).
- Secrets only from environment variables; `.env` is gitignored; `.env.example` is committed.
- Dependency updates checked (Dependabot enabled).

## 10. Payments in detail

### 10.1 Flutterwave (test mode)
> At implementation time, **read the current Flutterwave docs** (API version, endpoints, header names may differ from what is written here). The flow below is the intent.

1. `POST /api/v1/payments/card` (student, for one of their applications) → API creates a `Payment` (`INITIATED`) with a unique `tx_ref`, calls Flutterwave to create a hosted checkout, stores the status `PENDING`, returns the checkout link.
2. Frontend redirects the student to Flutterwave. After payment, Flutterwave redirects back to a frontend result page. **That redirect is not proof of payment.**
3. Flutterwave calls our **webhook** (`POST /api/v1/webhooks/flutterwave`, public, but protected):
   - Verify the webhook signature/secret-hash header against our configured secret.
   - Look up the payment by `tx_ref`.
   - Call Flutterwave's **verify transaction** endpoint server-side.
   - Confirm: status is successful **and** amount, currency and `tx_ref` match our record.
   - Mark `SUCCESSFUL` in a transaction; ignore if already `SUCCESSFUL` (**idempotent**, since webhooks can repeat).
   - Store the verified provider response in `provider_payload`; write an audit log; email the student.
4. The result page polls our API (`GET /payments/{id}`) so the student sees the real status.
5. Use Flutterwave **test cards** for demos; document them in the README.

Edge cases to handle: duplicate webhooks, webhook arriving before the redirect, student closing the tab, amount mismatch (flag `FAILED` + audit entry), unknown `tx_ref` (log and return 200 to avoid retries).

### 10.2 Manual payments (Bank of Khartoum / Western Union)
Configuration (not hard-coded): beneficiary name, bank account details, Western Union receiver details, WhatsApp number, instruction text in EN/AR. Store in a `settings` table or application config so admins can change them without redeploying (settings table is preferred; add it to the schema when this feature is built).

Student flow:
1. Choose **Bank of Khartoum** or **Western Union** on the payment screen.
2. System creates a payment (`AWAITING_TRANSFER`) and shows: payment reference, amount, currency, beneficiary details, step-by-step instructions (EN/AR).
3. Student pays outside the system.
4. Student clicks **"I have paid"**, optionally entering sender name, transfer reference/MTCN, a note, and optionally uploading a receipt. Or taps **"Send on WhatsApp"** (prefilled with the payment reference). → status `UNDER_REVIEW`.
5. Student gets an email when the payment is approved or rejected (with reason).

Reviewer flow:
1. **Pending manual payments** queue, oldest first, with filters (method, status, date).
2. Open a payment: see student, application, amount expected, references, any proof.
3. Optionally attach proof received on WhatsApp.
4. Approve (enter amount received + verification note) or Reject (reason required).
5. Approval marks the payment `APPROVED`, updates the application's payment state, writes the audit log, sends email.

### 10.3 Money rules
- Amount expected comes from server-side configuration, **never from the client**.
- On approval, if amount received differs from expected, require an explicit note.
- Payments are never deleted; they are cancelled or rejected with history preserved.

## 11. Email without a domain

- **Development:** Mailpit (or MailHog) in Docker Compose, which shows all emails in a local web UI.
- **Demo/production without a domain:** Gmail SMTP with an **app password**, or Brevo with a single verified sender address. Expect some emails in spam; mention this in the README.
- Provider is selected by environment variables; code uses Spring's `JavaMailSender` only.
- Templates: simple HTML + plain text, in EN and AR, chosen from the user's `preferred_locale`.
- Emails sent: verification code, password reset code, application status change, payment received/approved/rejected, application submitted confirmation.
- Sending is **asynchronous** (`@Async` or an outbox table later) so API requests are not slowed by SMTP.
- If a domain becomes available later (for example, via a student developer program), add SPF/DKIM and switch the sender; no code changes.

## 12. File storage (Cloudflare R2)

### 12.1 Abstraction
```java
public interface FileStorageService {
    StoredFile store(String folder, MultipartFile file);   // returns storage key + metadata
    InputStream load(String key);
    void delete(String key);
    String createTemporaryDownloadUrl(String key, Duration validity); // optional, for presigned URLs
}
```
Implementations: `R2FileStorageService` (AWS SDK v2, S3-compatible endpoint) and `LocalFileStorageService` (dev/tests). Selected by property `storage.provider=r2|local`.

### 12.2 Rules
- Bucket is **private**. Nothing is public.
- Keys are random: `applications/{applicationId}/{uuid}.{ext}`. Original filename lives only in the DB.
- Allowed types: PDF, JPG, PNG. Max size configurable (default 5 MB per file). Verify the **actual content** (magic bytes), not only the extension or the client-provided content type.
- Upload goes through the API (multipart) so validation and authorization happen first.
- Download: API checks ownership/role, then either streams the object or returns a **short-lived presigned URL** (e.g., 5 minutes).
- Deleting an application's documents is an admin/student action with an audit entry.
- Cost note: R2's free allowance covers the project's expected volume; egress is free.

## 13. Internationalization and SEO

### 13.1 Languages
- Locales: `en` (default) and `ar` (**RTL**). URLs: `/en/...` and `/ar/...`.
- UI strings in `messages/en.json` and `messages/ar.json` (`next-intl`).
- Content from the API is returned in the requested language through `Accept-Language` or a `?lang=` parameter, falling back to English if the Arabic field is empty.
- `<html lang dir>` set per locale; use logical CSS properties (`ms-`, `me-`, `ps-`, `pe-`) so layouts flip correctly.
- Choose an Arabic web font with good readability (e.g., Noto Sans Arabic or Cairo) and test long Arabic text in cards and tables.
- Dates, numbers and currency formatted per locale; consider Western digits for IDs and amounts to avoid confusion.

### 13.2 SEO and discoverability
- **Server-render all public pages** (server components); never hide public content behind client-only fetching.
- `generateMetadata` per page: unique title, description, canonical URL, Open Graph, Twitter cards.
- **`hreflang` alternates** between `/en/...` and `/ar/...` on every public page.
- `sitemap.ts` (all published universities, programs, countries, FAQ, both languages) and `robots.ts`.
- **JSON-LD structured data:** `CollegeOrUniversity`, `Course`, `FAQPage`, `BreadcrumbList`, `Organization`.
- Clean, readable slugs: `/en/universities/university-of-kigali`, `/en/study-in-rwanda/from-sudan`.
- An **`/llms.txt`** file summarizing the site for AI assistants, plus well-structured, factual content (clear headings, FAQ-style Q&A) that AI answer engines can quote.
- Performance: image optimization (`next/image`), small JS, good Core Web Vitals, accessible markup (headings, alt text, contrast).
- Content that earns traffic (this matters more than technical tricks): guides such as "How to apply to universities in Rwanda from Sudan", visa and cost explanations, and a clear FAQ, all in Arabic and English. Content must be accurate and kept up to date; add a "last updated" date.
- Private pages (student/admin/auth) get `noindex`.

## 14. Admin dashboard

Navigation (carried over from Marina, plus additions marked ★):
1. **Admin Dashboard**: counts (applications by status, pending manual payments, new students this week), recent activity.
2. **Applications**: list/filter/search; detail view with documents, status change, notes, history.
3. **Students**: list/search; profile and their applications.
4. **Universities**: CRUD for universities and their programs (EN/AR fields, publish toggle, logo upload).
5. **FAQ**: CRUD, ordering, categories, publish toggle.
6. **Country Info**: CRUD for country guides (EN/AR), publish toggle.
7. ★ **Payments**: manual payments queue (see §10.2) and card transaction list.
8. ★ **Audit Log**: searchable log of sensitive actions.
9. ★ **Users**: create staff users, assign `ADMIN`/`PAYMENT_REVIEWER`, disable accounts (small screen; admin only).

Bootstrap admin:
- On first startup, if no admin exists, create one from `BOOTSTRAP_ADMIN_EMAIL` and `BOOTSTRAP_ADMIN_PASSWORD`, with `must_change_password = true`.
- No default credentials in the README, ever. (The old Marina README published a default admin login; this must not repeat.)

## 15. REST API outline (`/api/v1`)

Conventions: JSON; plural nouns; pagination `?page=&size=&sort=`; errors as RFC 7807 `ProblemDetail` with field errors; ISO-8601 UTC timestamps; UUIDs in paths.

**Auth**
- `POST /auth/register`, `POST /auth/verify-email`, `POST /auth/resend-code`
- `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`
- `POST /auth/forgot-password`, `POST /auth/reset-password`
- `POST /auth/change-password`

**Public (no auth)**
- `GET /public/universities`, `GET /public/universities/{slug}`
- `GET /public/programs`, `GET /public/programs/{slug}`
- `GET /public/countries`, `GET /public/countries/{slug}`
- `GET /public/faqs`
- `GET /public/settings/payment-instructions` (non-secret instructions only)
- `GET /public/sitemap-data` (slugs + last modified for the sitemap)

**Student**
- `GET/PUT /me/profile`
- `GET/POST /me/applications`, `GET/PUT /me/applications/{id}`, `POST /me/applications/{id}/submit`, `POST /me/applications/{id}/withdraw`
- `GET/POST /me/applications/{id}/documents`, `GET /me/documents/{id}/download`, `DELETE /me/documents/{id}`
- `POST /me/applications/{id}/payments/card`, `POST /me/applications/{id}/payments/manual`
- `POST /me/payments/{id}/mark-paid` (manual; proof optional)
- `POST /me/payments/{id}/proofs` (optional upload), `GET /me/payments/{id}`, `GET /me/payments`

**Webhooks**
- `POST /webhooks/flutterwave`

**Admin / reviewer**
- `/admin/dashboard/stats`
- `/admin/applications` (list, detail, `POST .../status`, notes)
- `/admin/students`
- `/admin/universities`, `/admin/programs` (CRUD)
- `/admin/faqs`, `/admin/countries` (CRUD)
- `/admin/payments` (queue/list), `GET /admin/payments/{id}`, `POST .../approve`, `POST .../reject`, `POST .../proofs` (attach on behalf)
- `/admin/audit-logs`
- `/admin/users` (create staff, change role, disable)
- `/admin/settings` (payment instructions, fee amount, WhatsApp number)

Documentation: every endpoint appears in Swagger with examples; secured endpoints show required roles.

## 16. Quality: testing, CI, DevOps

### 16.1 Testing strategy (keep it meaningful, not huge)
- **Unit tests** (JUnit + Mockito): application state machine, payment approval rules, reference generation, code verification logic, upload validation.
- **Integration tests** (Spring Boot Test + **Testcontainers PostgreSQL**): auth flow, ownership checks (student A cannot read student B's data), payment webhook idempotency, manual payment approval, Flyway migrations apply cleanly.
- **Flutterwave** is mocked in tests (WireMock or a fake client).
- **E2E** (Playwright, 2-3 flows): register → verify → apply → pay; admin approves manual payment; language switch to Arabic renders RTL.
- Target: solid coverage on services and security rules; do not chase a percentage for its own sake. Show a coverage badge only if it is honestly high.

### 16.2 CI (GitHub Actions)
- Backend: build, run tests (Testcontainers works on GitHub runners), report.
- Frontend: lint, type-check, build.
- Optional: Docker image build on `main`.

### 16.3 Local development
```
docker compose up        # postgres, mailpit, api, frontend (and optionally minio for S3-compatible local storage)
```
- `.env.example` documents every variable.
- Seed data (demo universities, programs, FAQs, country info) loads only with the `dev`/`demo` profile.
- Spring profiles: `dev`, `test`, `demo`, `prod`.

### 16.4 Environment variables (names; values never committed)
```
# Database
DB_URL, DB_USERNAME, DB_PASSWORD
# Security
JWT_SECRET, JWT_ACCESS_TTL_MINUTES, JWT_REFRESH_TTL_DAYS
BOOTSTRAP_ADMIN_EMAIL, BOOTSTRAP_ADMIN_PASSWORD
# Flutterwave (test keys)
FLW_PUBLIC_KEY, FLW_SECRET_KEY, FLW_WEBHOOK_SECRET_HASH
# Storage
STORAGE_PROVIDER=r2|local
R2_ACCOUNT_ID, R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY, R2_BUCKET, R2_ENDPOINT
MAX_UPLOAD_MB
# Mail
MAIL_HOST, MAIL_PORT, MAIL_USERNAME, MAIL_PASSWORD, MAIL_FROM
# App
APP_FRONTEND_URL, APP_API_URL, APP_WHATSAPP_NUMBER, APPLICATION_FEE_USD
```

### 16.5 Hosting (decide later; keep it near $0)
- Frontend: Vercel free tier is the easy option.
- Backend + PostgreSQL: a small VPS with Docker Compose, or a free/cheap container host with a managed Postgres. Check free-tier limits (sleeping services, database expiry) before committing, because a sleeping or expired demo hurts the portfolio goal.
- Files: Cloudflare R2.
- Health check: Spring Actuator `/actuator/health` exposed safely.

### 16.6 Observability (light)
- Structured logs with request IDs; never log passwords, tokens, document contents or full payment payloads.
- Actuator health + basic metrics. No heavy monitoring stack.

## 17. Code conventions

- **Java:** records for DTOs; constructor injection; no field injection; meaningful exceptions mapped to `ProblemDetail`; no business logic in controllers; no `@Autowired` on fields; small methods; Javadoc only where it adds meaning.
- **Naming:** packages lowercase; classes `PascalCase`; DB `snake_case`; REST paths `kebab-case`.
- **Migrations:** `V1__init.sql`, `V2__...sql`; never edit an applied migration, always add a new one.
- **Git:** branches `feature/...`, `fix/...`; **Conventional Commits** (`feat:`, `fix:`, `docs:`, `test:`, `chore:`); small, focused commits; PRs even when working alone (the history is part of the portfolio).
- **Frontend:** TypeScript strict mode; server components by default, client components only when needed; no `any`; components small and typed; accessibility checked.
- **Formatting:** automated (Spotless or Checkstyle for Java; Prettier + ESLint for the frontend).
- **Docs:** update `README.md`, this file and the OpenAPI annotations with each feature.

## 18. Roadmap (phases with exit criteria)

### Phase 0, Setup (≈ 2-3 days)
Repo structure (`backend/`, `frontend/`, `docs/`), Docker Compose (Postgres + Mailpit), Flyway baseline, CI skeleton, README skeleton, this `context.md`.
**Exit:** `docker compose up` works; CI is green on an empty test.

### Phase 1, Auth and users
Register, email code verification, login, refresh/logout, password reset, roles, bootstrap admin, rate limiting, security tests.
**Exit:** a student can register, verify (code visible in Mailpit), log in; ownership tests pass.

### Phase 2, Public content and admin CRUD
Universities, programs, FAQ, country info (EN/AR), admin screens, public pages (SSR), seed data.
**Exit:** public site lists real content in both languages; admin can manage everything.

### Phase 3, Applications and documents
Student profile, application wizard, document upload to R2 (and local), status workflow with history, admin review screens, emails.
**Exit:** full application lifecycle works end to end; file access rules tested.

### Phase 4, Payments
Flutterwave test-mode flow with webhook and verification; manual payment flow with optional proof, WhatsApp deep link, reviewer queue, approval/rejection, audit log, emails.
**Exit:** both payment paths demo cleanly; idempotency and amount-mismatch tests pass.

### Phase 5, SEO, polish and hardening
Metadata, hreflang, sitemap, JSON-LD, `llms.txt`, Arabic QA, accessibility pass, performance pass, error pages, empty states, security headers.
**Exit:** Lighthouse scores strong on key public pages; Arabic layout reviewed page by page.

### Phase 6, Delivery
Deploy demo, record a 2-3 minute walkthrough video, finish README (screenshots, architecture diagram, test cards, how to run), write the final-year report, prepare a presentation, tidy repo and commit history.
**Exit:** live link, README, report, and video all ready.

## 19. Open questions (resolve and record answers here)

1. **What exactly is paid?** Assumption: a fixed application fee per application. Confirm amount and currency, and whether tuition deposits are in scope.
2. **Who reviews manual payments** after launch (still undecided)? The reviewer role covers it technically.
3. **Which universities/programs** will be seeded with real data, and where does the content come from? Use only information that can be verified; mark "last updated".
4. **Hosting** choice for API + database (see §16.5).
5. **Required documents** per program, or the same list for all? Default: passport, transcript, photo.
6. **Student data retention and privacy:** how long documents are kept; a short privacy policy and consent checkbox are needed because passports and transcripts are sensitive.
7. **Academic requirements** from INES (report structure, diagrams such as use case, ERD, sequence diagrams) that this project must also produce.
8. **WhatsApp number** to use for the demo (a dedicated number is better than a personal one).
9. **Real-money future:** if a real client appears, the checklist for going live with Flutterwave (business verification, live keys, legal terms) needs to be added.

## 20. Portfolio and job-search checklist

What a Rwandan Java employer should see in this repo:
- [ ] Clear README: what it is, screenshots/GIFs, architecture diagram, tech stack, how to run in 1 command, test cards, live demo link.
- [ ] Clean feature-based Spring Boot structure, DTOs, validation, global error handling.
- [ ] Spring Security with JWT, roles and **ownership checks**, with tests proving them.
- [ ] Flyway migrations and a sensible relational model (ERD in `docs/`).
- [ ] Testcontainers integration tests and a green CI badge.
- [ ] Payment integration done right: webhook signature check, server-side verification, idempotency.
- [ ] Storage abstraction with an S3-compatible implementation.
- [ ] Docker Compose setup and environment documentation.
- [ ] OpenAPI/Swagger docs.
- [ ] Audit logging and a thoughtful manual-payment workflow, which shows real-world business thinking.
- [ ] Bilingual/RTL UI and SEO implementation.
- [ ] Commit history with Conventional Commits and a few merged PRs.
- [ ] A short "Design decisions" doc (`docs/decisions.md`): why a modular monolith, why BFF cookies, why optional proof, why R2.

CV bullets to aim for (write them from what is actually built):
- Built a bilingual (EN/AR) international-student admissions platform with Java 21, Spring Boot, PostgreSQL and Next.js.
- Integrated Flutterwave card payments with signed webhooks, server-side verification and idempotent processing; designed a manual-payment review workflow with a full audit trail.
- Implemented JWT authentication with role- and ownership-based authorization, rate limiting and private document storage on Cloudflare R2.
- Set up CI with GitHub Actions and Testcontainers-based integration tests; containerized with Docker Compose.

## 21. Rules for AI assistants working on this repo

1. Read this file before making changes; follow its structure and conventions.
2. Keep it **simple**. Do not introduce new frameworks, message queues, caches or microservices without asking.
3. Do not add dependencies silently; justify each one.
4. Never hard-code secrets, default passwords, or real personal data. Use environment variables and `.env.example`.
5. Every new endpoint needs: validation, authorization (role **and** ownership), a test, and an OpenAPI description.
6. Every schema change is a **new Flyway migration**; never edit applied ones.
7. Never trust the client for amounts, statuses, roles or ownership.
8. Payment status changes happen only through the defined workflows and always write an audit log.
9. Keep Arabic/RTL in mind for every UI change; add strings to both `en.json` and `ar.json`.
10. When unsure about a business rule, check §19 and ask rather than guessing; record the answer here.
11. Update this file when a decision changes.

## 22. Glossary

- **BFF:** backend-for-frontend; here, Next.js server handlers that call the Spring API and manage cookies.
- **BoK:** Bank of Khartoum.
- **WU / MTCN:** Western Union / Money Transfer Control Number, the reference for a WU transfer.
- **tx_ref:** the unique transaction reference we send to Flutterwave (our payment reference).
- **Idempotent:** processing the same event twice has the same result as processing it once.
- **RTL:** right-to-left layout (Arabic).
- **Presigned URL:** a temporary, signed link that allows downloading one private object.
- **Modular monolith:** one deployable application organized into well-separated feature modules.
