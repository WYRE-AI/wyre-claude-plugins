# WYRE email signature

```
/plugin install wyre-email-signature --marketplace WYRE-AI/wyre-claude-plugins
```

Skill-only plugin that replaces the retired
[sig.wyre.ai](https://sig.wyre.ai) generator. An agent builds a
self-contained, Outlook- and Gmail-safe HTML signature from name, title,
brand, contact fields, and an optional event line — no website required.

## Skill

| Skill | Covers |
|---|---|
| `wyre-email-signature` | Inputs, brand constants, HTML template (variant C default from wyre-sig), Cadence booking links, optional event banner, install steps for Outlook web / Windows / Mac and Gmail |

## Example prompts

- “Make me a WYRE Technology signature”
- “WYRE AI signature for Aaron, office (423) 993-0290, book via Cadence”
- “Add Channel Partners Orlando to my signature for Mar 2027”
- “Remove the event line from my signature”

The agent should return the HTML fragment (and briefly how to paste it).

## Brand assets

Signature images stay on the public `sig.wyre.ai` worker (immutable PNGs).
Do not embed base64 or relative paths — mail clients need absolute `https://`
URLs.

## Privacy

This marketplace is public. Do not commit personal cell numbers, home
addresses, or secrets. Office phones, work emails, and public brand assets
are fine. Cell is always an optional runtime input.

## External references

- Retired generator source (private, do not modify for this work):
  `https://github.com/WYRE-AI/wyre-sig`
- Cadence booking: `https://cal.wyre.ai` (team default; per-person slugs
  look like `https://cal.wyre.ai/<userSlug>/<eventSlug>`)
- WYRE Technology site: `https://wyretechnology.com`
- WYRE AI / Conduit: `https://wyre.ai`, `https://conduit.wyre.ai`
