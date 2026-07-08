# Meal Planner — Product Spec (Living Document)

This document captures finalized decisions per milestone. Update only when a milestone is agreed and locked.

---

## Milestone status

| Milestone | Status | Documented |
|-----------|--------|------------|
| M1 — Planner MVP | **Finalized** (rev. 2) | Yes (below) |
| M2 — Inventory + 3-day lists | **Finalized** (rev. 1) | Yes (below) |
| M3 — Maid handoff (WhatsApp) | Not started | — |
| M4 — Micros + supplements | Not started | — |
| M5 — Smart rebalance | Not started | — |
| M6 — Automation | Not started | — |

---

## Cross-cutting architecture (finalized)

### Backend + auth

| Decision | Choice |
|----------|--------|
| Backend | **Self-hosted** at home; exposed to mobile app over HTTPS (Tailscale / reverse proxy) |
| Auth | **JWT** (email + password); single-user initially, multi-tenant-ready schema |
| API | REST (JSON); sync endpoints for recipes, foods, plans, profile |
| Source of truth | **Server** when online; device SQLite is cache + outbound sync queue |

### Offline support

> ⚠️ **Deferred — not in M1.** Offline support and sync have been skipped for the first iteration. The app requires an internet connection for all operations. This will be revisited post-M2.

| Capability | Online | Offline |
|------------|--------|---------|
| View today’s approved plan | ✓ | ✗ deferred |
| View / edit recipes (local) | ✓ | ✗ deferred |
| Generate plan (AI) | ✓ via home LLM | ✗ deferred |
| Approve day, swap, skip | ✓ | ✗ deferred |
| Login | ✓ | ✗ deferred |

**Sync model:** deferred to T-INFRA-3 / T-INFRA-4 post-M2.

### Plan generation: AI + code

| Mode | When | How |
|------|------|-----|
| **Primary** | Server reachable | Home server calls **local LLM** with profile, macro targets, recipe catalog, recent plan history → structured JSON plan |
| **Fallback** | Offline or LLM/server error | **Rule-based code** on device (protein-first scorer; see M1 section) |

LLM output is **validated in code** (schema, macro bounds, allergy checks) before saving as draft. Invalid response → retry once → fallback to code.

### Deferred tasks (solve later)

| ID | Task | Notes |
|----|------|-------|
| **T-INFRA-1** | Home server scaffold | Node or Python API, Postgres/SQLite, JWT auth, HTTPS expose |
| **T-INFRA-2** | Local LLM plan endpoint | `POST /plans/generate`; prompt + JSON schema; Ollama / llama.cpp / etc. |
| **T-INFRA-3** | Mobile ↔ server sync | Pull/push recipes, plans, profile; conflict handling — **skipped for M1, deferred to post-M2** |
| **T-INFRA-4** | Offline support | Local SQLite cache on device, sync queue, last-write-wins conflict resolution — **skipped for M1, deferred to post-M2** |
| **T-DESIGN-1** | App UI design in **Pencil** | Core screens: Today, Week, Recipe list/editor, Plan generate, Settings, Login. **Status: pending** — requires Pencil desktop app running |

---

# M1 — Planner MVP (Finalized, rev. 2)

## Goal

Mobile app + home backend: compute clean-bulk macro targets, maintain **recipes**, generate daily/weekly plans (AI primary, code fallback), review/swap/approve — with offline read and queued writes. No inventory, WhatsApp, or micronutrients yet.

## Success criteria

- Auth login against home server; session persists for offline use.
- Recipes CRUD with metric ingredients (oils treated as regular ingredients).
- Plan generation via server LLM when online; code fallback offline; result is immediately live.
- Swap / skip meal slots before eating; macro totals update immediately.
- Today’s active plan readable offline.

## Out of scope (M1)

