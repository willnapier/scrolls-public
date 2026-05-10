---
title: MailForge
---

# MailForge

*Vendor-neutral module. Last updated: 2026-05-10.*

---

## What It Is

MailForge is a **browser-native mail client** built as a thin, opinionated frontend over a notmuch index. The whole MUA — list, read, compose, send, tag, search — runs through `http://127.0.0.1:8765/mail` in any browser tab. Sync stays in `mbsync` and `lieer`; outbound stays in `msmtp` and a Microsoft Graph send helper; OAuth tokens stay in a `pizauth` daemon. MailForge is the **UI layer only**.

It started as a small browser-handoff helper: pair with a TUI MUA via mailcap, render `text/html` and `application/pdf` parts in a real browser instead of in the terminal, and stop pretending terminals are good at HTML. That worked well enough that the next step was inevitable — move the rest of the MUA into the browser too.

The thesis: eight years of trying to make terminals do rich content well has consistently produced compromises. The browser already renders HTML, PDFs, inline images, and copy-paste correctly. notmuch already indexes the mail. mbsync, lieer, msmtp, pizauth already handle transport. The missing piece is a UI surface that respects all four. MailForge is that surface, in roughly 5,000 lines of Rust + a small amount of vanilla JS.

License: AGPL-3.0. Personal-tool-grown-into-a-product. Currently single-user; multi-user adjustments are enumerated and tractable but not the near-term focus.

---

## Why It Exists

The TUI mail clients (meli, aerc, mutt) are excellent at TUI-shaped tasks: keyboard navigation, threading, tag-based filing, fast scroll through dense listings. They are structurally bad at:

- HTML email — even the best TUI HTML renderers (`w3m -T text/html`, `lynx -dump`) produce visually-degraded output that loses formatting an Outlook-composed multipart message intends
- PDF attachments — terminals don't render PDFs; the answer is always to shell out
- Inline images — see HTML
- Copy-paste — terminal selection semantics fight the user across sender / subject / body boundaries
- Mouse-driven workflows — fundamentally a TUI mismatch, but real people sometimes want a click

The "Text piped through `w3m -T text/html`" banner that a TUI MUA prints once per HTML part is the philosophy showing through: it's faithfully rendering MIME structure, but the trade-off is that visually-formatted emails are degraded.

Rather than fork the TUI or migrate to a polished cousin, the cleaner answer is to route HTML and PDF through what already renders them excellently — the browser — and then notice that *all* the other mail UI primitives also work better in the browser. So MailForge moves the whole MUA there, while leaving sync, send, and OAuth in the well-trodden CLI tooling.

---

## Architecture

### Layers

```
┌─ Browser (the rendering surface)─────────────────────┐
│  Vanilla HTML/JS today; Leptos refactor in flight    │
│  Keyboard-first; mouse where it actually helps       │
└──────────────────────┬───────────────────────────────┘
                       │ HTTP / SSE
┌──────────────────────▼───────────────────────────────┐
│  MailForge daemon (Axum, on 127.0.0.1:8765)           │
│  Reads + writes via notmuch CLI + mail-parser         │
│  Renders mailbox listings, message views, composer   │
│  CSP-sandboxes HTML email in iframes                 │
│  Caches rendered messages under ~/.cache/mailforge/   │
└──────┬─────────────────┬─────────────────┬──────────-┘
       │                 │                 │
┌──────▼──────┐  ┌───────▼───────┐  ┌──────▼──────────┐
│  notmuch    │  │  mailcurator  │  │  msmtp /        │
│  (index)    │  │  (lifecycle)  │  │  graph-send     │
│             │  │               │  │  via pizauth    │
│  ~/.notmuch │  │  policies.toml │  │                 │
└──────┬──────┘  └───────────────┘  └─────────────────┘
       │
┌──────▼──────────────┐
│  Maildir on disk    │
│  (one tree per      │
│   account: personal │
│   + practice)       │
└──────┬──────────────┘
       │
┌──────▼──────────────┐
│  mbsync / lieer     │
│  (inbound sync)     │
└─────────────────────┘
```

The whole stack is composable, single-purpose, and replaceable a piece at a time. notmuch can be swapped for another tag-based index. mbsync can be swapped for lieer or vice versa per account. msmtp can be swapped for any RFC-compliant SMTP. The MailForge daemon is opinionated UI glue — none of the underlying components depend on it.

### Goals & Non-Goals

