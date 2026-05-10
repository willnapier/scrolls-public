---
title: PracticeForge
---

# PracticeForge

*Vendor-neutral module. Last updated: 2026-05-10.*

---

## What It Is

PracticeForge is two things built as one system:

1. **A clinical AI note-writing system** that generates defensible, practitioner-voiced session notes and clinical letters for therapy practitioners. It transforms post-session observations (two to three sentences of shorthand) into structured, evidence-based clinical notes that hold up under professional and regulatory scrutiny.

2. **An open-source practice management system (PMS)** designed to replace the practice's incumbent commercial PMS — currently costing the host practice in the low five figures per year. PracticeForge handles scheduling, client registry, search, SMS reminders, billing, document management, and a web dashboard for practitioners and administrators.

The system is being built by a Chartered Counselling Psychologist (BPS), initially for their own practice, with a clear path to deployment across approximately twenty colleagues at a collective London private practice operating across multiple therapeutic modalities.

The open-source plan: the PMS layer (scheduling, search, SMS, client database, admin dashboard) will be released under AGPL. The AI layer (voice profile wizard, faithfulness filter, prompt engineering) stays proprietary. Other practices adopt PracticeForge for the PMS; the AI layer is a paid add-on.

---

## The Core Workflow

1. Practitioner writes a brief observation after a session (bullet points, shorthand)
2. System generates a draft note via a frontier LLM (currently Anthropic Claude), prompted with the practitioner's clinical voice configuration and therapeutic framework
3. A multi-level faithfulness filter checks the draft against the source material before it reaches the practitioner — hard-blocking confabulated specifics, flagging uncertain sentences
4. Practitioner reviews, edits where needed, approves
5. Approved notes accumulate into clinical update letters (typically every four to six sessions, or for insurer re-authorisation)
6. Letters are delivered securely — either email (SMTP via lettre) or a recipient-facing portal with OTP authentication, view audit trail, and immediate revocation
7. PDFs file automatically to the practice management system for the permanent clinical record

---

## Architecture

### Tier Separation — Practitioner-Internal vs Public-Facing (decided 2026-05-10)

PracticeForge has a hard tier boundary between surfaces facing practitioners-on-LAN and surfaces facing the public internet. The dashboard never crosses it.

| Tier | Surface | Threat model | What it serves |
|---|---|---|---|
| **A — Practitioner-internal** | `practiceforge` dashboard (Leptos + Axum, localhost-bound on 127.0.0.1) | Trusted-but-fallible insiders, physical-access scenarios | Clinic workflow, notes, scheduling, billing UI, calendar, search, settings — the daily-use surface for ~20 practitioners + 2 admins |
| **B — Public-facing** | `clinical-portal` (separate Fly.io crate) | Full public-internet — automated scanners, credential-stuffing, hostile actors. Auth-grade hardening required. | Secure PDF delivery to **all recipient types**: referrers (current scope), clients, other collaborating professionals (planned scope) |
| **C — Marketing** | The practice website | Public, no PHI, low risk | Practice info, contact (maintained independently of PracticeForge) |

**The rule:** new public-facing flows go to `clinical-portal` as a separate crate or as a new sibling Fly.io app. They do *not* get folded into the `practiceforge` dashboard. Dashboard code lives behind a localhost bind forever.

### Explicitly Rejected Public-Facing Features (2026-05-10)

These were considered and rejected — do not build without explicitly revisiting this decision:

1. **Client self-booking.** Coordination risk (clients booking outside agreed protocol), contractual risk (terminated/aggrieved clients using soon-to-expire access for one last appointment), new public attack surface. **Replacement:** email-based scheduling assistance — client emails a request, the dashboard parses it and suggests slots from calendar availability, drafts a reply, the practitioner reviews/edits/sends, the booking lands in the calendar atomically on send. Stays Tier A. Human-in-loop on every booking.

2. **Collaborator-direct upload (referring doctors uploading PDFs to PracticeForge).** The status-quo email transport (password-protected PDF) is already trusted and regulated-friendly. **Replacement:** mailcurator-driven correspondence automation — auto-decrypt password-protected PDFs from known referrers, auto-classify (referral / GP letter / assessment), land in the correct client folder, notify the practitioner. No new auth model for collaborators. No new public-facing service.