- Kitchen inventory and 3-day shopping (M2).
- WhatsApp / maid messages (M3).
- Vitamins, minerals, supplements (M4).
- Eating-out logging and auto-rebalance (M5).
- Push notifications and cron jobs (M6).

---

## User profile (M1)

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | uuid | auto | |
| `email` | string | yes | Auth |
| `display_name` | string | no | |
| `sex` | enum: male, female | yes | BMR |
| `age` | int | yes | Years |
| `height_cm` | number | yes | Metric |
| `weight_kg` | number | yes | Metric |
| `activity_level` | enum | yes | sedentary … very_active |
| `goal` | fixed: `clean_bulk` | yes | |
| `protein_g` | number | yes | **Stored** — user-confirmed / edited value |
| `carbs_g` | number | yes | **Stored** — user-confirmed / edited value |
| `fat_g` | number | yes | **Stored** — user-confirmed / edited value |
| `target_kcal` | number | yes | **Stored.** Recomputed and saved whenever `protein_g`, `carbs_g`, or `fat_g` changes: `(P × 4) + (C × 4) + (F × 9)`. Used by Today screen for macro progress display. |

> **Not stored in BE:** `surplus_kcal`, `protein_g_per_kg`, `fat_pct_of_calories`. These are transient onboarding inputs used only on the FE to seed the initial PCF suggestion. They are never sent to or persisted by the server.
| `meals_per_day` | int | yes | Default 4 |
| `meal_times` | time[] | yes | breakfast, lunch, snack, dinner |
| `dietary_tags` | string[] | no | e.g. vegetarian |
| `allergies` | string[] | no | Hard exclude in planner |

---

## Macro target engine (M1 — rev. 3)

**PCF is the source of truth. Calories are always derived from P, C, F — never stored independently.**

### Phase 1: Onboarding suggestion — FE only, never hits the server

All TDEE calculation happens **on the device** during onboarding. Nothing is sent to the BE until the user taps "Start Tracking".

1. BMR — Mifflin–St Jeor (weight kg, height cm). *(FE)*
2. TDEE = BMR × activity multiplier. *(FE)*
3. Suggested kcal = TDEE + surplus_kcal *(surplus_kcal range −400 to +400; FE-only variable)*
4. Protein g = protein_g_per_kg × weight_kg. *(FE-only input)*
5. Fat g = (suggested_kcal × fat_pct) / 9. *(fat_pct is FE-only input)*
6. Carbs g = (suggested_kcal − 4×protein − 9×fat) / 4.

The resulting `protein_g`, `carbs_g`, `fat_g` are shown on **Onboarding Step 3** as editable suggestions. `surplus_kcal`, `protein_g_per_kg`, and `fat_pct_of_calories` are **discarded after this step** — they never reach the server.

### Phase 2: User confirms → POST to server

7. User reviews and optionally edits `protein_g`, `carbs_g`, `fat_g` on Step 3.
8. FE computes `target_kcal = (P × 4) + (C × 4) + (F × 9)`.
9. App POSTs `{ protein_g, carbs_g, fat_g, target_kcal, ...biometrics }` to server. Server stores exactly what it receives — no macro logic runs on the BE.

### Ongoing

- If the user edits any macro in **Settings**, the FE recomputes `target_kcal` and PATCHes `{ protein_g, carbs_g, fat_g, target_kcal }` to the server.
- All screens (Today, Week, macro display) read `target_kcal` directly from the stored profile.
- Warn in UI if `carbs_g < 100`.
- All units **metric** (g, ml, kg, cm).

---

## Food library (M1)

Base unit for all foods: **per 100g** or **per 100ml** (liquids). User-facing input only in **g** or **ml**.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | |
| `user_id` | uuid? | null = system seed |
| `name` | string | e.g. "Sunflower oil", "Chicken breast" |
| `unit_type` | enum | `solid` → per 100g; `liquid` → per 100ml |
| `calories` | number | Per 100g or 100ml |
| `protein_g` | number | |
| `carbs_g` | number | |
| `fat_g` | number | |
| `category` | enum | protein, carb, fat, vegetable, fruit, dairy, oil, spice, other |
| `is_oil` | bool | true for cooking oils (ghee, sunflower, olive, etc.) |
| `source` | enum | seed, user |

