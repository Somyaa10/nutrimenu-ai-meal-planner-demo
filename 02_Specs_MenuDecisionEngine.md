# Specs: Menu Decision Engine
Companion to the PRD. This is what you'd hand to a builder (or paste into Bolt.new/Lovable) to actually build the MVP.

Platform target: **mobile-first responsive web app** (see plan doc for why, and for the tool recommendation).

**Core UX principle (from the PRD):** the app should require zero daily verification. Every default state, every piece of copy, and every confirmation affordance below should be built so it's genuinely ignorable on a normal day — editing exists for exceptions and special days, not as a habit the app trains the user into.

---

## 1. Data Model

**Household**
- id, name, location_city, location_state, climate_zone_id, primary_cuisine (e.g. "North Indian", "Bengali", "South Indian", "Gujarati", "Punjabi", "Maharashtrian" — the household's native/default cuisine, independent of where they currently live), occasional_cuisines: [string] (e.g. ["South Indian", "Italian", "Chinese"] — cuisines they enjoy sometimes, not their everyday default), created_at

**ClimateZone** (reference table, seeded — not user-editable)
- id, name (e.g. "Deccan Plateau", "North Indian Plains", "Western Coastal", "Eastern/Northeast", "Himalayan/Hill")
- month_to_season: jsonb mapping each month to a season label (summer / monsoon / winter / spring) — this is what differs by zone
- Used to resolve "what season is it right now" for a household, from their location + today's date

**Member** (belongs to a Household)
- id, household_id, name, age, weight_kg, height_cm, activity_level (sedentary / light / moderate / athlete)
- diet_type (veg / non_veg / vegan / eggetarian / pescatarian)
- excluded_foods: [string] — hard no's (e.g. "paneer", "milk")
- conditional_foods: [{food: string, note: string}] — e.g. {food: "eggs", note: "not daily"}
- goals: [string] — free text, e.g. "improve gut health"
- health_notes: [string] — free text, self-reported (e.g. "sensitive digestion")
- preferred_grains: [string] (e.g. "rice", "roti", "millet") — optional
- dal_likes: [string] — optional
- spice_level (mild / medium / spicy) — optional
- cooking_style_notes: string — free text, optional (e.g. "avoid deep-fried")

**Supplement** (belongs to a Member)
- id, member_id, name, usual_frequency, skip_condition (free text, e.g. "when bloated")

**BloodworkEntry** (belongs to a Member)
- id, member_id, marker (e.g. "iron", "vitamin D", "B12"), value, unit, date, flagged_low (bool)

**NutrientTarget** (belongs to a Member, one active row per nutrient)
- id, member_id, nutrient (protein / fibre / iron / calcium / vitamin_d / b12 / etc.)
- baseline_value, ai_adjusted_value, user_override_value (nullable), unit, rationale (text — why the AI landed here), last_updated

**MealSlot** (enum: breakfast / mid_morning_snack / lunch / evening_snack / dinner)

**MenuDay**
- id, household_id, date, generated_at, generation_status (pending / ready / needs_attention — "needs_attention" only when a shortfall couldn't be auto-resolved, per 7.4/7.5)

**MenuItem** (belongs to a MenuDay + MealSlot)
- id, menu_day_id, slot, name, applicable_member_ids: [id], quantity_note (e.g. "for 2"), food_item_ids: [id] (links to the FoodItem(s) making up this item, for calculated nutrient totals), calculated_protein_g, calculated_fibre_g, is_gap_filler (bool — true for top-up sides added purely to close a shortfall), is_side (bool — true for an optional curd/yoghurt/salad-type add-on attached to a main item), generated_dish_id (nullable, links back to the GeneratedDish log entry it came from), source (ai_suggested / user_added / user_edited / gap_filler)

**MealHistory** (derived/log — used for variety checks)
- menu_item names + dates, queried for "last 5-7 days" when generating new suggestions

**FoodItem** (built-in staple ingredient database — ships pre-seeded, not something the household builds)
- id, name (e.g. "Akshayakalpa high-protein paneer"), brand (nullable, for generic staples like "toor dal"), prep_state (raw / cooked / boiled / fried), protein_per_100g, fibre_per_100g, other_nutrients (jsonb, optional — iron/calcium/vitamins where known), season_tags: [string] (e.g. ["monsoon"], empty if available year-round), is_builtin (bool), added_by (system / user), source_url (nullable — the page the values were pulled from, for branded items), verified_by_user (bool), last_verified_at (nullable)
- Seed this table with ~100-150 common staples (dals, rice/grain varieties, common vegetables, paneer/tofu/soy variants, eggs, seasonal fruit) with generic per-100g values across their common prep states, plus a short pre-seeded list of popular Indian brands for common categories (paneer, curd/yoghurt, tofu, protein powder, milk) so many households won't need to look anything up at all

**Substitution** (mapping between FoodItems)
- id, food_item_id, substitute_food_item_id, note (e.g. "1:1 by weight, similar texture in most dishes")
- Used to re-suggest a dish with an equivalent item (e.g. paneer → tofu) for a member who excludes the original

**GeneratedDish** (a log of AI-composed dishes — this is what builds a personal history organically, not a user-maintained recipe collection)
- id, name (e.g. "moong dal tadka"), food_item_ids: [id] with per-item grams (ingredient breakdown as generated), suitable_slots: [MealSlot], season_tags: [string], is_favorited (bool), household_id, generated_at
- Favorited dishes are weighted higher in future generation for that household; this table is populated automatically as the AI composes dishes, never something the user has to pre-fill

---

## 2. Screens & Flows

### 2.1 Onboarding
1. Create household → name it, set location (city + state) → mapped to a climate zone in the background, no jargon shown to the user
2. Set cuisine preferences → primary/native cuisine (e.g. "North Indian"), plus an optional multi-select of cuisines they enjoy occasionally (e.g. "South Indian," "Chinese," "Italian") — a couple of taps, not a form
3. Add Member #1 → form: name, age, weight, height, activity level, diet type, exclusions, conditional foods, goals, health notes
4. Quick food preferences for Member #1 (optional, skippable, 3-4 taps): preferred grains, dal likes, spice level
5. Optional: "Do you use specific brands for paneer/curd/tofu/protein powder?" — pick from a short pre-seeded list, or search a brand not listed (see 2.2a) — entirely skippable, generic staple values are used until this is filled in
6. Add supplements for Member #1 (optional, can skip)
7. Add bloodwork highlights for Member #1 (optional, can skip — a few key flagged values, manual entry)
8. Repeat for Member #2 (or "add another member")
9. → Nutrient Target screen (auto-generated, see 2.2)

### 2.2a Brand Selection / Search (triggered from onboarding step 4, or later from settings)
- Short list of pre-seeded popular brands per category (paneer, curd/yoghurt, tofu, protein powder, milk) — one tap to select, no lookup needed since these ship pre-verified
- "Search a different brand" → user types brand + product name → triggers the Brand Nutrient Lookup call (3.0) → result is saved and usable immediately, shown with source link and an optional "looks right?" confirm — the household isn't blocked on reviewing it
- Low-confidence lookups are visually flagged as worth double-checking, but still usable right away

### 2.2 Nutrient Targets screen (per member)
- Shows each nutrient (protein prominent, others as directional "focus" chips)
- Protein: baseline (1g/kg) shown, AI-adjusted value shown with a one-line rationale ("Adjusted to 1.4g/kg for athletic activity level"), editable field for user override
- Fibre/iron/calcium/vitamins: shown as qualitative focus flags (e.g. "Iron: focus area" if bloodwork flagged low), not gram-exact
- Save → this becomes the standing target until edited again

### 2.3 Today's Menu screen (core screen)
- **No "Generate" primary action.** The day's menu is already generated by the time the household opens the app (see 3.4 for the scheduling mechanism) — this screen just displays what's ready. There is no empty/loading state on a normal day.
- Date header, 5 meal slot sections (breakfast, mid-morning, lunch, evening snack, dinner) — mid-morning defaults to a seasonal fruit suggestion
- Each slot: AI-suggested item(s), tagged with which member(s) it covers, a quantity note, and calculated (not estimated) protein/fibre pulled from the FoodItem it links to
- Any slot can have an optional **side** attached (curd/yoghurt/salad) — shown as a small secondary line under the main item, with its own quick "add a side" tap target
- Actions per item: Accept (default state) / Regenerate / Replace manually (free text) / **Swap substitute** (one tap, shows mapped substitutes like paneer↔tofu, re-tagged to the member who needed the swap) / **Favorite** (heart icon, one tap)
- Any **gap-filler side** the system adds to close a shortfall renders visually distinct from a full menu item (smaller, tagged "added to reach today's protein") so it doesn't read as a planned dish
- Bottom summary bar: per-member protein progress (e.g. "Ana: 48g / 55g target"), with a flag icon if short — tapping it shows *why* (which meal slots are light) and what was tried to resolve it
- "Skip supplement today" toggle per member, visible if they have a supplement with a skip condition — re-triggers a protein gap check when toggled, which can trigger a new gap-filler suggestion

### 2.4 History / Variety check (lightweight, can be same screen)
- Small strip showing last 5 days' dinners, so the user can eyeball repeats even before regenerating

### 2.5 Favorites screen
- A simple list of dishes the household has favorited over time — populated automatically as they use the app, nothing to set up upfront
- Favorited dishes are weighted higher in future generation; the user can also manually pick "make this again" from here for a specific slot

---

## 3. AI Logic — What the model needs to do

There are five distinct calls/jobs. Keep them separate (different prompts, different jobs):

### 3.0 Brand Nutrient Lookup (runs when a household selects/searches a brand not already in the database)
**Input:** brand name + product name as typed or picked by the user (e.g. "Akshayakalpa high protein paneer")
**Task:** Search the web for the brand's official product page or a major retailer listing (Blinkit/Amazon/BigBasket) that shows the printed nutrition label, fetch it, and extract protein/fibre/other nutrient values per 100g. This must be a real search-and-extract step, not the model answering from its own training knowledge — label numbers are exactly the kind of fact models get subtly wrong from memory alone.
**Output (structured):** `{name, brand, protein_per_100g, fibre_per_100g, other_nutrients, source_url, confidence: "high"|"low"}` — "low" confidence (e.g. conflicting sources, no clear label found) should surface a more visible confirm-or-edit prompt to the user rather than presenting it as certain
**UX requirement:** the looked-up value is saved and used immediately — no blocking confirmation step before it's usable. Show it with the source link and a lightweight "looks right? ✓" affordance the user can engage with whenever they want, but never make menu generation wait on it. Low-confidence results should be visually flagged as worth a second look, without blocking their use either.

### 3.1 Nutrient Target Generation
**Input:** member's weight, activity level, goals, health notes, bloodwork flags
**Task:** Calculate protein target starting at 1g/kg, adjusting upward toward a 2g/kg ceiling based on activity level and goals (e.g. athlete → higher end; sedentary + weight goals → lower-mid end). Flag fibre/iron/calcium/vitamin focus areas based on health notes and any flagged-low bloodwork values. Always return a one-line rationale.
**Output (structured):** `{protein_g_per_kg, protein_target_g, rationale, focus_nutrients: [{nutrient, reason}]}`
**Guardrail:** This is a wellness planning estimate, not medical advice — the UI should say so once, and the AI should never claim to diagnose or treat a condition; it just reasons about food-planning targets.

### 3.2 Menu Generation
**Input:** all members' current targets + exclusions + conditional foods + food format preferences (grains, dal likes, spice level) + the household's primary and occasional cuisines + supplement skip states (today) + last 5-7 days of menu history + meal slot being generated + the household's current season (resolved from their climate zone + today's date) + the FoodItem staple database (for real protein/fibre values) + past favorited GeneratedDishes for this household + the Substitution map
**Task:** **Compose the specific dish live** for the slot — decide the exact variant (e.g. "moong dal tadka," not just "dal") using general knowledge of common Indian home-cooking patterns, respecting every member's hard exclusions and leaning toward their format preferences and favorited dishes. **Weight dish selection toward the household's primary cuisine most days, but genuinely mix in their occasional cuisines rather than defaulting to primary-cuisine-only** — a North Indian household that listed South Indian and Italian as occasional should actually see idli or a simple pasta show up sometimes, not just North Indian dishes indefinitely; roughly bias toward primary cuisine on most days with occasional-cuisine dishes appearing periodically, not as a rare fluke. Break the dish into a realistic ingredient list with quantities (e.g. "150g paneer, 1 cup rice, 1 tsp oil"), then calculate protein/fibre by mapping each ingredient onto the FoodItem staple database — don't estimate total nutrition freeform. Favor in-season produce. Bias protein-heavier compositions toward breakfast/lunch slots and lighter-but-nonzero toward dinner. For the mid-morning slot, default to a seasonal fruit unless the user has set a different preference. Avoid repeating what's been served in the last several days unless options are genuinely limited. Optionally suggest a side (curd/yoghurt/salad) alongside the main item where it fits naturally.
**Output (structured):** `{items: [{name, cuisine, ingredients: [{food_item_id, grams}], applicable_members, calculated_protein_g, calculated_fibre_g, quantity_note, side: {name, food_item_ids} | null}], reasoning}`
**Note:** if an ingredient the AI wants to use isn't yet in the FoodItem database, it should substitute the closest matching staple already in the database rather than inventing a nutrient value — this keeps every number traceable to a real source.

### 3.3 Gap Resolution (runs after each day's menu is generated)
**Input:** the day's running protein/fibre totals per member vs. their targets, the Substitution map, the FoodItem library
**Task:** If any member is short, first try substituting an existing menu item with a mapped equivalent that raises their total (e.g. swap a lower-protein item for a paneer/tofu-based one they can eat) without changing the dish materially for the rest of the household. If substitution alone can't close the gap, suggest a top-up gap-filler side (e.g. "50g paneer, salt & pepper") calculated to close the remaining gap. If neither resolves it, return a clear shortfall reason instead of silently leaving the day short.
**Output (structured):** `{resolved: bool, actions_taken: [{type: "substitution"|"gap_filler", detail}], remaining_shortfall: [{member, nutrient, amount}] | null}`

### 3.4 Scheduled Generation (the mechanism behind "no Generate button")
**Trigger:** a backend scheduled job (e.g. runs once daily, such as late evening for the next day, or early morning for the same day — pick one consistent time) rather than anything the user initiates
**Task:** run 3.2 (Menu Generation) for all five slots, then 3.3 (Gap Resolution), for every household, so a complete `MenuDay` with `generation_status: ready` (or `needs_attention` if unresolved) exists before the household would reasonably open the app
**Fallback:** if a household somehow has no generated day yet (e.g. right after onboarding, before the first scheduled run), generate on first open as a one-time exception — this should be rare, not the everyday path
**Build note:** this is real backend infrastructure (a cron job or scheduled function), not just a UI pattern — flag this explicitly when prompting Bolt/Lovable/Antigravity, since "the menu is just there" is invisible work that's easy for a build tool to skip in favor of an on-demand "Generate" button, which would undercut the whole point

All calls should return **structured JSON**, not prose, so the UI can render fields directly (protein numbers into a progress bar, rationale into a tooltip, shortfall reasons into the flag detail, etc.)

---

## 4. Non-Functional Notes
- **The default view should never demand review.** Nothing in the daily UI should read as "please check this" — flags, badges, and confirms are all optional to engage with, never gating. If a build decision makes the daily screen feel like a checklist, that's a regression against the core UX principle, not a minor detail.
- Protein/fibre are calculated from the FoodItem staple database, not freeform AI estimates — v1 ships with ~100-150 common staples pre-seeded (this is build-side work, not something the household populates); label coverage gaps clearly if an ingredient truly isn't in the database yet
- **Brand lookup requires a real web search/fetch capability, not just an LLM call** — when building in Bolt.new/Lovable/Antigravity, this needs an actual search tool wired into the backend (e.g. a server function calling a search API, then an AI call to extract structured values from the fetched page). Confirm this is available before promising the "search any brand" flow — if it isn't, fall back to the pre-seeded brand list only for v1 and flag live brand search as a fast-follow.
- Default behavior when a shortfall can't be resolved: **flag it clearly with the reason, let the user proceed anyway** — don't hard-block finalizing the day. A blocked, unusable day is worse than a visible, honest shortfall.
- Data is single-household, single-user login for MVP — no multi-tenant auth complexity needed yet
- Manual entry only for bloodwork (no PDF parsing in v1)
- Mobile-first layout: single-column, large tap targets, bottom-sheet style forms for adding items

## 5. What's explicitly deferred to Part 2/3 (don't build in this spec)
- Grocery ordering integration (Blinkit/Instamart/FirstClub)
- Inventory/pantry checks
- Importing recipes from WhatsApp voice notes / YouTube / Instagram and turning them into cook-facing output — v1 doesn't need this at all, since dishes are AI-composed live; this stays a Part 3 concept for when the household wants to preserve a specific trusted recipe verbatim (e.g. Mom's exact version of a dish)
