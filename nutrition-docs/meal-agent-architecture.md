# Meal Planner Agent — Architecture (Draft for Review)

> **Status:** Draft — awaiting your edits before we build.
> **Goal:** Generate a daily meal plan that (a) respects the user's fixed dietary preferences, (b) hits macro/calorie targets within 5–10%, and (c) is not repetitive day to day.

---

## 1. Objective & guiding principles

| Principle | Meaning |
|-----------|---------|
| **Preference-first** | The user has constant habits (e.g. "3–5 boiled eggs every morning", "1–2 whey shakes as evening snack"). These are *pinned* into the plan before anything else is decided. |
| **Macro accuracy** | Total day must land within **±5–10%** of the target for calories, protein, carbs, fat. |
| **Practical meals** | No nonsensical plates (e.g. "dal + sambar" with no rice or roti). |
| **Variety** | Avoid repeating the same recipes as recent days. |
| **Deterministic where it matters, creative where it helps** | MILP does the math (guaranteed feasible), LLM does the human judgment (preferences + practicality). |

---

## 2. High-level architecture — 3 layers

```
                          ┌──────────────────────────────────────┐
   User Profile           │  LAYER 1 — Preference Resolver (LLM)  │
   • macro targets        │                                      │
   • preferences text ───►│  Assign preferred items to slots.     │
   • recipe/food catalog  │  Subtract their macros from targets.  │
                          └──────┬─────────────────────┬─────────┘
                                 │                     │
              residual_targets   │                     │  preferedItems
              (to MILP)          │                     │  (slot + recipe)
                                 ▼                     │
                 ┌──────────────────────────────┐      │
                 │  LAYER 2 — MILP Solver        │      │
                 │  (PuLP/CBC)                   │      │
                 │  Binary, 1 serving each.      │      │
                 │  Enumerate ALL feasible plans.│      │
                 └───────────────┬──────────────┘      │
                                 │  all valid plans     │
                                 │  (no quantity)       │
                                 ▼                     ▼
                          ┌──────────────────────────────────────┐
                          │  LAYER 3 — Practicality Filter (LLM)  │
                          │                                      │
                          │  Merge preferedItems into slots.      │
                          │  Reject impractical combos. Pick one  │
                          │  at random (variety handled by MILP). │
                          └───────────────┬──────────────────────┘
                                          │ final plan
                                          ▼
                              validate + save to DB
```

---

## 3. Data model changes

### 3.1 New: user meal preferences

Preferences are **constant** and stored per user. To keep it simple, **quantity is baked into the recipe** — e.g. a recipe called **"5 boiled eggs"** (qty always 1), **"1 whey shake"** (qty always 1). No min/max ranges.

A new table `meal_preferences`:

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | |
| `user_id` | uuid | FK |
| `recipe_id` | uuid | The preferred recipe, e.g. "5 boiled eggs", "1 whey shake" |
| `slot_type` | enum | breakfast / lunch / snack / dinner |
| `is_active` | bool | |

> Free-text preferences (`preferences_text` on `user_profiles`) may still be stored so Layer 1 can read the user's intent, but the resolved output is always a fixed recipe + slot.

### 3.2 Preferred items as recipes

"5 boiled eggs" and "1 whey shake" are just **normal recipes** with fixed macros for that portion. This reuses the existing `recipes` table and `MealPlanSlotRecipe` join. Quantity is always **1 serving** — no schema change needed.

---

## 4. Layer-by-layer contract

### LAYER 1 — Preference Resolver (LLM)

**Purpose:** Read the user's preference text, map each preferred item to a **recipe + slot**, and compute the *residual* macro budget for the MILP. Its `preferedItems` output is passed **directly to Layer 3** (it bypasses the MILP). The `residual_targets` output goes to Layer 2.

**Input:**
String of the user Prompt: <*prompt from user profile*> + available recipe catalog.