**Seed data:** ~40–60 Indian + gym staples; include common **oils** as first-class foods with `is_oil: true`.

---

## Recipe data model (M1 — finalized)

User-maintained recipes replace “meal templates”. A recipe is a structured, metric-only definition with **ingredients** (oils included as regular ingredients).

### Recipe

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | uuid | auto | |
| `user_id` | uuid | yes | |
| `name` | string | yes | e.g. "Paneer bhurji" |
| `description` | string | no | Short summary |
| `slot_types` | enum[] | yes | breakfast, lunch, snack, dinner |
| `servings` | number | yes | Default **1** (one eating occasion) |
| `prep_time_min` | int | no | |
| `cook_time_min` | int | no | |
| `tags` | string[] | no | vegetarian, meal_prep, quick |
| `instructions` | string[] | no | Ordered steps |
| `notes` | string | no | Private; maid-facing in M3 |
| `is_active` | bool | yes | Inactive excluded from generation |
| `created_at` / `updated_at` | timestamp | auto | Sync |

**Computed (cached on save, per serving):** `calories`, `protein_g`, `carbs_g`, `fat_g` — sum of all ingredients divided by `servings`. Calories derived as `(P×4) + (C×4) + (F×9)`.

### RecipeIngredient

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | uuid | auto | |
| `recipe_id` | uuid | yes | |
| `food_id` | uuid | yes | → FoodItem |
| `amount_g` | number | conditional | Required for solid foods |
| `amount_ml` | number | conditional | Required for liquid foods |
| `preparation` | string | no | e.g. "finely chopped", "soaked 4h" |
| `sort_order` | int | yes | Display / instruction order |

**Rule:** exactly one of `amount_g` or `amount_ml` is set, matching food `unit_type`.

**Macro line:**  
`solid: (amount_g / 100) × nutrient_per_100g`  
`liquid: (amount_ml / 100) × nutrient_per_100ml`

### Cooking oils

Oils (ghee, sunflower oil, olive oil, etc.) are treated as regular ingredients — added in the Ingredients section like any other food item. There is no separate cooking oil section in the UI or data model.

### Example: Paneer bhurji (1 serving)

**Ingredients**

| Food | Amount |
|------|--------|
| Paneer | 150 g |
| Onion | 50 g |
| Tomato | 80 g |
| Green chili | 5 g |
| Spices (turmeric, etc.) | 3 g |

| Sunflower oil | 10 ml |

**Instructions:** (optional ordered steps)

---

## Plan generation (M1 — AI + code)

### Shared inputs

- User profile + daily macro targets.
- Active recipes (with cached macros), filtered by slot, tags, allergies.
- Optional: last 7 days plan (variety for LLM; repeat penalty for code).

### Primary: LLM via home server (T-INFRA-2)

**Endpoint:** `POST /api/v1/plans/generate`

**Request payload (conceptual):**

```json
{
  "start_date": "2026-06-09",
  "days": 7,
  "macro_targets": { "calories": 2800, "protein_g": 160, "carbs_g": 350, "fat_g": 78 },
  "meal_slots": ["breakfast", "lunch", "snack", "dinner"],
  "recipes": [ { "id": "...", "name": "...", "slot_types": [], "macros": {}, "tags": [], "allergens": [] } ],
  "constraints": { "dietary_tags": [], "allergies": [], "avoid_recipe_ids": [] }
}
```

**Expected LLM response (JSON):**

```json
{
  "days": [
    {
      "date": "2026-06-09",
      "slots": [
        { "slot_type": "breakfast", "recipe_id": "uuid", "skipped": false }
      ]
    }
  ],
  "rationale": "optional short text"
}
```

