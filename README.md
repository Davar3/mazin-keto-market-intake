# فۆرمی مارکێتی نوێ - Mandub market intake

A single, self-contained HTML page the mandubs (field reps) fill in while standing in a
shop. They tap one **کۆپیکردن / Copy** button and paste a structured text block into
WhatsApp/Telegram to the owner, who then creates the market in the real B2B app.

This is an **interim tool** for the period before the mandubs are trained on the app
itself. It has no backend, no build step, no database - it only produces text.

- `index.html` - the whole thing. One file. Nothing else is required.

## How to open it

- **On a phone:** open the GitHub Pages URL once; after that it works with no signal
  (the browser caches the page). Tell the mandub to "Add to Home screen".
- **On a computer:** double-click `index.html`, or drag it into a browser tab.

Note: `navigator.geolocation` only works on `https://` (or `localhost`). Opened as a
local `file://` the GPS button will report that location is unavailable and the mandub
falls back to the manual Lat/Lng fields - the form still works and still submits.

## Fields

Derived 1:1 from `createMarketSchema` in
`src/features/markets/actions.ts` of the Keto-Market-system repo.

| Key in the block | Column / target | Required |
|---|---|---|
| `MANDUB`, `MANDUB_NAME` | `market_mandubs.mandub_id` (who to assign) - `nzar`, `hiwa`, `drmazin`, or `other` with a typed name | yes |
| `MARKET_NAME` | `markets.name` | yes |
| `MANAGER_PHONE` | `markets.manager_phone` | yes |
| `GOVERNORATE` | `markets.governorate_id` (by name) | yes |
| `DISTRICT` | `markets.district_id` (by name) | yes |
| `SUBDISTRICT` | `markets.subdistrict_id` (by name) | no |
| `CUSTOMER_PHONE` | `markets.customer_phone` | no |
| `CUSTOMER_PHONE_2` | `markets.customer_phone_2` | no |
| `ADDRESS` | `markets.address` | no |
| `OPEN_FROM`, `OPEN_TO` | `markets.open_from` / `open_to` (HH:MM) | no |
| `PAYMENT_METHOD` | `markets.payment_method` - `cod` / `credit` / `consignment` | yes (defaults to `cod`) |
| `MAPS_URL` | not a DB column - the Google Maps link the mandub pasted, for you to open | no |
| `LAT`, `LNG` | `markets.lat` / `markets.lng` | no (but both or neither) |
| `LOCATION_SOURCE`, `GPS_ACCURACY_M` | capture metadata, not DB columns - `gps`, `maps_link`, `maps_link_only`, `manual` or `none` | - |

System-owned columns (`id`, `created_by`, `current_debt_iqd`, `active`, `owner_id`,
`debt_cap_iqd`, `discount_pct`, `doc_locales`, `slug`, `is_own_branch`, the public-site
metadata) are deliberately **not** on the form.

**No owner login is collected.** The owner-account fields were removed at the owner's
request: creating a market login is a decision made in the app, not something to settle
over WhatsApp. The app generates any password at creation time and
shows it once - nothing that becomes a credential should travel through WhatsApp.

Governorate and district are picked from dropdowns carrying the exact names seeded in
the `governorates` / `districts` tables, so the owner can match them to their UUIDs
without guessing spelling. Subdistrict is free text (only ~1/4 of districts are seeded).

## The copy-paste block

Stable ASCII keys, one field per line, human label in parentheses. `-` means the mandub
left it empty. A missing optional value is still emitted, so the shape never changes.

```
MAZIN KETO - MARKET INTAKE
INTAKE_V: 2
SUBMITTED_AT: 2026-08-23 18:14 (Asia/Baghdad)
MANDUB: drmazin
MANDUB_NAME (ناوی مەندووب): د. مازن
---
MARKET_NAME (ناوی مارکێت): مارکێتی ڕۆژهەڵات
MANAGER_PHONE (ژمارەی بەڕێوەبەر): 07501234567
CUSTOMER_PHONE (ژمارەی کڕیاران): 07701234567
CUSTOMER_PHONE_2 (ژمارەی دووەمی کڕیاران): -
ADDRESS (ناونیشان): مەخموور، شەقامی 60 مەتری، بەرامبەر مزگەوتی گەورە
GOVERNORATE (پارێزگا): هەولێر
DISTRICT (قەزا): مەخموور
SUBDISTRICT (ناحیە): قەرەچۆغ
OPEN_FROM (کاتی کردنەوە): 09:30
OPEN_TO (کاتی داخستن): 23:00
PAYMENT_METHOD: credit
LAT (Lat): 36.190000
LNG (Lng): 44.010000
MAPS_URL (لینکی گووگڵ ماپ): -
LOCATION_SOURCE: gps
GPS_ACCURACY_M: 12
END
```

