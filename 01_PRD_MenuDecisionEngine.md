# PRD: Menu Decision Engine
**Product:** [Working name] NutriMenu — Part 1 of the household meal-planning app
**Owner:** Somya
**Status:** Draft v1
**Scope of this doc:** Only Part 1 — deciding what to eat, respecting nutrient needs and preferences. Grocery ordering and recipe-sharing are separate, later parts (referenced but not specified here).

---

## 1. Problem Statement
Health-conscious young households (often dual-income couples with a household cook) struggle daily to decide what to eat. The decision has to simultaneously satisfy:
- Each member's food preferences and restrictions
- Each member's nutrient needs (protein, fibre, iron, calcium, vitamins), which vary by weight, activity level, health conditions, and lab work
- Variety, so the same 4-5 meals don't repeat endlessly
- Quantities appropriate for the number of people, so the cook can actually prepare the right amount

Today this is done manually, from memory, every single day — mentally juggling multiple people's constraints. It's mentally taxing and nutrient targets are usually a guess, not a calculation.

## 2. Design Principle: Trust by Default
The app only succeeds if it removes the daily mental load, not relocates it into a review habit. That means:
- **A day's menu is auto-finalized, not pending approval.** The user should never need to review or approve each item before it's usable — the menu that appears each day is already the answer, ready to hand to the cook.
- **Every safety/quality mechanism exists to make that trust justified, not to create a checklist.** Hard nutrient targets, exclusion enforcement, and the substitution/gap-filling engine all run silently in the background so the *default* output is already correct — they are not surfaced as things the user must verify.
- **Editing is for flexibility and exceptions, not a daily task.** Regenerate, swap, replace, and override all exist for special days or genuine preference changes — none of them should read as "you should probably check this."
- Any flag, confirm affordance, or badge added anywhere in this product should be judged against this bar: does it demand attention, or is it truly optional and ignorable? If it demands attention on a normal day, it doesn't belong in the default view.

## 3. Goals
- Let a household add each member's profile once (diet type, restrictions, weight/height, activity, health notes, supplements)
- Automatically calculate personalized nutrient targets per member (protein especially, plus fibre/iron/calcium/vitamins directionally), using an AI model reasoning over bloodwork + weight + activity — with the user able to override/tweak any target
- Generate a day's (eventually week's) menu per meal that collectively meets the household's nutrient targets, respects restrictions, rotates variety, and leans on what's actually in season for the household's region
- Have the AI generate the specific dish for each slot live — including which variant (e.g. which dal, which rice, which sabzi) — using a built-in staple ingredient database for real nutrient grounding, so the household never has to set up or maintain a recipe collection themselves
- Deliver each day's menu already finalized and trustworthy — editing, regenerating, or swapping stays available whenever the user wants it, but nothing requires their review to be usable

