# UI Specs: Menu Decision Engine
Addendum to the PRD + Specs doc — paste this alongside them when prompting Antigravity.

## 1. Visual Direction
**Vibe:** Warm, calm wellness — not clinical, not loud. This is a health app but it should feel like a supportive daily habit, not a hospital dashboard. Given the persona (gut health focus, avoiding stress around food), the UI should feel *reassuring*, not like another thing to be judged by.

**Data density:** Simple by default. Each screen leads with a calm, glanceable state (icons, checkmarks, a soft progress ring) — numbers (grams of protein, exact targets) are one tap away, not sitting on the surface by default. This avoids the app feeling like a strict tracker.

## 2. Color Palette
| Role | Color | Use |
|---|---|---|
| Background | Warm off-white / cream — `#FAF7F2` | Base background, cards |
| Primary | Sage green — `#5B7F63` | Headers, primary buttons, active states, nutrient progress rings |
| Accent | Soft terracotta — `#E08E6D` | Highlights, "needs attention" flags (e.g. protein gap), key CTAs |
| Text (primary) | Warm charcoal — `#3A3733` | Body text, headers |
| Text (secondary) | Muted taupe — `#8C8579` | Captions, secondary labels |
| Success/complete | Muted moss — `#7A9B76` | "Target met" states |
| Card surface | White with warm tint — `#FFFDF9` | Cards on top of the cream background |

Rationale: sage + terracotta reads as "wellness" without being sterile-clinical (avoids cold blues/greys common in medical apps) and without being overly vibrant/gamified (avoids feeling like a fitness-competition app).

## 3. Typography
- Font: a rounded, friendly sans-serif — e.g. **Inter** or **Nunito Sans** (both free, widely available, work well on mobile)
- Headers: medium weight, generous line height
- Body: regular weight, 16px minimum for readability
- Numbers (protein grams, etc.): can use a slightly heavier weight when shown, so they feel confident rather than clinical-small

## 4. Component Style
- Rounded corners throughout (12-16px radius) — cards, buttons, input fields
- Soft drop shadows, not hard borders — cards should feel like they're gently sitting on the background, not boxed in
- Icons: simple, single-line/outline style (not filled/bold) for meal slots, nutrients, supplements — keeps things calm
- Progress: use a soft circular ring (not a stark bar chart) for protein/nutrient progress — rings read as "how you're doing today," bars read as "performance metric"

## 5. Screen-by-Screen Notes

**Onboarding — Add Member**
- One question per screen-section, generous whitespace, not a long form dumped at once
- Progress dots at top (not a percentage bar) so it feels like a short conversation, not a form
- Cuisine step: primary cuisine as a single-select chip row, occasional cuisines as a multi-select chip row below it — visually light, no sense of "getting this wrong"

**Nutrient Targets screen**
- Each member gets a card with their name, a soft ring showing today's protein target as a calm circle (not filled yet — this is the *target*, not progress)
- Rationale text in muted secondary color, small, not alarmist
- Edit icon (pencil) subtle, top-right of the protein number

**Today's Menu screen (core screen)**
- Five cards, one per meal slot, already populated when the screen opens on a normal day — no loading spinner, no "generating your menu..." state; that would contradict the whole point of generating ahead of time. A brief loading state is only acceptable in the rare fallback case (first open right after onboarding).
- Each card shows: item name in primary text, small tag icons for which member(s) it covers, and a collapsed "see details" for protein/fibre numbers
- A small cuisine tag (e.g. "South Indian") on any item that isn't the household's primary cuisine — a quiet visual cue for why today looks a bit different, not a label that needs explaining
- A single soft progress ring per member at the top of the screen (tap to expand into the numeric breakdown) — this is the "simple by default, tap for numbers" pattern
- "Regenerate" as a subtle icon button (circular arrow), not a heavy primary button — this is a secondary action, not the main flow
- "Swap substitute" as a small icon action (two arrows) next to any item with a known substitute (e.g. paneer↔tofu) — one tap, no form
- Gap-filler sides (added purely to close a protein/fibre shortfall) render as a smaller, visually lighter card than a real menu item — a thin dashed border or reduced-opacity card works — with a small label like "added to reach today's target," so it never reads as a planned dish
- Flags for nutrient gaps: a small terracotta dot/badge, not a red alarm — reassuring, not alarming. Tapping it expands into a plain-language reason (e.g. "12g short — no more substitutes available today"), not just a number

**Brand-looked-up values**
- A small, low-key "✓ looks right?" affordance near a branded item's protein number (not a modal, not blocking) — a tiny checkmark and source link, easy to ignore, easy to tap if curious

**Supplement skip toggle**
- Small toggle chip per supplement, terracotta when "skipped today," sage/neutral when "as usual" — low visual weight, this is a quick daily adjustment not a big decision

**Mid-morning slot**
- Same card style as the other four meal slots, not visually demoted — a small fruit icon distinguishes it, and it defaults to a seasonal suggestion without the user needing to ask

**Favorites screen**
- Simple list, populated automatically as dishes get favorited — no empty-state "add something" pressure, since it fills up naturally through use
- Each entry: a small filled-heart icon, the dish name, last-served date, and a one-tap "make this again" for a chosen slot

## 6. What NOT to do
- No red/alarm colors for "you're short on protein" — use terracotta (attention, not alarm)
- No dense nutrition-label-style tables as the default view — that's available on tap, not the landing state
- No gamified streaks/badges — this isn't a fitness-competition app, it's a calm daily planning tool
