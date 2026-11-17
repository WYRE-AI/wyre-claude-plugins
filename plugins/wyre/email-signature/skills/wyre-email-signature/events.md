# Signature event lines

Optional promo banner above the personal block. Keep this file as the
editable event catalog so the skill stays current without a website.

## How to add or update

1. Add a row (or edit an existing one) in the table below.
2. Set `active` to `yes` while the banner should be the default when a
   user asks to “add the event” without naming dates.
3. After the event, set `active` to `no` or delete the row, then
   regenerate signatures without the event row.
4. For a one-off line the user dictates (“Meet us at …”), you do not need
   to edit this file — use their copy for that render only.

## Catalog

| id | active | label | dates | link | notes |
|---|---|---|---|---|---|
| channel-partners-orlando-2027 | yes | Meet us at Channel Partners, Orlando | Mar 2027 | (none unless user supplies) | Example promo used in worked HTML samples; update dates/link when confirmed |

Only one active default event at a time unless the user asks for a
specific `id`.

## Event row HTML

Place as the **first** row inside the outer 520px table, then an 12px
spacer, then the personal zone.

Without link:

```html
<tr><td bgcolor="#0a0a0a" style="background-color:#0a0a0a;border-radius:8px;padding:10px 14px;"><span style="font-family:Arial,Helvetica,sans-serif;font-size:13px;font-weight:700;line-height:18px;color:#ffffff;">LABEL</span><span style="font-family:Arial,Helvetica,sans-serif;font-size:12px;line-height:18px;color:#a3a3a3;">&nbsp;&middot;&nbsp;DATES</span></td></tr>
```

With link (https only):

```html
<tr><td bgcolor="#0a0a0a" style="background-color:#0a0a0a;border-radius:8px;padding:10px 14px;"><a href="LINK" style="font-family:Arial,Helvetica,sans-serif;font-size:13px;font-weight:700;line-height:18px;color:#00C9DB;text-decoration:underline;">LABEL</a><span style="font-family:Arial,Helvetica,sans-serif;font-size:12px;line-height:18px;color:#a3a3a3;">&nbsp;&middot;&nbsp;DATES</span></td></tr>
```

Escape label and dates. Omit the dates span when empty.