**Server-side validation (code):**

- JSON schema valid.
- Each `recipe_id` exists and matches slot type.
- No allergy violations.
- Day macro sum within ±15% of targets (protein within ±10g); else reject and retry or fallback.

### Fallback: rule-based code (device or server)

When offline, LLM timeout, or validation fails twice:

1. For each day, for each slot in order:
2. Candidates = active recipes for slot.
3. Score vs remaining macros (protein first, then L1 distance on cal/P/C/F).
4. Penalize same recipe as prior day same slot.
5. Pick best; subtract macros from remaining.

**Generate week** → `MealPlan` status `draft`. **Regenerate day** → single day only.

### Plan states

| Status | Meaning |
|--------|---------|
| `active` | Generated and live — the default state; user eats from this |
| `modified` | User swapped or skipped one or more slots before eating |

> There is no `draft` or `approved` state. A generated plan is immediately active. The distinction between original and modified is tracked internally for sync purposes but not surfaced to the user as a gate.

---

## User intervention (M1)

> **UX principle:** The generated plan is live by default — no manual approval step. Meals are assumed to happen as planned. The user edits a slot *before eating* if they want to change it.

| Action | Behavior |
|--------|----------|
| Swap meal | Pick another recipe for slot before eating; recompute totals immediately |
| Skip slot | Mark skipped before eating; totals drop |
| Edit recipe | CRUD; if used in today's plan, macro totals refresh automatically |
| Generate | AI if online, else code fallback; result is immediately live (no draft state shown to user) |

---

## Screens (M1)

1. **Login** — email/password → home server.
2. **Onboarding** — profile → macro summary → create first recipe.
3. **Today** — slots, macro progress, approve.
4. **Week** — 7-day grid.
5. **Generate plan** — CTA; shows “AI” vs “Offline rules” badge.
6. **Recipes** — list, search, filter by slot/tag.
7. **Recipe editor** — metadata, **ingredients (g/ml)**, **cooking oil (ml)**, instructions, macro preview.
8. **Food library** — search, add custom (g/ml nutrients per 100).
9. **Settings** — profile, server URL, macro overrides, sync status.

**Tabs:** Today | Week | Recipes | Settings

---

## Tech stack (M1 — finalized, rev. 2)

| Layer | Choice |
|-------|--------|
| Mobile | React Native + Expo, TypeScript |
| Local DB | SQLite (expo-sqlite) + sync queue |
| Client state | Zustand |
| Backend | Self-hosted (Node.js + Fastify or Python + FastAPI) — **T-INFRA-1** |
| Server DB | PostgreSQL (or SQLite for solo home server) |
| Auth | JWT access + refresh tokens |
| LLM | Local on home machine via Ollama (or equivalent) — **T-INFRA-2** |
| Design | Pencil (.pen files in repo) — **T-DESIGN-1** |

---

## Data model — server & client (M1)

### Server tables

- `users` (auth)
- `user_profiles`
- `food_items`
- `recipes`
- `recipe_ingredients`
- `meal_plans`
- `meal_plan_slots`

### Client SQLite (mirror + sync)

Same entities + `sync_queue` + `sync_metadata` (last_pull_at, server_cursor).

### Entity relationships

```
User 1──1 Profile
User 1──* Recipe
Recipe 1──* RecipeIngredient ──* FoodItem
Recipe 1──* RecipeCookingOil ──* FoodItem (is_oil)
User 1──* MealPlan 1──* MealPlanSlot ──? Recipe
```

---

## M1 build order (updated)

1. **T-DESIGN-1** — Pencil: core screens + recipe editor (ingredients / oil sections).
2. **T-INFRA-1** — Server scaffold, auth, recipe CRUD API.
3. Mobile scaffold, auth, SQLite mirror.
4. Profile + macro engine + tests.
5. Seed foods (incl. oils) + food library UI.
6. Recipe CRUD + macro rollup (ingredients + oils).
7. **T-INFRA-2** — LLM plan endpoint + validation.
8. Code fallback planner on device.
9. Week / Today UI, generate, swap, skip, approve.
10. **T-INFRA-3** — Sync layer.
11. Onboarding + polish.

