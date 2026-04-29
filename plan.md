# Beverage Tracker App — Project Brainstorm

## App Name Ideas
- **Sipped** — clean, minimal, shareable
- **Poured** — action-oriented
- **TapLog** — hints at beer culture + logging
- **BrewPassport** — leans into the travel/map angle
- **On Tap** — casual and familiar

---

## Tech Stack

| Layer | Recommendation | Notes |
|---|---|---|
| Frontend | React or Vanilla HTML/CSS/JS | Vanilla keeps it lightweight; React better for complex state |
| Backend | Flask | Already familiar; handles API routes cleanly |
| Database | Supabase (PostgreSQL) | Auth, storage, and DB in one platform |
| Maps | D3.js + TopoJSON/GeoJSON | Gold standard for choropleth world maps |
| Charts | Recharts or Chart.js | Time tracker views |
| AI | Claude API | Analogy generation feature |
| File Storage | Supabase Storage | Photo upload buckets |

---

## Feature 1 — Drink Logger

### Overview
The core input form. Every other feature (maps, BAC, analytics, photos) is powered by data entered here.

### Brand Database
Seed a `brands` table with 200–300 popular brands. Minimum columns:
- `name`
- `origin_country`
- `typical_abv`
- `drink_type`

When a user selects a brand (e.g. Corona), ABV, origin country, and drink type auto-populate. Users can override ABV and add unlisted brands, which enrich the database over time.

### Standard Drinks Calculation
Compute live as the user fills in ABV and volume.

**Formula (Canadian standard, 13.6g alcohol):**
```
std_drinks = (volume_ml × (abv / 100) × 0.789) / 13.6
```

This becomes the universal unit across the app — time tracker, BAC calculator, and AI analogies all operate in standard drinks.

### Two Entry Modes
1. **Detailed mode** — full form with brand, ABV, volume, location, timestamp
2. **Quick log** — floating `+` button, tap Beer/Wine/Spirit, confirm in 2 seconds. Defaults to one standard drink; user edits details later

### Volume Presets
| Label | Volume |
|---|---|
| Bottle | 341 mL |
| Can | 355 mL |
| Pint | 473 mL |
| Wine glass | 150 mL |
| Shot | 44 mL |

### Location
- Use `navigator.geolocation` on mobile for GPS-based country auto-detection
- Reverse geocode via `nominatim.openstreetmap.org` (free)
- Pre-fill the country dropdown, but keep it editable for retroactive entries

### Validation Rules
- ABV capped at 100%; warn above 60%
- Volume capped at ~2000 mL with warning
- If `std_drinks > 4` for a single entry, show a subtle yellow flag (not a gate)
- Retroactive timestamps allowed; future timestamps blocked

### Edit & Delete
- Swipe-to-delete on mobile; `...` menu on desktop
- RLS policy enforces `user_id` equality on all mutations

### Database Schema — `drinks` table
```sql
create table drinks (
  id               uuid primary key default gen_random_uuid(),
  user_id          uuid references auth.users not null,
  brand_id         uuid references brands(id),
  brand_name       text not null,
  drink_type       text not null,
  abv              numeric(4,1),
  volume_ml        numeric(6,1),
  std_drinks       numeric(4,2) generated always as
                     ((volume_ml * (abv / 100) * 0.789) / 13.6) stored,
  country_consumed text,
  origin_country   text,
  logged_at        timestamptz not null default now(),
  notes            text
);
```

`std_drinks` is a Postgres generated column — computed automatically, never needs recalculation in app code.

---

## Feature 2 — Time Tracker

### Views
- Week / Month / Year / All-time toggle
- Bar or area charts (Recharts or Chart.js)

### Metrics per Period
- Total standard drinks
- Total volume consumed
- Most common drink type
- Favourite brand

### Gamification
- Longest sober streak
- Most drinks in a single week
- First drink in a new country

### Unit Settings
Standard drink sizes vary by country. Make this a user preference:
| Country | Standard drink |
|---|---|
| Canada / USA | 13.6g alcohol |
| UK | 10g alcohol |
| Australia | 10g alcohol |

---

## Feature 3 — Location Map (Where You Drank)