3. **Insurer funding-status check by clients.** Same logic as self-booking, with the additional concern that funding-status adds *financial* PHI to the existing clinical PHI surface. Marginal benefit doesn't justify the addition.

### Inference (current state — 2026-05-07 onward)

**Frontier LLM per practitioner.** Each practitioner configures their own AI backend at install time:

- **Anthropic API** (default) — `claude-opus-4-7` for clinical drafting, with `sonnet` and `haiku` selectable
- **Claude CLI subscription** — for practitioners on Anthropic's Max plan who prefer subscription billing

**Per-practitioner Anthropic commercial DPA** (settled 2026-05-06). Each practitioner has their own Anthropic API key with auto-incorporated DPA covering session notes, letters, and private practice. The practice itself is **not** a party to AI drafting — its role attaches at the finalised-letter filing boundary. A practice-level DPA would only be needed if practice-level AI is added (not currently planned).

The previous Gemma-LoRA-on-RunPod-via-Ollama stack (Trooper.ai, Vast.ai, Vertex AI) was **removed 2026-05-07**. Local inference may be revisited in approximately 12 months when laptop-class local inference matures, but it will be a fresh decision (probably MLX or vLLM, not Ollama).

### PHI Safety — Type-Enforced

Every prompt to the LLM routes through a `PseudonymMap::apply` step on every PHI source; AI output is reverted before display, save, or log. The LLM client API takes a `Dephid` newtype so raw client text cannot reach the API at the type level — forgetting to dephi becomes a compile error. This closes the gap that earlier audits surfaced (a CLI path was leaking PHI before the May 2026 audit; the type-enforcement refactor durably prevents that class of failure).

### Voice Profile (frontier-era, replaces LoRA)

Frontier closed-weight LLMs cannot be LoRA-fine-tuned. The voice-via-LoRA pipeline that was active in early April 2026 is therefore retired.

Practitioner voice now comes from three sources, configured via a first-run wizard:

1. **Exemplars** — a handful of approved notes the practitioner provides as in-context examples (style demonstration)
2. **Style description** — a prose description of register, sentence rhythm, characteristic phrasing, what to avoid
3. **Optional RAG retrieval** — the prompt can pull a small number of relevant prior notes for the same client to anchor the new note in the practitioner's accumulated style on that client

The wizard is private and per-practitioner. No group deliberation; no central style template; individual informed consent on every wizard choice.

### Single-Binary Consolidation (2026-05-06)

The earlier multi-binary topology (`practiceforge` + `clinical` + `clinical-portal`) was consolidated. The `clinical-notes` Cargo crate folded into `practiceforge`; what was `clinical X` is now `practiceforge X`. The workspace is three crates: `practiceforge`, `clinical-core`, `clinical-portal`. Distribution is now two binaries (`practiceforge` and a transitional `tm3-download`), reducing to one once the in-tree PMS replaces the legacy PMS.

### File Structure (per client)

```
~/Clinical/clients/<id>/
├── identity.yaml      # Name, DOB, contact, PMS ID, funding, referrer
├── notes.md           # All session notes (### date headers)
├── summary.md         # AI-generated client summary (for context)
├── correspondence/    # Imported referral letters, reports
└── admin/             # Attendance, billing
```

Real names throughout, no de-identification at rest. Dephi happens only at the LLM boundary, not in storage.

### Configuration

Single config file: `~/.config/practiceforge/config.toml`

```toml
[ai]
backend = "anthropic"        # or "claude-cli"
model   = "claude-opus-4-7"

[paths]
clinical_root = "~/Clinical"
```

Plus a separate `secrets.toml` (chmod 600) for the API key.

---

## Tenancy Posture — Release Plan

Two design axes need active attention as the product matures, navigated separately:

**Axis 1 — Self-containedness for colleague deployment.** A new colleague should be able to install the binary, run the setup wizard, and end up with a working clinical workflow without manual config editing. This is the *deployability* axis.