---

## Open questions (defer)

- Exact LLM model and prompt tuning → T-INFRA-2.
- Training vs rest day macros → M5.
- HTTPS expose method (Tailscale vs Cloudflare tunnel) → T-INFRA-1.

---

---

# M2 — Inventory + 3-day lists (Finalized, rev. 1)

## Goal

Track kitchen inventory per food item, auto-deduct at end of day based on the active plan, compute a shopping list by diffing the upcoming 3-day plan against current stock, and surface a bulk-order alert on the Today screen when 4 or more items need restocking.

## Success criteria

- Each food item has an inventory record with current stock and tier.
- End-of-day close-out deducts all non-skipped slot ingredients from inventory.
- Shopping list is computed from next 3-day plan vs current stock.
- Today screen shows a "Time to shop" banner when ≥ 4 items are below threshold.
- User can review, check off, and mark items as restocked.

## Out of scope (M2)

- WhatsApp send of shopping list (M3).
- Micronutrient tracking per food (M4).
- Eating-out deviations (M5).

---

## Inventory data model (M2)

### InventoryItem

One record per `FoodItem` the user wants to track. Foods with no inventory record are treated as always-available (e.g. spices in bulk).

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `id` | uuid | auto | |
| `user_id` | uuid | yes | |
| `food_id` | uuid | yes | FK → FoodItem |
| `tier` | enum | yes | `fresh`, `perishable`, `staple`, `frozen` |
| `quantity_g` | number | conditional | For solid foods |
| `quantity_ml` | number | conditional | For liquid foods |
| `restock_threshold_g` | number | conditional | Low-stock trigger for solids |
| `restock_threshold_ml` | number | conditional | Low-stock trigger for liquids |
| `last_restocked_at` | timestamp | no | Set when user marks restocked |
| `updated_at` | timestamp | auto | Sync |

**Tier → replenish cycle:**

| Tier | Cycle | Examples |
|------|-------|---------|
| `fresh` | 3 days | paneer, curd, tomato, leafy greens |
| `perishable` | 5–7 days | eggs, chicken, fish |
| `staple` | when low | rice, lentils, oats, oils |
| `frozen` | when low | frozen peas, frozen corn |

---

## End-of-day close-out (deduction)

**Trigger:** midnight server job (or manual "Close out today" tap on Today screen).

**Algorithm:**

1. Fetch today's `MealPlanSlot` records for the user.
2. For each slot where `skipped = false`:
   - Sum `RecipeIngredient.amount_g / amount_ml` for each ingredient.
   - Sum `RecipeCookingOil.amount_ml` for each oil.
3. For each food: subtract consumed quantity from `InventoryItem.quantity_g` or `quantity_ml`.
4. Clamp to 0 (never go negative in DB — flag a warning log instead).
5. After deduction, recompute shopping list (see below).

**Server job:** runs at 00:05 local time. If the user closes out manually before midnight, the job is a no-op for that day.

---

## Shopping list computation

Runs after every end-of-day deduction and on-demand when user opens the Shop view.

**Algorithm:**

1. Pull next 3 days of `MealPlanSlot` (starting from tomorrow; if no plan exists for a day, skip it).
2. For each slot → recipe → ingredients + cooking oils, accumulate total needed per `food_id`.
3. For each food with an `InventoryItem`:
   - `needed = total_needed_for_3_days`
   - `available = current quantity`
   - `deficit = max(0, needed − available)`
   - Also flag if `available < restock_threshold` regardless of plan (staple low-stock).
4. Output: list of `ShoppingListItem` records.

### ShoppingListItem (ephemeral, not persisted)