### How It Works
- Data source: `country_consumed` field on each drink entry
- Aggregate drink count per country per user
- Render as D3 choropleth over a GeoJSON world map
- More drinks in a country = deeper colour fill

### Colour Scale
Use D3's `scaleSequential` mapped to the count range. Apply a log transform so single-visit countries still show a visible tint:
```js
const colorScale = d3.scaleSequential()
  .domain([0, d3.max(counts)])
  .interpolator(d3.interpolate('#E1F5EE', '#04342C')); // teal ramp
```

---

## Feature 4 — Origin Map (Where Drinks Are From)

### How It Works
- Data source: `origin_country` field on each drink entry (auto-populated from brand database)
- Aggregate drink count per origin country
- Same D3 choropleth as location map, different colour ramp (e.g. purple: `#EEEDFE → #26215C`)

### Example
If a user drank Modelo, Corona, and Pacifico (all Mexico), Mexico accumulates a high count and fills in deeply even if the user has never visited Mexico.

---

## Feature 5 — Combined World Map

### How It Works
Overlays the location and origin layers on a single map with blended colour:
- **Location layer** — teal ramp, indicates where user was
- **Origin layer** — purple ramp, indicates where drinks are from
- **Overlap** — blend or mixed shade where both are high

### UI
A 3-tab toggle control:
- **Where I've been** (location only)
- **What I've drunk** (origin only)
- **Combined** (both layers)

### Map–Photo Connection
When a user hovers or taps a country, a sidebar/tooltip shows:
- Drink count and most common brand
- 3 most recent photos from that country
- Query: `select dp.* from drink_photos dp join drinks d on d.id = dp.drink_id where d.country_consumed = $1 and dp.user_id = $2 limit 3`

---

## Feature 6 — BAC Calculator

### Formula
Uses the **Widmark formula**:
```
BAC = (A / (r × W)) − (β × T)
```
- `A` = grams of alcohol consumed
- `r` = 0.68 (male) or 0.55 (female) — body water constant
- `W` = body weight in grams
- `β` = 0.015 — alcohol elimination rate per hour
- `T` = hours elapsed since first drink

