# Industrial Workforce OS — architecture and implementation plan

## Product direction
A skills-first recruitment workspace for Malaysia's industrial workforce. The primary unit is a ready-to-work profile: skills, location, shift, availability and salary expectations. Employers move from an explained match to a shortlist to an interview without switching tools. Keep basic access free; charge employers for useful automation, team controls and reporting. Avoid paid worker visibility and opaque candidate ranking.

## Increment 1 — delivered interactive prototype
Responsive employer dashboard, local worker registration and employer profile, job posting with salary ranges, deterministic matching, shortlist toggles, local interview schedule, WhatsApp message preparation, worker discoverability consent, local export/deletion, and an admin preview. Seed data is fictional. Browser storage persists only on the current device. Role selector demonstrates screens; it does not authenticate or enforce access. WhatsApp opens a draft; it never auto-sends. Interviews do not send invitations. No application submission, billing, account verification, real-time collaboration or production moderation is connected.

## Production architecture
Use a TypeScript responsive web/PWA client, a small HTTP API on Cloudflare Workers, and managed PostgreSQL with Supabase Auth and row-level security. Use object storage only when documents become necessary; the first production release should avoid identity documents and CV uploads. Serve static assets from a CDN. Use database migrations and a separate staging environment. Keep service credentials exclusively in server secrets. This is a proposed stack, not a provisioned backend. Validate vendor quotas and pricing before purchase; no monthly price is assumed here.

Browser -> CDN/PWA -> authenticated API -> PostgreSQL (RLS)
                                -> job queue/outbox -> approved notification provider
                                -> audit events and redacted operational metrics

Accounts use verified email links initially. Add phone OTP only after a delivery and abuse-cost assessment. Workers own their profiles; employers join organizations through invitations. Employer access and admin permissions must be verified server-side for every request. Client role switches carry no authority. Use HttpOnly, Secure, SameSite session cookies with CSRF protection if adopting a backend-for-frontend; otherwise validate JWTs at the API boundary and never place privileged keys in the client. Require MFA for admins.

## Data model
- users: auth identity, locale, status; no client-editable privileged role.
- worker_profiles: user_id, display_name, approximate location, shift preferences, availability, visibility, created_at.
- skills and worker_skills: normalized skill IDs, experience, self-declared/verified distinction.
- organizations and memberships: company, verification status, user_id, role (owner/recruiter/viewer).
- jobs and job_skills: organization, country, location, salary_min/max, currency, shift, openings, status, required skills.
- applications: worker, job, status and history; unique worker/job pair.
- shortlists: organization/job/worker/recruiter; unique organization/job/worker.
- interviews: application, starts_at in UTC, IANA time zone, mode, location, status, idempotency key.
- consent_events: purpose, policy version, grant/withdraw timestamp, actor.
- notification_outbox: recipient reference, purpose, channel, status, retries, deduplication key.
- audit_events: actor, organization, operation, record ID, timestamp; exclude message bodies and unnecessary personal details.
- subscriptions and entitlements: organization, provider reference, plan, verified lifecycle events.

Use foreign keys, salary and time constraints, unique indexes and transactions. Tenant-scoped indexes support organization/status/time queries. Never trust organization IDs or subscription claims supplied by a browser.

## Authorization and privacy
Workers can edit/export/delete their own profile and manage discoverability. Employers can publish jobs and manage their organization's applications, shortlists and interviews. Return a limited discovery profile only when visibility and employer eligibility permit it. Contact details require an explicit purpose/consent policy; avoid phone enumeration and exposing phone numbers in search responses. Recruiters cannot access other tenants' notes. Admin moderation uses a separate guarded surface with immutable auditing and least privilege. Define retention and deletion handling, including backups and necessary record retention, before launch. Obtain Malaysian privacy/legal review for notices, processors, transfers and retention; this document is not a compliance certification.

Security baseline: schema validation and output escaping; parameterized SQL; RLS enabled and tested on every exposed table; request limits; signup/OTP abuse controls; CSP and security headers; encrypted transport and managed encryption at rest; dependency scanning; redacted logs; backup restoration drills; secrets rotation; file type/size scanning if uploads are introduced. Admin actions and subscription webhooks require strong verification and replay protection. Do not infer nationality, ethnicity or protected characteristics for matching.

## Matching v1
Explainable score = required skill overlap (70%) + compatible shift (20%) + exact location (10%). Production should use normalized skills and location radius/worker travel preferences, separate hard requirements from preferences, show missing skills, and let workers correct their profiles. Scores do not evaluate competence or guarantee suitability. No LLM or paid AI API is required. Add learned ranking only with consent, enough outcomes, bias monitoring and a measurable benefit.

## Incremental delivery and acceptance
1. Prototype: review the employer and worker flows with 5–10 workers and 3–5 SMEs. Validate registration completion, time to shortlist, and clarity of the score. Delivered here; usability research remains.
2. Secure foundation: provision staging, migrations, Auth, organizations, roles, RLS and privacy events. Acceptance: anonymous/worker/cross-tenant requests cannot retrieve or mutate protected records; privileged roles cannot be self-assigned; backup restore succeeds.
3. Recruitment workflow: durable profiles/jobs/applications, normalized skills, server matching, transactional shortlist/interview endpoints. Acceptance: two users see consistent state; duplicate applications and conflicting interview updates are handled; worker withdrawal removes discovery access.
4. Communications: consent checks, calendar export/invitations, WhatsApp click-to-chat; later approved WhatsApp Business templates and opt-out-aware queueing. Acceptance: time zones round-trip, retries cannot duplicate sends, no message is sent without authorization.
5. Admin and monetization: company verification, report/block/moderation workflow, audit trail, free limits and paid automation entitlements. Acceptance: signed webhooks and server entitlements are tested; free basic recruiting remains usable; no real charge in staging.
6. Pilot and launch: accessibility/mobile QA, security review, rate-limit/load tests, incident procedures, deletion/export validation, monitoring and support. Production launch is gated on the preceding controls, not appearance.

## Malaysia first, ASEAN ready
Store UTC timestamps and render Asia/Kuala_Lumpur initially. Design language dictionaries for English and Bahasa Melayu; this prototype is English only. Use ISO country/currency codes and localized salary periods. Model work authorization as worker-provided structured information only when relevant. Future expansion needs country-specific privacy, recruitment and employment review, local languages and phone/address formats. Do not assume one policy applies across ASEAN.

## Commercial approach
Free: worker profile, job publishing, explained matching, manual shortlist and interview planning. Paid employer automation: collaborative pipelines, approved bulk outreach, reminders, exports and reporting. Validate willingness to pay before setting prices. Track employer activation, qualified shortlist rate, interview attendance, time to fill, worker withdrawal and complaint rates. Avoid optimizing only for applicant volume.

## Running the prototype
Serve dist/ with a local static server. No npm packages or API keys are required. Do not enter real personal data: this is a local-data prototype with no secure account boundary. The hosted Site starts owner-private. A production multi-user platform requires increments 2–6.