**Axis 2 — Genericity beyond the host practice.** Naming, defaults, and assumptions baked in as host-practice defaults that would need to become configurable in a generic edition (PMS name conventions, identity labels, six-session letter cadence, reschedule policy details).

**Release ordering (2026-04-29 decision):** ship a **host-practice edition first**. Closing axis 1 (deployability) for the host case is the near-term priority. Axis 2 (genericity) is deferred. Real colleague deployments will surface deployability gaps faster than imagined ones; naming/defaults are mechanically substitutable later.

**Operational rule:** prefer "host-practice-default with override hook" over "hardcoded host-practice" where the override is cheap (`--account` flags, env-var fallbacks, config keys with sensible defaults). Don't go further into per-tenant abstraction yet.

---

## The Faithfulness Filter (post-Ollama)

A multi-level faithfulness pipeline catches confabulation before the practitioner sees a draft.

### Layer 1: String Match (Hard Gate)

Catches fabricated direct quotes — phrases in quotation marks that don't appear in the observation or client file. A hard failure triggers automatic regeneration of the entire note.

### Layer 2: NLP Structural (Soft Flag)

Stemming (`rust-stemmers`), entity extraction, and word overlap scoring. Flags sentences where the model's output has low lexical overlap with the source material. These appear as inline warnings for the practitioner to review.

This is a starter implementation designed for swappability. An NLP researcher in cognitive science independently validated the Embed-Annotate approach as the right direction and raised the granularity question: sentence-level may hide confabulation in compound sentences; clause-level splitting is the middle ground.

### Layer 3 (Embeddings): Currently Inactive

Layer 3 was Ollama-based sentence embeddings. With the Ollama footprint deletion (2026-05-07), Layer 3 is dormant. A frontier-API equivalent (Anthropic embeddings if/when offered, or a local lightweight embedder) is a future addition. The system continues to operate with Layers 1 and 2 — the original cost finding still holds: confabulation prevention via prompt is cheaper than detection after.

### Faithfulness Strategy Registry

Six named strategies are documented (Prompt-Rail, Gate-Regen, Embed-Annotate, NLP-Prefilter, Karpathy-Loop, Embed-Stream), with seven binary assertions (F1–F7) and five confabulation-trap scenarios. An eval harness scores F1–F7 mechanically against a test client.

### Anti-Confab Policy (architectural)

Safety-critical filtering lives in **prompts and deterministic gates**, never in model weights. This means model swaps (Opus → Sonnet → Haiku → some future frontier model) don't require redoing the safety work. The same gates run against any backend.

---

## Prompt Engineering and Quality Gates

### System Prompt Architecture

The model receives a structured prompt containing the complete client file (all previous sessions, formulation, therapeutic arc), session metadata (date, session number, authorisation status), an instruction block specifying clinical documentation standards, and the practitioner's observation.

**Client summaries:** for clients with long histories, a compressed clinical summary replaces the full file in-prompt — using the summary plus the last three sessions instead of every session reduces generation time substantially. Summaries auto-regenerate in the background after every `note-save`.

**PROMPT-ANCHOR split:** the instruction block lives in two files:

- `modality-act.md` — practitioner-owned therapeutic framework (ACT terms, core stance, note style). A CBT colleague drops in `modality-cbt.md`; the framework-specific instructions swap.
- `faithfulness-prompt.md` — universal grounding rules (no confabulated specifics, source material only). This is the target for the Karpathy-style self-optimising loop.

This replaced an earlier monolithic 28KB reference document with a 1KB prompt anchor. Voice via wizard exemplars; framework via prompt; safety via deterministic gates. Each lever swappable independently.

### Prompt Refinements

Based on systematic comparison of model outputs against clinical defensibility criteria:

- **Consent and collaboration** — instructions to show that in-session experiments were agreed with the client. Redundancy of "collaborative + agreed" is banned (either word implies the other).
- **Tentative framing** — interpretive links to developmental history use hedged language while anchoring to the existing formulation. The model hypothesises rather than asserts.
- **Faithfulness to source** — every specific detail must come from the observation or client file. The model must not invent plausible-sounding specifics.
- **First name usage** — the client's first name throughout rather than "the client."
- **Risk assessment** — a brief default ("No immediate concerns noted") unless the observation specifically describes risk factors.