One Canadian standard drink ≈ 13.6g alcohol (make this configurable per user's country setting).

### Timeline Input
User inputs drinks with timestamps at 30-minute increments:
```
1 beer at 10:00 PM
2 beers at 10:30 PM
1 wine at 11:00 PM
```

Render a live BAC curve (line chart) that:
- Rises as each drink is added
- Falls at 0.015 per hour after consumption
- Shows a labelled horizontal rule at the legal limit (0.08 in Canada/USA)
- Displays projected "back to zero" time

### User Inputs
- Body weight (lbs or kg, configurable)
- Height (for context/future use)
- Biological sex (for the `r` constant)
- Drink entries with timestamps

### Disclaimer
Include a clear disclaimer: *BAC calculator is an estimate for educational purposes only. Results do not account for food, medications, individual metabolism, or other factors. Do not use this to make decisions about driving.*

---

## Feature 7 — AI Analogies (Claude API)

### Overview
After a session or viewing a time period, the app calls the Claude API to generate creative, funny comparisons for how much the user drank.

### API Call Structure
```javascript
const response = await fetch("https://api.anthropic.com/v1/messages", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1000,
    system: `You generate creative, funny, single-sentence analogies comparing 
             an amount of alcohol consumed to something relatable in the real world. 
             Be concrete and use real numbers. Return a JSON array of 4 analogies 
             with keys: category (string), analogy (string). 
             Categories: volume, fuel, calories, weight. 
             Return JSON only, no markdown.`,
    messages: [{
      role: "user",
      content: `The user drank ${stdDrinks} standard drinks totalling ${volumeMl} mL 
                of alcohol this week. Generate 4 analogies.`
    }]
  })
});
```

### Analogy Categories
| Category | Example output |
|---|---|
| Volume | "Your 2.4L of beer would fill 6 standard water balloons" |
| Fuel | "The ethanol you drank could fuel a car for approximately 0.8 km" |
| Calories | "That's roughly equivalent to 4.2 Big Macs in caloric content" |
| Weight | "The pure alcohol you consumed weighs about the same as a large apple" |

### Display
Render as a shareable "stats card" — styled summary with each analogy as a row. This is the most screenshot-worthy feature in the app.

---

## Feature 8 — Photo Attachment

### Overview
Users can attach up to 5 photos per drink log entry. Photos are tied to the drink's location and surface on the world map when that country is tapped.

### Storage Architecture
Two Supabase Storage buckets:
- `drink-photos` — private; only accessible to the owning user
- `drink-photos-public` — public read; for photos marked visible on profile

**Storage path convention:** `{user_id}/{drink_id}/{uuid}.jpg`

This namespacing makes it easy to:
- Delete all photos for a drink in one list + delete operation
- Delete all photos for a user on account deletion

### Upload Flow
1. Client-side validation: MIME type, file size < 10 MB, count ≤ 5
2. Client-side compression via Canvas API: resize to max 1200px long edge (~500 KB result)
3. Flask `/log-drink` endpoint inserts the `drinks` row, returns `drink_id`
4. Client uploads directly to Supabase Storage (bypasses Flask server — no large files through API)
5. Insert rows into `drink_photos` table with storage paths

### HEIC Support
iPhones save as HEIC by default. Two options:
- `heic2any` npm package for client-side conversion before upload
- Supabase image transformation: append `?format=webp` to any Storage URL for automatic transcoding (easier)

### Database Schema — `drink_photos` table
```sql
create table drink_photos (
  id           uuid primary key default gen_random_uuid(),
  drink_id     uuid references drinks(id) on delete cascade,
  user_id      uuid references auth.users not null,
  storage_path text not null,
  public_url   text,
  caption      text,
  venue_name   text,
  venue_tags   text[],
  is_public    boolean default false,
  uploaded_at  timestamptz default now()
);
```

`on delete cascade` on `drink_id` removes photo rows when a drink is deleted (still need to manually delete files from Storage bucket in app logic).

### Venue Tagging
`venue_tags` is a Postgres array. Query example — all photos tagged Festival:
```sql
SELECT * FROM drink_photos WHERE 'Festival' = ANY(venue_tags);
```

Preset tags: Bar, Restaurant, House party, Festival, Brewery, Rooftop, Patio.

Venue name is free text for v1. Can upgrade to Google Places or Foursquare autocomplete later.

### Privacy
- `is_public = false` → stored in private bucket, only visible to owner
- `is_public = true` → stored in public bucket, visible on public profile
- Add `is_deleted` soft-delete flag instead of hard deletes for recovery

---

## Database Overview

### Core Tables
| Table | Purpose |
|---|---|
| `auth.users` | Supabase built-in auth |
| `brands` | Seeded brand lookup (name, origin, ABV, type) |
| `drinks` | Core drink log entries |
| `drink_photos` | Photos attached to drink entries |

### Key Relationships
```
auth.users
    └── drinks (user_id)
            ├── brands (brand_id)
            └── drink_photos (drink_id)
```

### Row Level Security
Every table should have RLS enabled with a policy enforcing `user_id = auth.uid()` on all select, insert, update, and delete operations.

---

## Portfolio & Nice-to-Haves

| Feature | Notes |
|---|---|
| Google OAuth | Via Supabase Auth — one-click setup |
| Public profile | `app.com/username` — shows maps, stats, public photos |
| Shareable stats card | Screenshot-friendly AI analogies summary |
| Friends / compare | Side-by-side map comparison between two users |
| Mobile-first design | Maps + photo feed are the demo centrepiece |
| Progressive Web App | Add to home screen, offline logging queue |

---

## Development Phases

### Phase 1 — Core
- Supabase project setup (auth, DB schema, RLS)
- Brand database seed (~200 entries)
- Drink logger form with autocomplete
- Basic time tracker (week/month views)

### Phase 2 — Maps
- GeoJSON world map base with D3
- Location choropleth
- Origin choropleth
- Combined toggle view

### Phase 3 — Advanced Features
- BAC calculator with Widmark formula + timeline chart
- AI analogies via Claude API
- Photo upload + compression + venue tagging

### Phase 4 — Polish
- Public profiles + shareable links
- Mobile PWA
- Map–photo connection (country hover → photo gallery)
- Friends / compare feature