**Goals:**
- Browser-native mail UI that respects how the OS actually works (HTML, PDF, copy-paste, mouse, inline images)
- Keyboard-first navigation that matches a TUI's speed for a TUI user, while not punishing mouse-driven moments
- Thin: stay a UI over notmuch + transport tools rather than reimplement IMAP/SMTP
- Single-purpose: read and write mail. Nothing else.

**Non-goals:**
- Not a TUI fork. The browser DOM is the rendering surface; TUI primitives don't apply.
- Not a webmail product. Binds to localhost only. No login, no auth, no TLS.
- Not multi-user. One human, one host, one browser session.
- No IMAP/SMTP client (mbsync, lieer, msmtp own those). No contacts DB, no calendar.
- No Windows port today.

### URL Scheme

```
GET  /mail                              redirect → /mail/<default>/inbox
GET  /mail/<account>/<mailbox>          mailbox listing
GET  /mail/<account>/<mailbox>?page=N   pagination
GET  /mail/<account>/<mailbox>?q=...    in-mailbox search
GET  /mail/m/<message-id>               read view (single message)
GET  /mail/t/<thread-id>                thread view (multi-message)
GET  /mail/compose                      blank composer
GET  /mail/compose?reply=<msg-id>       reply composer (prefilled)
GET  /mail/compose?fwd=<msg-id>         forward composer (prefilled)
GET  /mail/search?q=...                 cross-mailbox search

POST /api/tag                           {ids, add, remove}
POST /api/trash                         {ids}    (sugar: +trash -inbox)
POST /api/archive                       {ids}    (sugar: -inbox)
POST /api/seen                          {ids}    (sugar: -unread)
POST /api/send                          form-encoded; redirects on success
POST /api/draft                         autosave composer state
POST /api/mailcurator/sweep             trigger curator policy
POST /api/mailcurator/blacklist         install + run kill-sender policy
POST /api/unsubscribe/execute           RFC 8058 one-click + browser fallback
POST /api/html-trusted/{add,remove}     manage HTML auto-render trust list
```

### CSP-Locked Iframe Surface — `/v/<uuid>/...`

The original browser-handoff feature serves rendered HTML emails at `/v/<uuid>/`. Every HTML email gets:

- **Content-Security-Policy on the body iframe**: `script-src 'none'` (always — JavaScript blocked unconditionally), `frame-src 'none'`, `object-src 'none'`, `base-uri 'none'`, `form-action 'none'`. External HTTPS images allowed by default with a one-click toggle to block.
- **iframe `sandbox=""`** — no scripts, no same-origin
- **URL anchor collapsing** — anchors whose visible text is a long URL get visible text replaced with `[<domain>]`; `href` preserved intact. Click-through still works.

The `/v/<uuid>/` surface is a hard security boundary — it never gets folded into the main MailForge UI routes. Even when the rest of MailForge migrates to a new framework, this boundary stays exactly where it is. Sprint plans for the Leptos refactor explicitly exclude `/v/<id>/...` from migration.

---

## Components

### notmuch as the Backend

notmuch is a tag-based mail index built on Xapian. It indexes mail bodies and headers, supports a structured query language (`from:`, `to:`, `subject:`, `date:`, `tag:`, boolean `and/or/not`), and treats tags as the primary metadata. MailForge reads via the `notmuch show` and `notmuch search` CLI; writes via `notmuch tag`.

**Why notmuch and not literal `rg` over the maildir:** notmuch's Xapian index makes "rg-grade speed" feasible at scale. A single query over a 200,000-message, 16GB mailstore takes 50–200ms regardless of size; `rg` would scan the whole tree every query. notmuch indexes message bodies on import, so the practical search experience IS the "search emails like rg" point — fast, full-content, with structured operators on top.

If a future need arises for literal byte-level regex (e.g. searching raw HTML markup that notmuch strips during indexing), an `&engine=rg` mode could be added. Out of scope for now.

### Path-Based Account Discrimination (changed 2026-05-08)

Mailbox queries scope by physical maildir path, not by tag.

The query for a practice mailbox is `path:practice/** and tag:inbox and ...`; for a personal mailbox it's `tag:inbox and not path:practice/** and ...`. Same shape for the other mailboxes.

**Why** — notmuch deduplicates messages by Message-ID. A message that arrived at BOTH a personal Gmail address AND a practice M365 address is ONE notmuch document with multiple file paths. A `tag:practice` discriminator is on the document, so it leaks across both views. Path-based scoping is the honest discriminator: each maildir holds physically distinct files, so `path:practice/**` matches exactly the practice corpus regardless of any tag-drift between Gmail-label sync and mbsync state.