| Field | Type | Notes |
|-------|------|-------|
| `food_id` | uuid | |
| `food_name` | string | Denormalized for display |
| `tier` | enum | For grouping |
| `deficit_g` | number | How much to buy (solid) |
| `deficit_ml` | number | How much to buy (liquid) |
| `reason` | enum | `plan_deficit`, `below_threshold` |

Shopping list is **not stored** — it is recomputed on demand. The server endpoint is `GET /api/v1/inventory/shopping-list`.

---

## Bulk-order alert on Today screen

**Rule:** if the shopping list has **≥ 4 items** (across all tiers), surface a banner on the Today screen.

**Banner copy:** "Time to stock up — X items running low. Tap to review."

**Behavior:**
- Banner appears at top of Today screen below the macro summary.
- Tapping opens the Shop view (full shopping list).
- Banner dismisses automatically when item count drops below 4 (after a restock).
- No push notification in M2 (M6).

---

## User flow

1. **End of day** — close-out runs (auto at midnight or manual tap).
2. **Shopping list recomputed** — deficit items flagged.
3. **Today screen** — if ≥ 4 items, bulk-order banner appears.
4. **Shop view** — user sees items grouped by tier (fresh / perishable / staple / frozen), with deficit quantities.
5. **User shops** — taps each item as bought; enters restocked quantity.
6. **Mark restocked** → updates `InventoryItem.quantity_g/ml`, sets `last_restocked_at`.
7. **Shopping list recomputed** — banner clears if < 4 items remain.

---

## Screens (M2)

### Today screen addition

- **Bulk-order banner** (conditional, ≥ 4 low-stock items) — below macro summary, above slot cards.

### New: Shop screen (tab or modal)

| Section | Content |
|---------|---------|
| Header | Item count + "next 3-day plan" label |
| Grouped list | Tier sections (fresh first, then perishable, staple, frozen) |
| Each row | Food name · deficit quantity · reason badge (Plan / Low stock) · checkbox |
| Footer CTA | "Mark all checked as restocked" → quantity entry sheet |

**Bottom tabs:** Today | Week | Recipes | **Shop** | Settings

---

## API additions (M2)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/inventory` | GET | All inventory items for user |
| `/api/v1/inventory/:food_id` | PUT | Update quantity / threshold / tier |
| `/api/v1/inventory/close-out` | POST | Trigger end-of-day deduction manually |
| `/api/v1/inventory/shopping-list` | GET | Computed shopping list (next 3 days) |
| `/api/v1/inventory/restock` | POST | Mark items as restocked with new quantities |

---

## Server tables (M2 additions)

- `inventory_items` — one row per tracked food per user.
- `inventory_deductions` — audit log of each close-out (date, food_id, amount_deducted). Useful for debugging and M5 analytics.

---

## Data sync (M2)

`inventory_items` syncs via the same `sync_queue` mechanism as M1 entities. `inventory_deductions` is server-only (audit log, not needed on device). Shopping list is computed server-side and not cached on device.

---

## M2 build order

1. DB: `inventory_items` + `inventory_deductions` tables (server + client mirror).
2. Inventory CRUD API + mobile UI (food library gets "track in inventory" toggle).
3. Close-out algorithm + server midnight job + manual trigger endpoint.
4. Shopping list computation endpoint.
5. Shop screen (list, grouped by tier, check-off, restock quantity entry).
6. Today screen: bulk-order banner (≥ 4 items threshold).
7. Sync layer extension for inventory entities.

---

## Open questions (defer)

| Question | Defer to |
|----------|----------|
| Quantity units for display (show g or convert to "1 packet / 500g bag"?) | UX decision at build time |
| Should `staple` tier items auto-add to list below threshold even if not in next 3-day plan? | Yes — already in algorithm (`below_threshold` reason) |
| Offline shopping list? | Device can cache last-computed list; recompute requires server |