### Deterministic Quality Gates

Post-generation validation runs before the note is presented:

- **Structural warnings (soft):** missing session header, missing Risk or Formulation lines, LLM refusal patterns
- **Lint gates (hard — trigger automatic regeneration, up to three attempts):** sentences containing both "collaborative" and "agreed"

### Evaluation Framework

A structured eval system (`skill-eval`) tests note quality against defined assertions across two layers: mechanical (tool choice, file access, safety rules — pattern matching, no LLM) and quality (clinical accuracy, framework adherence, defensibility — evaluator LLM). Faithfulness to source is a categorical failure if violated.

---

## Observation / Interpretation Separation (proposed structural change)

A proposed note shape has two distinct sections:

- **Observation** — tightly gated. Observable, functional, reporting register. What the client said, did, presented. Faithfulness Layers 1 and 2 are aggressive here.
- **Interpretation** — framework-coloured (ACT / CBT / psychodynamic / etc.), anchored to specific items in Observation, labelled as inference rather than fact.

This structure resolves both the anti-interpretive-register preference (no mentalist "therapist sees what is hidden" claims slipping into the observable record) and the theory-neutrality question (different modality colleagues get appropriately framework-coloured Interpretation, while Observation stays modality-agnostic). Estimated as approximately 3–4 dev-days extending the existing GatingPolicy scaffolding.

---

## PracticeForge PMS — The Incumbent Replacement

PracticeForge replaces the practice's commercial PMS as a parallel development track. Not gated on colleague adoption — development runs ahead.

### What the Incumbent Provides (and What PracticeForge Replaces)

| Incumbent feature | PracticeForge replacement |
|---|---|
| Appointment diary/calendar | Scheduling module (ICS-based, RRULE recurrence) |
| Client database | Central client registry (git-backed YAML) |
| Document upload/download | Headless-Chrome adapters during transition; native after cutover |
| SMS reminders | SMS module (Twilio REST API) |
| Client search | Tantivy full-text search (BM25 scoring) |
| Reporting | Billing module + attendance reports |

### Module Status (April–May 2026)

| Module | Key capabilities |
|---|---|
| Central client registry | Git-backed canonical database. Client CRUD, practitioner assignments, import (bulk + single), letter push, sync with remote (pull/push/conflict detection), auto-sync timer. |
| Tantivy search | Multi-field search across notes, correspondence, identity, diagnosis. Auto-rebuilds stale indexes. Highlighted snippets. |
| Calendar / scheduling | ICS (RFC 5545) storage. Recurring series (RRULE), one-off appointments, cancel (single via EXDATE or entire series), move, status (arrived/completed/no-show/late-cancel). ICS export endpoint for calendar-app subscription. |
| SMS reminders (Twilio) | Preview/send/status/test. JSONL delivery logging. Configurable templates. |
| Admin dashboard (Axum + Leptos) | Web server on localhost:3457. Multi-practitioner calendar, client CRUD, search, billing overview. **Migrating from vanilla HTML/JS to Leptos (Rust → WebAssembly), 6 phased sprints — Sprint 0 merged 2026-05-09; Sprints 1–3 in flight.** |
| Incumbent export/migration | Client export, diary archive conversion, document download orchestration, validation with field normalisation. |
| Billing (Manual) | Phase 1 — file-based JSON. Vendor-neutral trait architecture. |
| Billing (Xero) | Phase 2 — OAuth2 PKCE, invoice CRUD. |
| Billing (Stripe) | Phase 3 — Payment Links for online client payments. |

### Centralised Client Registry Architecture

The "no shared state" architecture holds for clinical content (session notes stay local) but breaks down for client identity and practice administration. If a client changes phone number, all practitioners need the update. The practice administrator needs a single authoritative view for regulatory compliance.

**Solution:** a bare git repository on the practice server as the canonical client registry. Plain text files (YAML, markdown) diff and merge cleanly. Full audit trail via commits. Offline-capable.

**Centralised:** client identity (name, DOB, contact, funding, referrer, status), practitioner assignments, appointment schedule, clinic update letters (mandatory — defensibility records), inbound correspondence.