Parse it with one regex per line:

```js
const fields = {};
for (const line of text.split("\n")) {
  const m = line.match(/^([A-Z0-9_]+)(?:\s*\(([^)]*)\))?\s*:\s*(.*)$/);
  if (m) fields[m[1]] = m[3] === "-" ? null : m[3];
}
```

### Location, and why the coordinate boxes are hidden

Three ways in, in the order the form offers them:

1. **"شوێنی من بەکاربهێنە"** - GPS, for when the mandub is standing at the shop.
   Best case: exact pin plus an accuracy figure.
2. **A pasted Google Maps link** - the realistic fallback. The form pulls
   coordinates out of any link that carries them (`/@lat,lng`, `?q=`, `!3d!4d`,
   `?query=`), and accepts a bare `36.19, 44.01` too. It also strips the
   sentence the share sheet wraps around the URL.
3. **Typed numbers** - collapsed behind a disclosure, because nobody types
   `36.190000` correctly on a phone in a shop doorway. It exists only for the
   case where GPS is refused *and* there is no link.

**Short links carry no coordinates.** `maps.app.goo.gl/...` - what the Android
share sheet produces - is an opaque redirect; resolving it needs a network
request, and this page makes none. Those are accepted and passed through as
`MAPS_URL` with `LOCATION_SOURCE: maps_link_only`, for you to open and read the
pin from. That is a deliberate trade: keeping the page offline-capable is worth
more than auto-resolving a link you can click.

A GPS fix clears any pasted link, so the two can never disagree.

`INTAKE_V` is the schema version of the block. **Bump it if you change or remove a key**,
so an old page still sitting on a mandub's phone can be told apart from a new one.

## Behaviour worth knowing

- Phones are normalised: Arabic-Indic digits become Latin, spaces are stripped, a
  missing leading `0` is added. Validation matches the app's zod rule (`07` + 9 digits).
- Everything typed is saved to `localStorage` on every keystroke and restored on reload,
  so a phone call mid-entry loses nothing. "سڕینەوەی فۆرمەکە" needs two taps to wipe it.
- The Copy button stays disabled until every required field is valid; the list of what's
  still missing sits right above it, and tapping the disabled button jumps to the first
  problem field.
- Copy uses `navigator.clipboard.writeText` and falls back to `document.execCommand`
  for older Android WebViews. The same text is always visible in the read-only box so it
  can be long-pressed and copied by hand.
- No external requests of any kind - no CDN, no fonts, no analytics. Verified.

## Deploy to GitHub Pages

Nothing is initialised yet - no git repo, no remote. When you've decided the repo name
and whether it's public or private:

```bash
cd /Users/davar/Desktop/Work/Dr.Mazin/mandub-market-intake
git init -b main
git add index.html README.md
git commit -m "Mandub market intake form"
gh repo create <REPO-NAME> --private --source=. --push
```

Then either:

- **Settings → Pages → Source: Deploy from a branch → `main` / `/ (root)`**, or
- `gh api -X POST repos/<OWNER>/<REPO>/pages -f "source[branch]=main" -f "source[path]=/"`

The page lands at `https://<owner>.github.io/<repo>/` (GitHub Pages serves `index.html`
at the root). Give that link to the mandubs.

**Pages on a private repo needs a paid GitHub plan.** On the free plan, publishing means
a public repo - which is fine here: the page holds no secrets, no keys and no customer
data. If you'd rather it not be public at all, host the same file anywhere else static
(it is one file with no dependencies) or just send them the file itself.

Since the mandubs' phones cache the page, after any edit tell them to pull-to-refresh
once, or add a cache-buster to the link (`?v=2`).