**What it decides:** which recipe each preference maps to and which slot it belongs in (e.g. "5 boiled eggs" → breakfast, "1 whey shake" → snack). Quantity is always 1. It then subtracts the preferred items' macros from the day target to produce the residual budget.

**Output:**
```json
{
  "preferedItems": [
    { "slot": "breakfast", "recipe_id": "<5-boiled-eggs>" },
    { "slot": "snack",     "recipe_id": "<1-whey-shake>" }
  ],
  "residual_targets": { "calories": 2381, "protein_g": 70.8, "carbs_g": 341.6, "fat_g": 55.8 }
}
```
*(residual = target − sum of preferred item macros)*

- `residual_targets` → **Layer 2 (MILP)**
- `preferedItems`   → **Layer 3** (merged back into slots after solving)

---

### LAYER 2 — MILP Solver (PuLP/CBC)

**Purpose:** Fill the slots to hit the **residual** macro targets. Each recipe is a simple **binary on/off, 1 serving**. Enumerate **all feasible plans** (not a capped N).

**Input:**
```json
{
  "residual_targets": { "calories": 2381, "protein_g": 70.8, "carbs_g": 341.6, "fat_g": 55.8 },
  "recipes_by_slot": { "breakfast": [...], "lunch": [...], "snack": [...], "dinner": [...] },
  "recent_recipe_ids": ["...", "..."],
  "tolerance": { "calories": 0.10, "protein_g": 8, "carbs_g": 0.10, "fat_g": 0.10 }
}
```

**Decision variables — binary only, no quantity:**

| Variable | Domain | Meaning |
|----------|--------|---------|
| `x(slot, recipe)` | binary `{0, 1}` | 1 = recipe used in that slot (exactly 1 serving) |

Macros: `slot_protein = Σ x(slot,recipe) × recipe.protein_per_serving`.

**Constraints:**
- Full-day (residual) macros within **±10%** tolerance (protein ±8g).
- Each slot: 1–3 *distinct* recipes (`Σ x(slot,·) ∈ [1,3]`). Preferred items do **not** count toward this limit — they are added on top in Layer 3.
- No recipe used twice in the same day.
- Recency penalty on recently used recipes (soft, in objective).

**Enumerate ALL feasible plans:** after each solve, add an **integer cut** excluding that exact combination and re-solve. Repeat until the model is infeasible → this yields **every** valid plan. (Optionally cap for runtime safety, but default is exhaustive.)

**Output — recipe IDs only, no quantity:**
```json
{
  "plans": [
    {
      "slot_assignments": [
        { "slot_type": "breakfast", "recipe_ids": ["<oats>"] },
        { "slot_type": "lunch",     "recipe_ids": ["<chapati>", "<paneer-bhurji>"] },
        { "slot_type": "snack",     "recipe_ids": ["<banana>"] },
        { "slot_type": "dinner",    "recipe_ids": ["<rice>", "<dal>"] }
      ],
      "day_macros": { "calories": 2381, "protein_g": 71, "carbs_g": 342, "fat_g": 56 }
    },
    { ... plan 2 ... }
  ]
}
```

> Note: `day_macros` here is the **residual** (preferred items not included yet). Preferred items are added back in Layer 3, at which point the plan hits the full day target.

> If the solver returns **0 plans**, tolerance is too tight or preferences make it infeasible → surface a clear error (see §6).

---

### LAYER 3 — Practicality Filter + Picker (LLM)

**Purpose:** Merge the preferred items back into each plan's slots, discard combinations that are *culinarily wrong*, then randomly pick one survivor for variety.

**Input:**
```json
{
  "plans": [ ...from Layer 2... ],
  "preferedItems": [
    { "slot": "breakfast", "recipe_id": "<5-boiled-eggs>" },
    { "slot": "snack",     "recipe_id": "<1-whey-shake>" }
  ]
}
```
> No history needed — recency/variety is already handled by the MILP's recency penalty in Layer 2.