**Stays local:** session notes (by default — optional encrypted mirror via config flag), drafts, observations, voice profile config.

**Sync model:** automatic pull on dashboard startup plus periodic via Tailscale. Push is event-driven (on letter send, note save, attendance report). Auto-merge for non-overlapping changes; flag for manual resolution on same-field conflicts.

**MVDDA (Minimum Viable Defensible Documentation Architecture):** deterministic defensibility applies to all notes. Session notes are practitioner-private; letters are practice-owned floor. AI is a pluggable layer above this floor — the floor stands without AI.

### Scheduling System

**Per-practitioner availability model:** `availability.yaml` defines per-day hours, breaks, modality (remote/in-person), reschedule eligibility, soft stops, travel-gap protection.

**Reschedule workflow:** CLI, dashboard endpoints, and UI (modal slot picker on DNA/cancelled appointments). Availability calculator with preference cascade scoring: minimise day span, prefer mid-week, contiguous packing, soft-stop penalty. The `--send` flag emails slot offers to clients.

**Self-booking is explicitly rejected** (see Tier Separation). Replacement is email-based scheduling assistance with human-in-loop confirmation.

### Billing Module

**Phase 1 (built):** vendor-neutral trait architecture (`AccountingProvider`, `PaymentProvider`). Manual backend with file-based JSON. Invoice generation from session data — reads identity for rate and funding, notes for session dates, auto-detects uninvoiced sessions. Self-pay (invoice to client) and insurer-funded (invoice to insurer). Batch invoicing.

**Reminder system with four tone presets:** sensitive, tentative, businesslike, assertive. Configurable escalation per practitioner. Insurer reminders always businesslike. Templates are deterministic — no LLM needed.

**Reminders send via email** (lettre SMTP). Reminder log tracks sent reminders in JSON. Dashboard: conditionally visible "Billing" tab. Invoice table with Paid/Cancel per row, batch button, pending reminders panel.

**Modular extension** supports third-party-payer arrangements (firm-bills-for-partners, corporate wellness, university schemes) as config + trait impl, not architecture work.

**DNA/CSN policy:** original slot is always billed; reschedule is free; triage for waiver only.

### Incumbent-PMS Retirement Direction (2026-05-09)

Five architectural decisions banked:

1. **Hybrid hosting:** on-premises practice server primary; cloud backup secondary
2. **Per-practitioner OAuth** for any cloud surfaces (no shared service accounts)
3. **Filesystem + rsync** for primary state; git for YAML config and audit-relevant text
4. **All-three migration paths supported** (cold cutover, shadow mode, side-by-side) — practices choose
5. **v1.5 ICS** with a `phi_level` token flag for tier-aware export

The incumbent PMS dependency is two-tiered: **soft surface** (own scrapers — needs monitoring + degraded-mode handling) and **hard surface** (practice-shared — ends at colleague pace, not at the developer's). Auto-adapt is fiction for server-side changes.

---

## The Dashboard

Single Axum web server on localhost:3457, serving all users.

### Clinic View

- Today's appointments with status (pending, arrived, completed, DNA, late-cancel)
- Observation textarea per client with draft persistence
- Streaming note generation via SSE — POST → `/api/generate-stream` → frontier LLM → per-chunk dephi-revert before display
- Accept/edit/reject workflow
- Dismiss with 8-second undo bar
- Letter cadence badge (L!) — click shows referrer name, practice, email
- Funding status badges per client

### Variant-Aware A/B Compare

The compare view uses `{model, rail, preset}` tuples for one-click side-by-side comparison. Accept = Save in one click. Parallel streaming. JSONL schema v2 with legacy fallback.

### Email Setup Wizard

First-run modal overlay. Auto-detects SMTP from the practitioner's email domain, tests delivery, stores password in OS keychain. OTP verification via the SMTP just configured. Config saved only after verification succeeds. Closed-loop — proves SMTP and email ownership in one step.

### Letter Drafting Panel

L! badge click opens inline drafting view. Loads draft, shows referrer info, editable textarea, Build PDF then Send then badge resets.

### Calendar and Scheduling

Create recurring series and one-off appointments. Cancel with reason. Calendar blocks panel. ICS export endpoint returns RFC 5545 output, optional practitioner filter, `Content-Disposition: attachment` for calendar-app subscription.

### Security Posture (Tier A)

Dashboard binds to 127.0.0.1 only. Session cookies HttpOnly + SameSite=Strict. Email login via OTP. Multi-practitioner deployment: expose over Tailscale, add TLS, add Secure cookie flag. The dashboard is **never** publicly internet-accessible.

---

## Client Onboarding Pipeline

Fully automatic, zero-touch onboarding from incumbent-PMS data.

**Incumbent API cache:** headless Chrome runs an in-browser `fetch()` against the incumbent's API (the PMS's edge layer blocks direct API calls). Cached at a few MB (thousands of clients), auto-refreshes when stale. Onboarding reads from cache — no browser launch needed per client.