## 4. Non-Goals (out of scope for this doc)
- Grocery ordering / inventory checking (Part 2)
- Recipe sourcing and formatting for the cook, WhatsApp/YouTube/Instagram recipe ingestion (Part 3)
- Multi-household/family accounts beyond one household unit
- Precise clinical nutrition/medical advice — this is a planning aid, not a diagnostic tool
- Hyper-local produce specificity (e.g. what's fresh in one specific city vs. its wider climate region) — v1 works at the climate-zone level, not city-by-city

## 5. Target Users & Persona

**Primary persona — household menu manager**
> Ana, 29. Works out regularly, wants to eat higher protein and improve gut health. Doesn't eat paneer or milk (acne-prone), avoids added sugar. Uses a plant protein (Cosmix) but skips it when bloated. Eats yoghurt/tofu occasionally, eggs most days but not daily. Manages the household menu for herself and her husband, Ben.

**Secondary persona — household member with different needs**
> Ben, Ana's husband. Eats paneer, milk sometimes, drinks whey protein. Athlete — higher protein/calorie needs than Ana.

**Key insight from the persona:** even two people in one household can have opposite constraints (she avoids paneer/milk, he includes them) and different protein targets (bodyweight-based, adjusted for activity). The engine must plan **per-member targets** but output a **shared, executable menu** the cook can cook once.

## 6. User Stories
1. As a household manager, I can add household members and, for each, capture: diet type (veg/non-veg/vegan/eggetarian), foods to avoid, weight, height, activity level, health notes (e.g. gut health concerns), supplements taken (and conditions under which they skip them), and bloodwork highlights.
2. As a household, I can set our location once, so the app knows our region's current season and can lean toward what's actually in season, rather than treating "seasonal" as generic to all of India.
3. As a user, I want the app to suggest a daily nutrient target (protein especially) per member, starting from a baseline (1g protein per kg bodyweight) and adjusted upward (toward ~2g/kg) based on activity level, goals, and gut/health notes — and I want to be able to manually override this number.
4. As a user, I want to see, each day, a suggested menu (breakfast, mid-morning fruit, lunch, evening snack, dinner) that meets both members' targets collectively, respects what each person won't eat, avoids repeating the last few days' meals, and favors in-season produce.
5. As a user, I want to swap out any suggested item and see the nutrient totals update.
6. As a user, I want to mark a supplement as "skip today" (e.g. bloating) and have the day's plan re-account for the protein gap.
7. As a user, I want quantities scaled to the number of people eating, so the cook knows how much to prepare.
8. As a user, I don't want to set up or maintain a recipe collection — I want the app to decide the specific dish (which dal, which rice, which sabzi) for me, guided only by a few quick preferences I set once (preferred grains, dal likes, spice level).
9. As a user, when a generated dish lands well, I want a one-tap way to mark it a favorite so it can resurface later, without having to manually build a recipe entry for it.

## 7. Key Requirements

### 7.0 Household Profile
- Location (city + state, or a region picker) — mapped once to a climate zone, used to determine the household's current season
- **Cuisine identity:** a primary/native cuisine (e.g. "North Indian," "Bengali," "South Indian") independent of where the household currently lives, plus an optional list of cuisines they enjoy occasionally (e.g. "South Indian," "Chinese," "Italian") — this shapes *what kind* of dishes get suggested, separate from the nutrient/seasonal logic
- Members (see 7.1)

### 7.1 Member Profile (data captured once, editable)
- Name, age, weight, height, activity level
- Diet type: veg / non-veg / vegan / eggetarian / pescatarian
- Explicit exclusions (e.g. "no paneer, no milk")
- Occasional/conditional items (e.g. "eats eggs, not daily"; "yoghurt/tofu at times")
- Health notes relevant to nutrition (e.g. gut health focus) — self-reported, free text
- Supplements: name, usual frequency, conditions to skip (e.g. "skip when bloated")
- Bloodwork: key values user chooses to enter (e.g. iron, B12, vitamin D) — optional, manual entry for v1
- Goal: e.g. "improve gut health," "maintain athletic performance"
- Food format preferences (optional, quick-select — not a setup burden): preferred grains (rice / roti / millet / mixed), dal likes, spice level (mild/medium/spicy), any cooking-style notes (e.g. "avoid deep-fried") — this is what lets the AI generate specific dishes that actually fit the person, without needing a recipe collection

### 7.2 Nutrient Target Engine (AI-assisted)
- Baseline protein target = 1g × bodyweight(kg)
- AI adjusts the target upward (toward 2g/kg ceiling) based on activity level, stated goals, and health notes — and shows *why* it landed on that number
- **Protein and fibre are hard daily targets, not directional goals** — a day that falls short on either, even by one meal slot, is not an acceptable result and must be resolved via substitution or a top-up side dish (see 7.5) before the day is considered "done"
- Iron, calcium, and key vitamins remain directional "focus areas" (not gram-exact) based on age/sex/health notes and bloodwork flags — these stay softer than protein/fibre since v1's nutrient data isn't precise enough for them yet
- Nutrient math is calculated from a **starter nutrient library** (see 7.5.1) rather than a generic AI guess — grams of protein/fibre per item come from known values, not estimated per-meal by the AI
- Every target is user-editable; edits are remembered as the new baseline until changed again

### 7.3 Menu Generation
- Meal slots: breakfast, mid-morning (seasonal fruit by default), lunch, evening snack, dinner — five slots, not four
- **Generated proactively, ahead of time — not on-demand.** The day's menu is ready before the household needs it (e.g. generated the evening before, or first thing each morning via a scheduled job), so opening the app never means waiting on a "Generate" action. This is the concrete mechanism behind the Trust by Default principle: there's nothing to trigger, nothing to wait on, just a ready answer.
- Input: household's member targets + exclusions + food format preferences + the household's cuisine identity (primary + occasional) + recent meal history (to ensure variety) + meal slot + the household's current season (from 7.0's location) + the staple ingredient database (7.5.1)
- Output: the AI **composes the specific dish live** (e.g. "moong dal tadka," not just "dal") — name, variant, and a realistic ingredient breakdown with quantities — with **calculated** (not estimated) protein/fibre by mapping that breakdown onto the staple database, tagged to whichever members it's suitable for, and a quantity note scaled to headcount
- **Cuisine identity, not just current location:** dish style is weighted toward the household's stated native cuisine most days — a household's food identity comes from where they're *from*, not necessarily where they currently live. Occasional cuisines they've named (e.g. South Indian, Chinese, Italian) should genuinely show up sometimes, not be excluded just because they're not the primary — a North Indian household that likes dosa or the occasional pasta should actually see that reflected periodically.
- **Meal-level protein distribution:** breakfast and lunch should carry the heavier share of the day's protein target; dinner can be lighter but must never be protein-free — every meal slot contributes something
- **Seasonal-first filtering:** among valid options, the generator favors produce tagged for the household's current season before optimizing purely for nutrient fit
- **Sides, not just mains:** curd, yoghurt, or a simple salad can be attached to any meal or snack slot as an optional side — these are lightweight, flexible add-ons, not separate full items
- **Variety through the AI's own culinary judgment:** the AI draws on well-established home-cooking patterns (dal + rice, paneer + roti, etc., within the household's cuisine mix) to compose dishes live — no hand-built recipe collection required, since this kind of everyday meal-pattern knowledge is well within general model knowledge; variety stays constrained by "does this combination actually work," not just "is it different from yesterday"
- User can: accept (the default — no action needed), regenerate, or manually replace any item; a loved dish can be favorited in one tap so it can resurface later
- System tracks a rolling history (e.g. last 5-7 days) to reduce repeats