**Symptom that drove the fix**: trashing or archiving from one view would unexpectedly affect the other for dual-stored messages. Archiving in the Gmail web UI would remove `tag:inbox` from the dedup'd document, which silently emptied the practice inbox of dual-stored messages even though the M365 server still had them in Inbox.

This is a small architectural correctness story but it's load-bearing — the wrong discriminator silently corrupts user mental models for years before anyone catches it.

### Inbound Sync

- **mbsync** for IMAP-protocol accounts (typically the practice's M365 account)
- **lieer** for Gmail-API-protocol accounts (typically personal)

Both run on a timer. Neither is invoked from MailForge; they exist independently. MailForge reads whatever they wrote.

`gmail-push-tags` is a sister tool that pushes notmuch tag changes UP to Gmail (notmuch as source of truth). One subtlety: archiving directly in the Gmail web UI is fragile — the next push tick may re-add `INBOX`. Archive via MailForge instead; the tag change propagates to Gmail correctly on the next tick.

### Outbound Send

- **msmtp** for SMTP-capable accounts (Gmail)
- **graph-send** (a small custom tool) for Microsoft Graph accounts where the tenant blocks SMTP AUTH

Both consume tokens from `pizauth` via `pizauth show <account>`. Tokens live in the daemon's encrypted memory, not in keychain. This eliminates a class of macOS keychain-ACL issues that an earlier multi-binary email stack suffered.

### Browser as Renderer

The browser (any modern one — Firefox, Chrome, Safari) is the rendering surface. MailForge serves HTML pages with:

- A top bar (account picker, mailbox sidebar, current-mailbox title, search box, compose button)
- A listing table (date, sender, subject, tags, snippet)
- A read view (header banner + iframe-sandboxed body for HTML, plain pre-wrapped for text)
- A composer (plain `<form>` with progressive enhancement for autosave-on-keystroke)

Vanilla HTML/JS today. Leptos refactor in flight (see Current Development).

### MailForge Daemon

A single Axum binary listening on 127.0.0.1:8765. Same daemon serves the original `/v/<uuid>/` browser-handoff surface and the new `/mail/...` MUA surface. One launchd plist on macOS, one systemd-user unit on Linux.

Cache directory: `~/.cache/mailforge/` (rendered HTML emails, inline assets). Safe to `rm -rf` at any time — URLs become 404, no data loss.

---

## Privacy & Security Posture

### Localhost-Only

The daemon binds to 127.0.0.1. Nothing reaches the local network, never mind the internet. No login screen because there's no exposure surface to log in from. This is a deliberate, defensible posture for personal mail tooling — the threat model is "another local user on this machine," and the answer is "this is a single-user laptop." If MailForge is ever exposed (e.g. over Tailscale), TLS + auth would need to be bolted on; currently absent.

### CSP + iframe sandbox on rendered HTML

See `/v/<uuid>/...` above. Every HTML email opens inside an iframe with no scripts, no same-origin, no form action, no base URI — independent of any sender trust decision.

### Trusted-Sender HTML Auto-Render (added 2026-05-04)

Default behaviour for HTML messages is plaintext-preferred (press `v` to escalate to the iframe view). For senders the user has explicitly opted into, MailForge auto-renders HTML on first open — but only when the message's `Authentication-Results` header confirms DMARC/SPF/DKIM passed.

The decision tree:

- `dmarc=pass` → trusted, OR
- `dkim=pass` AND `spf=pass` (no DMARC published) → trusted
- explicit `*=fail` → forced plaintext + visible warning chip
- header missing → silent fallback to default (plaintext)

The trust list says "I'm okay with HTML chrome and tracker pixels for THIS sender." It is **NOT** a security exception: `script-src 'none'` and the iframe sandbox stay in force regardless. The DMARC/SPF/DKIM gate exists specifically so an attacker spoofing a trusted domain can't piggyback on the user's trust decision.

`?view=full` remains the explicit-HTML escape hatch; it works regardless of trust state.

### URL Anchor Collapsing

Anchors whose visible text is a long URL (or just plain long-without-spaces ≥80 chars) get visible text replaced with `[<domain>]` while `href` is preserved intact. Triggers: visible text == href, or starts with `http(s)://`, or is ≥80 chars without spaces. Anchors with nested HTML (`<a><img></a>`) pass through untouched.

No third-party shortener service is used. All processing is local.

---

## Curatorial Philosophy

### Friction-as-Feature

The flat unified inbox triggers curatorial hostility on purpose. Gmail-style auto-categorisation (Promotions, Updates, Forums, etc.) hides the volume of low-value email behind tabs, which silently trains the user to tolerate sender behaviour they would not tolerate if forced to look at it.

MailForge resists that. The default view is a flat list, sorted by date, of everything that isn't already classified-and-trashed. Senders generating noise are visible as noise. The curatorial impulse fires while looking at a row, and the UI surfaces zero-click affordances for acting on that impulse exactly when it fires.

This is a deliberate design commitment, not an oversight. Auto-categorisation would unwind the discipline.

### Attention Metric, Not Raw Unread

Sidebar mailbox counts show "new + unread in the last 24 hours," not raw unread. Raw unread accumulates stale residue (a four-month-old unread newsletter shouldn't pull attention the way a newly-arrived message should).

This converges with "everything in the inbox is something I haven't decided about yet" as the extract-and-destroy curator workflow matures. The end state is an inbox where unread count = recent-arrivals count = work to do.

### Per-Row Hover-Reveal Action Icons

Two affordances live as per-row hover-reveal icons in the From column of the listing view:

- **Sweep** (`broom`) — appears only on rows whose message carries a `curator-<name>-seen` tag (i.e. matched by an active curator policy). Click sweeps all messages matched by that single policy via `mailcurator run --now --only <policy>`.
- **Unsubscribe** (`unsub`) — appears only on rows whose message has a `List-Unsubscribe` header. Click runs the RFC 8058 one-click POST flow server-side (when supported), or hands off to the browser for cookie-bound https confirmation pages and `mailto:` URLs. On success the message is tagged `unsubscribed` (audit trail) and `trash` (clears the row).

Both gated by `data-*` attributes on the `<tr>`: `data-curator-policies` and `data-has-unsubscribe="true"`. CSS opacity rules keep the icon truly invisible (opacity 0 + pointer-events:none) until row hover — no visual noise on rows that aren't actionable. Keyboard shortcuts: `S` for sweep, `U` for unsubscribe, both acting on the cursor row.

The design philosophy: surface the curatorial trigger at zero clicks from where the impulse fires.

### Kill-Sender Keystroke (`K`)

Press `K` (capital, intentional — destructive actions get Shift) while reading a message OR on a listing row to install a future-protection policy for that sender's domain in the curator's blacklist.

**`K` does NOT immediately delete existing messages from the sender.** It installs a conditional policy (`intended_categories = ["bulk-marketing"]`, `delete_after_days = 1`) that catches future bulk-marketing classifications from this domain. To trash existing messages, use the explicit follow-up: filter `from:@<domain>` then `Ctrl+D` (bulk-trash by filter). This is the deliberate Unix-composability split — `K` is "future protection," `/from:@x⏎⌃D` is "purge existing," combine them when both are wanted.

### Bulk-Trash by Filter (`Ctrl+D`)

When the listing has an active `?q=` filter, `Ctrl+D` bulk-trashes everything matching across all pages. Server endpoint reuses listing's mailbox query + filter logic; refuses empty `q` (server) and refuses without `?q=` (client) — belt-and-braces against accidentally trashing whole mailboxes. Confirm dialog quotes the verbatim filter and count.

### Cross-Mailbox Search (`/` and `//`)

Single `/` focuses the per-mailbox filter. Quick `//` within 350ms escapes to `/mail/search` from anywhere — listing, message view, composer, search page itself. The chord works even after the filter input has taken focus from the first `/`.

`/mail/search?q=<notmuch-query>` runs full-text search across the entire notmuch index. Plain words match bodies AND headers; structured operators layer on top.

---

## mailcurator Integration

`mailcurator` is a per-sender email lifecycle tool that runs alongside MailForge:

- **Extract-and-destroy principle** — for known sender categories (vendor marketing, order confirmations, password-protected referrer PDFs), extract the value, then trash the email
- **JSONL storage** — extracted data lives in append-only logs the user owns
- **Two-tag idempotency** — re-tuning extractors needs both `-seen` and `-extracted` tags cleared; otherwise silent no-op
- **Quarantine first on new destroy policies** — vendors often share a single sender between marketing and order confirmations; always preview before flipping live
- **Subscription monitor** — separate `list/check/report/discover` subcommands track subscription lifecycle

MailForge writes mailcurator policies via small HTTP endpoints that do "write file + trigger":

- `POST /api/mailcurator/sweep?only=<name>` — pure trigger; no file mutation. Powers the per-row Sweep icon and `S` shortcut.
- `POST /api/mailcurator/blacklist` — write the policy file (atomic, idempotent on policy name) THEN trigger `--only <new-name>`. Powers the `K` keystroke.

Both endpoints shell out to the standalone `mailcurator` binary. MailForge holds no policy state in process. The configuration file IS the source of truth — `~/.config/mailcurator/policies.toml` is the listing/edit surface. This keeps MailForge stateless w.r.t. mailcurator and lets the user edit or remove policies by hand without coordinating with the daemon.

**Why string-append instead of TOML round-trip:** `policies.toml` is hand-edited and carries comments. A standard TOML round-trip would either drop those comments or pull in a span-preserving editor crate. Plain string append at the bottom + atomic rename preserves the existing structure exactly.

---

## TUI Fork — `meli` Patches

MailForge originated as a browser-handoff feature that paired with `meli` (a Rust TUI MUA) via mailcap. That coexistence is preserved: the user can still drive `meli` and have HTML/PDF parts open in MailForge's `/v/<uuid>/` viewer.

A small `meli` fork at `branch fix-envelope-and-colon` provides two opt-in flags:

1. **`x-meli-pipe-message` mailcap extension** (manual, via `m` shortcut) — when set, meli writes the parent envelope's RFC822 source to the helper's stdin instead of just the decoded part. This makes cid: assets resolvable in the browser-handoff path.
2. **`pager.html_filter_pipe_message` config flag** (automatic, on Enter) — when set, the pager's html_filter receives full RFC822 bytes instead of just the decoded HTML part. Pressing Enter on an HTML email auto-launches the browser (no separate keypress).

Both patches are defensive (opt-in flags, no behavioural change for existing configs) and small (~200 lines each incl. tests). Worth proposing upstream once they soak. This was the first OSS fork+patch contribution from the broader project — "fork upstream" is now a real option, not a theoretical one.

The fork ships as a patched binary at `~/.local/bin/meli`. Users without the patch fall back to the standalone `mailforge-html` shell wrapper (RFC 1524 `%s` tempfile convention) — degraded but functional.

---

## Current Development

### Leptos Refactor (Sprint 0 reviewed 2026-05-10)

MailForge is migrating from vanilla HTML/JS to Leptos (Rust → WebAssembly), in phased sprints aligned with the parallel PracticeForge Leptos refactor. The cdylib/wasm split is correct: tokio/axum/lettre/notmuch are kept out of wasm via `[target.'cfg(not(target_family = "wasm"))'.dependencies]`.

**Sprint 0 (foundation)** is reviewed but not yet merged — coordination with parallel work in flight. The CSP-locked `/v/<id>/...` iframe surface is explicitly excluded from migration and protected by a regression test that asserts the Leptos router does not register `/v` routes.

**Review process** mirrors PracticeForge's: belt-and-braces cloud review (`/ultrareview`) plus a 5-agent in-context fan-out per sprint. Review fixups for safety/build-hygiene/prod-deploy concerns are committed before merge rather than after.

### TUI-fork Upstreaming

Both `meli` patches sit in a private fork branch waiting to soak before being proposed to the project. No urgency — the local deployment works.

---

## Tenancy Posture

MailForge is single-user-tooling-grown-into-a-product. Most distribution adjustments are mechanical (one-line constants → config), not architectural. The architectural decisions (notmuch backend, browser-handoff for HTML, sandboxed iframes, three-tier nav scheme) carry over to a multi-user edition without change.

### Bake-In Points (current)

Things hardcoded today that need to become config-driven before MailForge is usable by another person:

1. **Account identities** — currently a compiled-in `const` with display names, identities, tag gates, and send-backend selection. Move to runtime-loaded `~/.config/mailforge/accounts.toml`.
2. **Keyboard layout** — `e` / `i` for row down / up matches a Colemak-DH home row at the same positions a QWERTY user expects `j` / `k`. Introduce a layout config or per-user keymap override file.
3. **Mailbox query conventions** — per-account mailbox→query map should ship from config rather than baked into source.
4. **External tooling assumptions** — startup check that warns if expected binaries (notmuch, mailcurator, mbsync/lieer, msmtp, pizauth, helix) are missing, with clear "what's broken without this" messaging.
5. **Daemon configuration** — port should be overridable via env var. Bind address stays localhost-only (security feature; not a casual-config item).
6. **Theme** — Solarized palette by default; override via user CSS or theme picker if/when there's demand.
7. **Visible tag chips** — current set is the developer's choice; per-user override file needed.
8. **Sweep button** — feature-detect `mailcurator` at startup; hide button when absent.

### End-State Distribution

- Single-binary install (cargo-install or homebrew tap)
- First-run wizard creates config files from a template + interactive prompts
- Tooling-prereqs check at first run
- Public AGPL-3.0 repo when matured

The order of adjustments tracks user-facing pain: accounts (item 1) first, then keyboard (item 2), then tooling matrix (item 4), then the rest.

---

## Architectural Philosophy

A few stable commitments behind the moving parts:

- **Do one thing well.** MailForge is the UI. Sync is mbsync/lieer. Send is msmtp/graph-send. Auth is pizauth. Index is notmuch. Each piece is replaceable independently.
- **The browser is a better mail rendering surface than a terminal.** This is the founding observation. Don't fight it.
- **Localhost-only is a feature.** No login because there's no exposure. No TLS because there's no network. Single-user single-host single-tab.
- **Friction-as-feature.** The flat unified inbox is intentional. Auto-categorisation hides the work; surface it instead.
- **Curatorial impulse fires while looking at a row.** Zero-click affordances at the point of fire (per-row hover-reveal icons; `K` and `S` keystrokes).
- **Configuration files ARE the source of truth.** mailcurator policies live in `policies.toml`; HTML trust lives in a JSON file; no hidden state in a database the user can't read.
- **String-append over schema round-trip.** When a config file carries human comments, atomic-append + rename preserves the structure exactly. Don't reach for a TOML library that drops comments.
- **Path discrimination over tag discrimination** for dedup'd notmuch documents that exist in multiple maildirs simultaneously. Tags can drift; physical paths cannot.
- **Patch upstream before forking.** Both `meli` patches are opt-in flags, no behavioural change for existing configs, ~200 lines each. Designed to be upstreamable.
- **Code extraction is mechanical.** When MailForge was extracted from a parent monorepo to its own crate, the work was 30 minutes — not the half-day a "greenfield" estimate would have suggested. The architecture was already shaped for extraction.

---

## Stack Summary

| Layer | Technology |
|-------|-----------|
| UI surface | Browser (Firefox / Chrome / Safari) |
| Daemon | Axum (Rust) on 127.0.0.1:8765 |
| Frontend | Vanilla HTML/JS migrating to Leptos (WebAssembly) |
| Mail index | notmuch (Xapian-backed) |
| Inbound sync | mbsync (IMAP), lieer (Gmail API) |
| Outbound send | msmtp (SMTP), graph-send (Microsoft Graph) |
| OAuth broker | pizauth |
| Lifecycle automation | mailcurator (separate binary) |
| HTML email render | iframe sandbox + CSP, mail-parser for cid: rewriting |
| TUI coexistence | meli (forked) — `m` shortcut + auto-render Enter |
| Daemon launcher | launchd (macOS), systemd-user (Linux) |
| License | AGPL-3.0 |

---

## Open Work

1. **Leptos sprints** — Sprint 0 reviewed; Sprint 1 (mailbox sidebar — smallest cohesive surface) proposed. Continue through phased sprints.
2. **`meli` patches upstream** — propose both patches once they have soaked locally for a few days.
3. **Account config externalisation** — move `ACCOUNTS` from `const` to runtime TOML; the highest-priority distribution adjustment.
4. **Keyboard layout config** — second-highest distribution adjustment.
5. **Tooling pre-flight** — startup check for required binaries with clear "install this before continuing" messaging.
6. **Cache pruning** — `~/.cache/mailforge/` grows over time. LRU eviction at startup or daemon-side TTL.
7. **`/mail/search` engine modes** — `&engine=rg` mode for byte-level regex over the maildir, complementing the default Xapian-indexed search.
8. **Reply-in-TUI link** — would require IPC back to the patched `meli`. Lower priority.

---

*This document describes MailForge as of 10 May 2026. Personal-tool-grown-into-a-product, single-user today, AGPL-3.0, multi-user adjustments enumerated and tractable. Browser-native by choice; thin UI over notmuch + transport tools by architecture; localhost-only by security posture.*