**Client ID derivation:** first initial of forename + first initial of surname + last two digits of birth year. Same-person collision detection: matching DOB returns existing ID. Different person appends `b`, `c`, etc.

**Pipeline:** `practiceforge onboard` reads the diary capture, identifies unmapped clients, derives IDs (with automatic DOB lookup), scaffolds client directories (`identity.yaml`, `notes.md`, `correspondence/`, `admin/`), and populates identity fields.

---

## Email System

Direct SMTP via the `lettre` crate — no proprietary email broker for sending. Each practitioner configures their own email via the setup wizard with auto-detection from domain. Passwords stored in OS keychain.

**Auth broker (`pizauth`):** for OAuth-based providers (Microsoft 365 / Gmail), `pizauth` is the sole token holder. Tokens live in the daemon's encrypted memory, not in keychain — eliminating the keychain-ACL invalidation problem that earlier multi-binary topologies had. Mail consumers (`mbsync`, `meli`, `msmtp`, `practiceforge`) all call `pizauth show <account>`.

**Sending capabilities:** clinical letters with PDF attachment, billing reminders (tone-appropriate templates), portal share notifications, reschedule slot offers.

The portal (`clinical-portal` on Fly.io London) uses Resend for OTP delivery to letter recipients. This is separate from the practitioner email system.

**Email environment:** the practitioner uses `meli` (Rust TUI) for inbox curation, naked in WezTerm — no terminal multiplexer wrap, since meli's native tabs cover that need. A single forked patch to upstream meli (envelope-open + colon-stacking fixes) is in flight — the first OSS fork+patch contribution from this project.

**Mailcurator** sits alongside email as a per-sender lifecycle tool (extract-and-destroy principle, JSONL storage). Vendor-managed marketing emails are auto-trashed after extraction; order confirmations are extracted and routed; password-protected referrer PDFs are auto-decrypted using the per-referrer registry.

---

## Letter Pipeline

Two letter types: **update** (to referrer, every four to six sessions) and **authorisation** (to insurer, requesting session re-authorisation).

**Pipeline:** draft generation produces markdown. The build step renders to branded PDF via lualatex, optionally AES-256 encrypted (password = uppercase initials + DOB as DDMMYYYY, NHS convention). The send step orchestrates SMTP with PDF attachment, files to the incumbent PMS via `tm3-upload`, and notifies the practice administrator.

**Portal delivery alternative:** generates an OTP-authenticated link. Recipient enters email, receives 6-digit code, views the letter in-browser. View audit trail. Immediate revocation. Deployed on Fly.io London.

**Letter audience register:** letters to GPs and other professionals need plainer language than session notes. The prompt steers register; the practitioner's voice configuration does not.

**AI for letter drafting:** currently template-only (no LLM). Whether and how to add AI to letter drafting involves clinical-accountability and confab-risk considerations distinct from session notes — five design alternatives are sketched, decision deferred.

---

## Compliance Posture

### Per-Practitioner DPA Architecture (2026-05-06)

| Aspect | Position |
|---|---|
| AI relationship | Per-practitioner Anthropic API key with auto-incorporated commercial DPA |
| De-identification | Type-enforced at LLM boundary (`Dephid` newtype); not at file-system rest |
| Practice's role | Attaches at finalised-letter filing, not at AI drafting |
| Practice-level DPA | Not currently needed (only required if practice-level AI is added) |