### 7.4 Household-Level View
- One combined day view is the default (so the cook gets one instruction), with per-member nutrient coverage shown underneath
- If a day's running total is short on someone's protein/fibre target, the system first tries to resolve it automatically via substitution or a top-up side dish (7.5) before ever showing the user a shortfall
- If it genuinely can't be resolved (e.g. no acceptable substitute/side available), the day is clearly flagged as short, with the specific reason shown (e.g. "12g short on Ana's protein — no more paneer/tofu-based sides available today") — the user can still proceed, but the shortfall is never silently hidden

### 7.5 Substitution & Gap-Closing Engine
- **Substitution mapping:** interchangeable ingredients (e.g. paneer ↔ tofu) are mapped so a dish built around one can be re-suggested with the other for a member who excludes it, without materially changing the recipe
- **Top-up sides:** when a member's running total is short, the system can suggest a simple, low-effort side (e.g. "plain paneer, salt & pepper, 50g") purely to close the gap — these are tagged as gap-fillers, not full menu items, so they read differently in the UI
- **Resolution order:** try a substitution first (keeps the meal intact for everyone), fall back to a top-up side if no substitution closes the gap

### 7.5.1 Nutrient Library (built-in staple set)
- v1 ships with a **built-in database of common staples** (~100-150 items — dals, rice/grain varieties, common vegetables, paneer/tofu/soy variants, eggs, common proteins) — this is real one-time setup work on the build side, not something the household has to populate
- **Branded items are looked up automatically, never manually entered.** The household picks or names a brand (e.g. "Akshayakalpa paneer"); the app searches for the brand's official listing, extracts the label values, and saves and uses them right away. A source link and an optional "looks right?" confirm are shown, but nothing blocks on the household reviewing it. A short set of popular brands (paneer, curd, tofu, protein powder) ships pre-verified so most households won't need to search at all.
- Values account for **preparation state** (raw / cooked / boiled / fried) since nutrient density changes with cooking
- The library can still grow (new branded items, items not yet covered), but a solid common-staple set ships from day one — this is what lets AI-composed dishes (7.6) actually get calculated, not just guessed