**What it does:**
1. **Merge** — add each `preferedItem` into its slot for every plan (so breakfast now = oats + 5 boiled eggs, snack = banana + 1 whey shake).
2. **Judge** each merged plan **practical / impractical** with a one-line reason.
   - Impractical example: "lunch = dal + sambar" (two liquids, no carb base).
   - Practical example: "lunch = chapati + paneer bhurji".
3. Return the **practical plan indices**.

**Output:**
```json
{
  "verdicts": [
    { "plan": 1, "practical": true,  "reason": "Balanced Indian plate" },
    { "plan": 2, "practical": false, "reason": "Dinner is dal + sambar with no rice/roti" },
    { "plan": 3, "practical": true,  "reason": "Good pairings" }
  ],
  "practical_plans": [1, 3]
}
```

**Picker (code, not LLM):** from `practical_plans`, choose one at **random** → day-to-day variety. If none are practical, fall back to the plan with smallest macro deviation. The chosen plan (slots + merged preferred items) is saved.

---

## 5. End-to-end example

**Targets:** 2933 kcal, 144g P, 350g C, 80g F
**Prefs:** "5 boiled eggs" @ breakfast, "1 whey shake" @ snack

1. **Layer 1 (LLM)** maps prefs to recipes + slots: `preferedItems = [5-eggs@breakfast, whey@snack]`. Subtracts their macros → residual budget (e.g. 2381 kcal, 70.8g P). Emits `residual_targets` (→ Layer 2) and `preferedItems` (→ Layer 3).
2. **MILP** fills slots with binary, 1-serving recipes to hit the residual ±tol. Enumerates **all** feasible plans (e.g. 40 of them). Each plan is recipe IDs only, no quantity.
3. **Layer 3 (LLM)** merges preferred items into each plan (breakfast = oats + 5 eggs, snack = banana + whey), flags impractical ones (dal + sambar with no carb), keeps the practical set.
4. **Random pick** selects one practical plan → validate + save.
5. Tomorrow, recency penalty + fresh random pick yields a different plan.

---

## 6. Failure handling

| Failure | Behavior |
|---------|----------|
| MILP infeasible (prefs + tolerance impossible) | Return error "targets unreachable with current preferences" (tolerance already at ±10%). |
| LLM filter fails / bad JSON | Skip filtering; random-pick from all N plans. |
| LLM filter marks all impractical | Pick the plan with smallest macro deviation. |
| omlx/LLM down | Same as above — code fallback keeps generation working. |

---

## 7. Components to build

| # | Component | File | Type |
|---|-----------|------|------|
| 1 | `meal_preferences` table + model (recipe + slot) | `models/preference.py` + migration | DB |
| 2 | Preference schema + CRUD API | `schemas/preference.py`, `routers/preferences.py` | API |
| 3 | Layer 1 — map prefs → recipe/slot + residual (per generation) | `agent/preference_resolver.py` | LLM |
| 4 | Layer 2 — MILP binary, 1 serving, enumerate all feasible plans | `agent/solver.py` (rewrite) | Code |
| 5 | Layer 3 — merge preferred items + practicality filter | `agent/analyser.py` (rework) | LLM |
| 6 | Random picker + fallbacks + orchestration | `agent/orchestrator.py` (rework) | Code |
| 7 | Seed recipes for preferred items ("5 boiled eggs", "1 whey shake") | `seeds/` | Data |

---

## 8. Resolved decisions

| Decision | Choice |
|----------|--------|
| Preferred items vs slot's 1–3 recipe limit | **Do not count** — added on top in Layer 3 |
| Macro tolerance | **±10%** (protein ±8g), no auto-relax |
| Preferences by day (training/rest) | **Always constant** |
| Plan enumeration | **Truly all** feasible plans (no cap) |
| Egg/whey count | Baked into recipe ("5 boiled eggs", "1 whey shake"), qty always 1 |
| Preference storage | Structured `meal_preferences` table (recipe + slot) |
```