### Regulatory Position

**USA (FDA / SaMD):** the PMS layer is unambiguously not a medical device — scheduling, billing, attendance, client database, document storage are all explicitly excluded. The AI note-generation layer falls under Clinical Decision Support (CDS), excluded from device regulation under the 21st Century Cures Act because it supports but does not drive the clinician's decision, and the clinician can independently review the basis for the recommendation.

**UK (MHRA):** same principle — software for storing/retrieving patient records and scheduling/billing/administration is not a medical device. Obligations are GDPR/ICO and professional indemnity (the practitioner's responsibility).

**Outcome measures (PHQ-9, GAD-7, etc.):** PracticeForge stores and retrieves forms but does not score or interpret them. Auto-scoring + treatment-change recommendation would enter SaMD territory.

---

## Multi-Practitioner Design

### Per-Practitioner Components

- Their own `practiceforge` binary (cross-platform: macOS, Linux, Windows)
- Their own `~/Clinical/` directory (or configured path)
- Their own incumbent-PMS session cookies
- Their own AI backend choice (Anthropic API or Claude CLI), key, and DPA
- Their own voice profile (wizard-configured)
- Their own modality configuration (`modality-act.md` / `modality-cbt.md` / etc.)
- Their own availability configuration (`availability.yaml`)
- Their own billing configuration (rate, reminder tones, escalation)
- Their own email configuration (SMTP, OTP-verified)

### Shared Components

- Central client registry (git-backed bare repo on practice server)
- Letter delivery portal (`clinical-portal` on Fly.io)
- SMS provider (Twilio account)

No shared GPU, no shared LoRA, no shared API key — the frontier-LLM-per-practitioner pivot eliminates the inference infrastructure that earlier designs centralised.

### Day-One Problem (Open)

A colleague on day one has zero past notes for voice exemplars. Options under evaluation: a generic "house style" exemplar pack, prompt-only guidance until 30–50 notes accumulate, hybrid approach. First impressions shape adoption.

### Cost Model

- Inference: per-practitioner Anthropic API spend, typically low single digits to low tens of pounds per month depending on volume and model tier
- SMS: shared Twilio cost for the practice
- Portal hosting: Fly.io shared (single tenant)
- Practice server: one-off hardware + low ongoing electricity

---

## Current Development Focus

### Leptos Refactor (commitment 2026-05-09)

Wholesale frontend migration from vanilla HTML/JS to Leptos (Rust → WebAssembly), planned as **6 phased sprints**. No new vanilla JS feature work meanwhile.

**Sprint review process:** belt-and-braces `/ultrareview` cloud + 5-agent in-context fan-out per sprint; approximately 45–90 minutes per sprint after Sprint 0 front-loading. Sprint 0 (foundation) merged 2026-05-09. Sprint 1 (in CI), Sprint 2 (in CI), Sprint 3 (clinic view, in review) running in parallel as of 2026-05-10.

### TM3 Replacement Track

Architectural decisions banked 2026-05-09 (see "Incumbent-PMS Retirement Direction" above). Implementation runs in parallel with the Leptos refactor. The incumbent dependency is structurally containable: soft-surface adapters (own scrapers) need monitoring; hard-surface API changes end at the colleagues' migration pace.

### Sub-Projects (Extractable as OSS)

- **Faithfulness filter** — could become an OSS library for any clinical AI tool
- **De-identification toolkit** — built around the `Dephid` newtype pattern
- **Iced focus system** — when Iced was the dashboard target, contributions to Iced upstream (tab traversal, focus tracking, arrow nav) were planned; deferred but the sub-project remains valid
- **meli envelope-open + colon-stacking patch** — first OSS fork+patch from this project; 2026-04-29

The architecture naturally matches an OSS shape (AGPL likely for the PMS layer; AI layer remains proprietary).

---

## Anti-Confab Policy (architectural)

Safety-critical filtering lives in **prompts and deterministic gates, never in model weights**. This means:

- Model swaps (Opus → Sonnet → Haiku → future frontier model) don't require redoing the safety work
- The same gates run against any backend
- A frontier-LLM pipeline (v5-Light) becomes feasible without retraining
- The system stays vendor-neutral at the safety boundary, not just the API boundary

This was reinforced by three concrete confab failures observed in Opus 4.7 testing (fabricated doctor name, in-session content not actually said, prompt-rail ignored) — case for deterministic gates rather than trusting model improvements.

---

## What Is Built (May 2026)

### Distribution

- **Two binaries** (post-consolidation): `practiceforge` and the transitional `tm3_download`. Reduces to one when the in-tree PMS replaces the legacy PMS.
- **Three workspace crates:** `practiceforge`, `clinical-core`, `clinical-portal`
- **Cross-platform:** macOS, Linux, Windows. No Apple Silicon dependency.

### Stable-Codesigned Tools (macOS)

`practiceforge`, `pizauth`, `tm3-diary-capture`, `mailcurator` are signed with a Developer ID Application certificate so TCC permissions and keychain ACLs persist across rebuilds. This was a real operational pain point earlier (rebuilds invalidating "Always Allow" prompts every 30 minutes); the cert obtained 2026-04-25 closes it permanently.

### Integrations

- **Incumbent PMS:** diary scraping (hourly, headless Chrome), document upload and download, client lookup via API cache, full export/migration
- **Fly.io:** secure letter portal (London region, persistent SQLite)
- **Twilio:** SMS appointment reminders
- **Tantivy:** full-text search across all client files
- **Anthropic API:** voice model inference (default backend)
- **lettre:** direct SMTP for emails and billing reminders
- **pizauth:** OAuth broker for Microsoft 365 / Gmail / Graph
- **Healthcode ePractice:** portal mapped for billing automation (awaiting TOTP secret)
- **Syncthing:** peer-to-peer sync between developer's machines
- **Git (bare repo):** central client registry on practice server

---

## Open Work

1. **Leptos sprints 1–5** continue through the planned 6-sprint cadence
2. **Incumbent-PMS retirement** — implementing the five architectural decisions banked 2026-05-09
3. **Letter-AI strategic decision** — five design alternatives sketched; decision pending on whether and how to add LLM to letter drafting
4. **Voice profile wizard polish** — exemplar curation UI, style description UX, RAG retrieval defaults
5. **Day-one experience** — what happens before any approved notes exist
6. **Multi-practitioner deployment** — TLS, per-practitioner identity, role-based access, multi-practitioner calendar view
7. **Observation/Interpretation structural separation** — proposed note shape, ~3–4 dev-days extension of GatingPolicy scaffolding
8. **Practice-level DNS** — referred-from-practice clients account for the majority; email from the practice domain requires DNS changes
9. **Faithfulness Layer 3** — frontier-API embeddings or local lightweight embedder to replace the deleted Ollama path
10. **Inference revisit (~2027)** — laptop-class local inference if/when frontier-tier models run on consumer hardware

---

## Architectural Philosophy

A few stable commitments behind the moving parts:

- **Vendor neutrality at the safety boundary, not just the API boundary.** Prompts and gates, not model weights.
- **Deterministic floors, AI ceilings.** MVDDA stands without AI; AI is a pluggable layer above. Notes are defensible regardless of model.
- **Tier separation is non-negotiable.** Tier A (practitioner-internal) and Tier B (public-facing) are different threat models, different code paths, different crates. The dashboard never crosses.
- **Friction-as-feature where it matters.** The mail flat unified inbox triggers curatorial hostility on purpose; Gmail-style auto-categorisation would unwind the discipline.
- **Per-practitioner choice is informed consent.** AI backend, voice profile, billing tone — all configured privately via wizards, not deliberated as a group.
- **One web UI for everyone.** The developer must dogfood the same UI colleagues use. A separate native frontend creates a testing gap.
- **Code extraction is mechanical.** Estimate 30 min – 2 h, not half-day, when extracting an existing module to a new crate or repo.

---

*This document describes the state of the system as of 10 May 2026. Frontier-LLM-per-practitioner inference; per-practitioner Anthropic DPA; Tier A/B separation; Leptos refactor in flight; in-tree PMS replacement under design. The system is under active development.*