### 7.6 AI-Composed Dishes & Seasonal Logic
- **No recipe setup required:** the AI composes the specific dish live for each slot — deciding which dal, which rice, which sabzi — drawing on general knowledge of common Indian home-cooking patterns, not a hand-built recipe collection the household has to populate
- **Nutrient grounding stays real:** the AI's ingredient breakdown for a composed dish (e.g. "150g paneer, 1 tsp oil...") gets mapped onto the staple database (7.5.1) for calculated, not guessed, protein/fibre — the remaining estimation is just realistic ingredient *quantities* for a given dish, which is a much narrower and more reliable task than guessing total nutrition freeform
- **Favoriting, not maintaining:** when a generated dish works well, the user can favorite it in one tap; favorited dishes are more likely to resurface later — this is how a personal "library" builds up organically, as a byproduct of using the app rather than a setup task
- **Seasonal tagging:** produce is tagged by season (e.g. "monsoon," "summer"), not by exact region — see 7.6.1 for how "current season" is determined
- **Location → season, not location → produce:** the household's location determines *which season it currently is* for them; it does not drive a fully region-specific produce list (that precision is out of scope for v1 — see Non-Goals)

### 7.6.1 Determining the Current Season
- Household sets their location (city + state) once during onboarding
- Location maps to one of a small set of predefined **climate zones** (e.g. North Indian Plains, Deccan Plateau, Western Coastal, Eastern/Northeast, Himalayan/Hill) — each zone has its own month-to-season calendar, since monsoon/summer/winter timing genuinely differs by region
- The app auto-infers the current season from today's date + the household's climate zone — no manual season-setting needed day to day
- Recipes/produce keep a universal season label (e.g. "monsoon"); the climate zone only decides *when* that label currently applies for this household

## 8. Success Metrics (for MVP / portfolio purposes)
- Time to decide a full day's menu drops from "10+ minutes of mental effort" to a few taps
- **Primary metric:** % of days where both members' protein AND fibre targets are fully met — a day resolved only via an explicit, visible shortfall flag does not count as a success, even though it's a better outcome than a silent gap
- Variety: no meal repeated more than once in a rolling 5-day window (self-reported/tracked), without resorting to combinations that don't actually make sense together
- Qualitative: does the household manager trust the suggested menu enough to send it to the cook with minimal edits — with zero daily review needed to get there?

## 9. Assumptions
- v1 supports a 2-person household but the data model should generalize to N members without redesign
- v1 ships with a built-in staple ingredient database (~100-150 common items) rather than a full certified nutrition database/API or a user-maintained recipe collection — the AI composes dishes live and quantities are calculated against this staple set; broader coverage (less common items, packaged foods) is a v2 upgrade
- Dish composition still involves the AI estimating realistic ingredient *quantities* for a given dish (e.g. how much paneer in a serving of bhurji) — this is narrower and more reliable than guessing total nutrition freeform, but isn't lab-precise
- Default behavior when a day can't be fully resolved is to flag the shortfall clearly and let the user proceed, not to hard-block finalizing the day
- Bloodwork is entered manually as a few key flagged values, not parsed from PDF reports in v1
- Menus generate proactively on a schedule (not on open/on-demand); this needs a scheduled backend job, which is a real build dependency, not just a UI behavior

## 10. Open Questions
- Should v1 support more than 2 household members, or is 2 an acceptable hard limit for now (simpler onboarding, still generalizable data model)?
- Do you want a weekly view eventually, or is "decide today, look at yesterday for variety" enough for v1?
- Should the cook ever see this directly (e.g. a simplified read-only view), or does Part 1 stop at "Ana sees the plan" and telling the cook stays manual/Part 3?
