---
name: "WYRE Email Signature"
description: >
  Create or update a WYRE AI or WYRE Technology email signature as
  self-contained HTML (inline styles, absolute https image URLs) that is
  safe for Outlook and Gmail. Replaces the retired sig.wyre.ai generator.
when_to_use: >-
  When someone asks for a new or updated WYRE email signature, a WYRE AI
  or WYRE Technology signature block, an event banner on their signature
  (for example Channel Partners Orlando), or help pasting a signature into
  Outlook or Gmail. Use when: email signature, WYRE signature, WYRE AI
  signature, WYRE Technology signature, sig.wyre.ai, Channel Partners
  signature, Outlook signature, Gmail signature.
---

# WYRE Email Signature

## Overview

Build a table-based HTML signature fragment for WYRE staff. Layout,
colors, and assets match the retired Cloudflare Worker generator
(`wyre-sig`, default layout **C · Message-led**). Output is a fragment
(no `<html>` wrapper) unless the user asks for a downloadable `.html`
file.

This skill does **not** call an API. Assemble HTML from the inputs and
the templates in sibling files. Never invent personal cell numbers or
home addresses — ask for cell if needed; leave it out when omitted.

## Inputs

Collect before rendering. Mark optional fields only when the user
supplies them.

| Input | Required | Notes |
|---|---|---|
| Name | yes | Display name |
| Title | yes | Job title as they want it shown |
| Brand | yes | `WYRE AI` or `WYRE Technology` — see [brands.md](brands.md) |
| Email | yes | Work email (local@domain for that brand) |
| Office phone | yes | Default office line `(423) 993-0290` if they do not specify |
| Cell | no | Personal mobile — never invent; never commit to the repo |
| Booking link | no | Cadence URL; default `https://cal.wyre.ai` when they want a book button |
| Booking label | no | Default `Book time with me` (people) or `Book a demo` / `Schedule time with us` (shared) |
| Photo URL | no | Absolute `https://` only; Aaron headshot: `https://sig.wyre.ai/photo/aaron.png?v=2` |
| Pronouns | no | Shown muted after the name |
| Location | no | Overrides brand address |
| Event | no | Promo line from [events.md](events.md) or a one-off the user describes |
| Layout variant | no | `a` Quiet, `b` Dual-action, `c` Message-led (default) — see [template.md](template.md) |

## Build steps

1. Resolve **brand** constants from [brands.md](brands.md) (company, domain,
   website, logo, social, product CTA).
2. Escape every user string for HTML (`&`, `<`, `>`, `"`, `'`).
3. Build contact line: office phone, optional cell (separated the same
   way), then `mailto:` email — muted middots between parts.
4. If booking is requested or implied, use an `https://` Cadence URL only.
   Team default is `https://cal.wyre.ai`. Per-person slugs look like
   `https://cal.wyre.ai/<userSlug>/<eventSlug>` (for example
   `https://cal.wyre.ai/aaron/intro`).
5. Render the chosen variant from [template.md](template.md). Prefer
   variant **c** unless the user picks another.
6. If an event is active or requested, prepend the event row from
   [events.md](events.md). To remove an event after it ends, re-render
   without that row (and clear or mark the event inactive in `events.md`
   when updating the skill for everyone).
7. Return the HTML fragment in a fenced `html` block, plus a one-line
   reminder of how to paste it (details in [apply.md](apply.md)).

## Output rules

- Self-contained fragment: nested `<table role="presentation">`, all
  styles inline, width 520px.
- Absolute `https://` image URLs only (hosted on `sig.wyre.ai`). No
  relative paths, no `data:` URIs, no `javascript:` links.
- Font stack: `Arial,Helvetica,sans-serif`. Ink `#1a1a1a`, muted
  `#6b6b6b`, cyan `#00C9DB`, lime `#97D700`.
- Keep the fragment under ~5,000 bytes and ≤5 images when practical
  (Outlook.com editor ceiling from wyre-sig).
- Every `<img>` needs `width`, `height`, `alt`, `style="display:block;border:0;..."`.
- Office phone default is `(423) 993-0290` unless the user supplies another.

## Example prompts and responses

| User says | Agent returns |
|---|---|
| “Make me a WYRE Technology signature” | Ask for any missing inputs (name, title, email), then HTML for brand Technology, no event line, plus paste hint |
| “WYRE AI signature for Aaron Sachs, Co-founder & CTO, aaron@wyre.ai, office (423) 993-0290” | Full AI-brand HTML (see example files); offer booking button via Cadence |
| “Add Channel Partners Orlando to my signature for Mar 2027” | Same signature with the Channel Partners event row from [events.md](events.md) |
| “Remove the event line from my signature” | Re-render without the event row |

Worked examples (open and reuse):

- [example-aaron-technology.html](example-aaron-technology.html)
- [example-aaron-technology-channel-partners.html](example-aaron-technology-channel-partners.html)
- [example-aaron-ai.html](example-aaron-ai.html)

## Updating events without a website

Edit [events.md](events.md): add a row for a new promo, set `active` to
`yes` while it should appear by default, set `active` to `no` (or delete
the row) after the event. One-off copy the user dictates does not require
a file change — bake their text into this render only.

## Apply in mail clients

See [apply.md](apply.md) for Outlook web, Outlook desktop (Windows and
Mac), and Gmail.

## Privacy

Public repo: never commit personal cells, home addresses, tokens, or
private roster fields. Office phone, work email, and `sig.wyre.ai` assets
are fine. Cell is runtime-only.
