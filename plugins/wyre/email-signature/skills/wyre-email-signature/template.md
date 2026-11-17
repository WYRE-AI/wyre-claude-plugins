# HTML template (from wyre-sig)

Constants: `FONT=Arial,Helvetica,sans-serif`, `INK=#1a1a1a`,
`MUTED=#6b6b6b`, `CYAN=#00C9DB`, `LIME=#97D700`, `BTN_TEXT=#0a0a0a`.

Default layout is **c**. All values below are placeholders — substitute
escaped brand and person fields.

## Shared pieces

**Spacer** (`px` vertical):

```html
<tr><td style="height:PXXpx;line-height:PXXpx;font-size:1px;">&nbsp;</td></tr>
```

**Lime rule + photo** (omit photo `<td>`s when no `photoUrl`):

```html
<td width="88" valign="top" style="width:88px;"><img src="PHOTO_URL" width="88" height="88" alt="NAME" style="display:block;border:0;border-radius:50%;"></td>
<td width="14" style="width:14px;font-size:1px;line-height:1px;">&nbsp;</td>
<td width="3" bgcolor="#97D700" style="width:3px;font-size:1px;line-height:1px;">&nbsp;</td>
<td width="16" style="width:16px;font-size:1px;line-height:1px;">&nbsp;</td>
```

**Badge** (variants a/c):

```html
<td width="16" style="width:16px;font-size:1px;line-height:1px;">&nbsp;</td>
<td width="72" valign="top" style="width:72px;"><a href="WEBSITE"><img src="LOGO_URL" width="72" height="72" alt="COMPANY logo" style="display:block;border:0;border-radius:16px;"></a></td>
```

**Name / title / contact / site / address:**

```html
<tr><td style="font-family:Arial,Helvetica,sans-serif;font-size:17px;font-weight:700;line-height:22px;color:#1a1a1a;">NAME</td></tr>
<tr><td style="font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:19px;color:#6b6b6b;">TITLE</td></tr>
<tr><td style="height:8px;line-height:8px;font-size:1px;">&nbsp;</td></tr>
<tr><td style="font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:19px;color:#1a1a1a;">OFFICE&nbsp;&nbsp;&middot;&nbsp;&nbsp;<a href="mailto:EMAIL" style="color:#1a1a1a;text-decoration:underline;">EMAIL</a></td></tr>
<tr><td style="font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:19px;"><a href="WEBSITE" style="color:#00C9DB;font-weight:700;text-decoration:underline;">DOMAIN</a></td></tr>
<tr><td style="font-family:Arial,Helvetica,sans-serif;font-size:12px;line-height:18px;color:#6b6b6b;">ADDRESS</td></tr>
```

If cell is provided, insert it after office with the same middot separator
before the email link. If pronouns are provided, append a muted span after
the name.

**Booking button:**

```html
<table role="presentation" cellpadding="0" cellspacing="0" border="0"><tr><td bgcolor="#97D700" style="border-radius:6px;"><a href="BOOKING_URL" style="display:inline-block;padding:9px 16px;font-family:Arial,Helvetica,sans-serif;font-size:13px;font-weight:600;line-height:16px;color:#0a0a0a;text-decoration:none;">BOOKING_LABEL &#8594;</a></td></tr></table>
```

**Social row** (skip when empty):

```html
<table role="presentation" cellpadding="0" cellspacing="0" border="0"><tr>
<td><a href="SOCIAL_URL"><img src="SOCIAL_ICON" width="28" height="28" alt="COMPANY on LABEL" style="display:block;border:0;border-radius:6px;"></a></td>
<td width="10" style="width:10px;font-size:1px;line-height:1px;">&nbsp;</td>
<!-- more icons -->
</tr></table>
```

## Variant c · Message-led (default)

Outer wrap:

```html
<table role="presentation" cellpadding="0" cellspacing="0" border="0" width="520" style="width:520px;">
  <!-- optional event row + spacer — see events.md -->
  <tr><td><!-- personal zone: photo | rule | text+book | badge --></td></tr>
  <tr><td style="height:14px;line-height:14px;font-size:1px;">&nbsp;</td></tr>
  <tr><td bgcolor="#f2f2f2" style="background-color:#f2f2f2;border-radius:10px;padding:12px 16px;"><span style="font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:19px;color:#1a1a1a;">PRODUCT_LEAD&nbsp;</span><a href="PRODUCT_URL" style="font-family:Arial,Helvetica,sans-serif;font-size:13px;font-weight:700;color:#00C9DB;text-decoration:underline;">PRODUCT_CTA &#8594;</a></td></tr>
  <tr><td style="height:10px;line-height:10px;font-size:1px;">&nbsp;</td></tr>
  <tr><td><!-- social row --></td></tr>
</table>
```

WYRE AI product lead/CTA: `Your team is already using AI.` /
`Run it with Conduit` → `https://conduit.wyre.ai`.

WYRE Technology product lead/CTA: `Managed IT that stays ahead of the curve.` /
`Learn more` → Technology website.

Personal zone: after address, 10px spacer, then booking button; badge on
the right.

## Variant a · Quiet

Same personal zone with badge. Instead of the gray well: cyan text link
“See PRODUCT in action →”, then booking button; social below.

## Variant b · Dual-action

Personal zone **without** badge. Booking button beside a dark “Explore
PRODUCT →” button (`bgcolor="#0a0a0a"`, white label, cyan arrow). Then
slim bar image (`slimbarUrl`, 520×40) linking to the website, then social.

## Plain-text companion (optional)

```
NAME
TITLE
—
COMPANY — TAGLINE
OFFICE · EMAIL
WEBSITE
ADDRESS
PRODUCT line
—
BOOKING_LABEL: BOOKING_URL
```
