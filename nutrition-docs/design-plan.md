# NutriPal — Design Plan

> Source of truth for UI/UX direction, visual theme, and screen-by-screen build order.  
> Derived from `nutri_pal_design.pen` (existing screens) + product spec in `meal-planner-plan.md`.

---

## 1. Design Theme

### 1.1 Brand Identity

**App name:** NutriPal  
**Tagline:** Indian diet clean-bulk tracking  
**Personality:** Clean, health-forward, calm confidence. Not sterile like a hospital app — warm enough to feel personal, precise enough to feel trustworthy.

---

### 1.2 Color Palette

| Role | Hex | Usage |
|------|-----|-------|
| **Primary Green** | `#16A34A` | CTAs, active tab, calorie numbers, avatar, icons, macro card fill |
| **App Background** | `#F8FAFC` | Screen background (all screens) |
| **Card / Surface** | `#FFFFFF` | Meal cards, input fields, bottom bar |
| **Text — Primary** | `#1E293B` | Headers, card titles, greeting |
| **Text — Secondary** | `#64748B` | Subtitles, body, recipe descriptions |
| **Text — Muted** | `#94A3B8` | Inactive tabs, placeholders, helper text |
| **Text — Deep** | `#0F172A` | App name/logo |
| **Green Tint (Breakfast)** | `#F0FDF4` | Breakfast slot icon background |
| **Amber Tint (Lunch)** | `#FFFBEB` | Lunch slot icon background |
| **Orange Tint (Snack)** | `#FFF7ED` | Snack slot icon background |
| **Blue Tint (Dinner)** | `#F0F9FF` | Dinner slot icon background |
| **White on Green** | `#FFFFFF` | Text inside macro card |
| **Frosted White** | `#ffffffcc` | Subdued text on green card (80% opacity) |
| **Frosted Tile** | `#ffffff33` | Stat tiles inside macro card (20% opacity) |

---

### 1.3 Typography

**Font Family:** Inter (single typeface throughout)

| Style | Size | Weight | Usage |
|-------|------|--------|-------|
| App Title | 30px | 700 | Logo on login screen |
| Screen Header | 24px | 700 | "My Recipes", "Settings" page titles |
| Section Header | 18–20px | 700 | "Today's Meals", greeting |
| Macro Hero | 34px | 700 | Calorie % inside macro card |
| Card Title | 15–16px | 600 | Meal slot name (Breakfast, Lunch…) |
| Body | 14px | 400 | Descriptions, subtitles |
| Label / Tag | 12–13px | 500–600 | Calorie amounts, filter chips, input labels |
| Tab Label | 11px | 500–600 | Bottom nav labels |

---

### 1.4 Spacing & Layout

| Token | Value | Notes |
|-------|-------|-------|
| Screen width | 390px | iPhone standard |
| Screen height | 844px | |
| Horizontal padding | 20px | Screen-level sections |
| Card padding | 14–24px | Inner card padding |
| Gap (list items) | 12px | Between meal cards / recipe cards |
| Gap (elements in card) | 4–14px | Within card content |
| Section gap | 8–12px | Between sections on screen |

---

### 1.5 Corner Radii

| Element | Radius |
|---------|--------|
| Macro hero card | 20px |
| Meal / recipe cards | 16px |
| Meal slot icon box | 14px |
| Input fields, CTA buttons | 12px |
| Filter pills (active/inactive) | 18px |
| Avatar circle | 20px (40px size → perfect circle) |
| Stat tiles in macro card | 12px |

---

### 1.6 Shadows & Elevation

| Element | Shadow |
|---------|--------|
| Input cards | `y:1, blur:4, color:#00000008` (very soft) |
| Primary CTA button | `y:4, blur:16, color:#16A34A20` (green glow) |
| No shadow on meal cards | Pure white on light background reads cleanly |

---

### 1.7 Iconography

- **Icon library:** Lucide (e.g. `leaf` on login screen)
- **Meal slot icons:** Emoji (☀️ breakfast, 🍛 lunch, 🥜 snack, 🌙 dinner)
- **Bottom nav:** Emoji (🏠 📅 📖 ⚙️)
- **Icon size in meals:** 24px emoji inside 52×52px tinted rounded box (radius 14px)

---

### 1.8 Component Patterns

**Macro Hero Card**  
Full-width, green fill (`#16A34A`), corner radius 20px, padding 24×20px. Contains: title, large % number + kcal sub-label, row of 3 frosted stat tiles (Protein / Carbs / Fat).

**Meal Slot Card**  
White fill, corner radius 16px, height 80px, padding 14px, horizontal layout. Left: 52×52 tinted icon box (color varies by slot). Center: name + description (vertical, gap 4px). Right: calorie amount in green.

**Input Field**  
White card, corner radius 12px, padding 16px, soft drop shadow. Contains label (13px, 600 weight, `#475569`) above input row with icon.

**Primary Button**  
Green fill `#16A34A`, corner radius 12px, height 52px, white label 16px 700 weight, green glow shadow.

**Filter Pill**  
Height 36px, padding 0×16px, radius 18px. Active: green fill + white text. Inactive: white fill + `#64748B` text.

**Bottom Navigation Bar**  
White fill, height 72px, padding 0×20px. 4 tabs evenly spaced. Active tab: green label. Inactive: muted `#94A3B8`.

**Search Bar**  
White fill, corner radius 12px, height 48px, padding 0×14px. Placeholder text in `#94A3B8`.

**Avatar**  
40×40px circle, green fill, white initial letter (16px, 600 weight).

---

## 2. Screens & Flows to Build

Each item below is a discrete design unit to be created in Pencil as a separate top-level frame (390×844px).

---

### Screen 1 — Login *(already exists)*

**Status:** Done in `nutri_pal_design.pen`  
**Notes:** App logo + leaf icon, email/password fields, Login button with glow, server/offline copy.

---

### Screen 2 — Onboarding (Step 1 of 3): Personal Details

**Purpose:** Collect basic biometrics.  
**Fields:** Name, Sex (segmented control), Age, Height (cm), Weight (kg)  
**CTA:** "Next →"  
**Pattern:** Step indicator (dots or 1/3 label), clean form, input cards matching login style.

---

### Screen 3 — Onboarding (Step 2 of 3): Goals & Activity

**Purpose:** Set activity level, training days, surplus, macro preferences.  
**Fields:** Activity level (card selector), Training days/week (stepper), Calorie surplus (slider or input, 200–400), Protein g/kg (slider, 1.6–2.2), Fat % (slider, 20–30%)  
**CTA:** "Next →"

---

### Screen 4 — Onboarding (Step 3 of 3): Macro Summary

**Purpose:** Show computed macro targets, let user confirm or tweak.  
**Content:** Computed TDEE, target kcal, Protein / Carbs / Fat in grams. Inline warning if carbs < 100g.  
**CTA:** "Start Tracking"

---

### Screen 5 — Today *(already exists)*

**Status:** Done in `nutri_pal_design.pen`  
**Notes:** Greeting + avatar, macro hero card, 4 meal slot cards, bottom nav.  
**Missing/TODO:** Pre-meal edit action per card, skip action, "AI" vs "Offline rules" plan source badge.

> **UX principle:** The generated plan is the default — no manual approval needed. Meals are treated as planned and will happen unless the user edits or skips before eating. The app trusts the engine.

---

### Screen 5a — Today (Pre-Meal Edit State)

**Purpose:** Show the Today screen with inline edit actions on a single meal card (tapped/long-pressed before eating).  
**Additions:**
- Each meal card has a subtle edit icon (pencil) visible on the right alongside the calorie count.
- Tapping a card before eating reveals two quick actions: **"Swap"** (pick a different recipe) and **"Skip"** (remove from today's plan).
- No "Approve" button anywhere — the plan is live by default.
- **Status badge:** Small pill at top showing "AI Plan" (green) or "Offline Rules" (amber) to indicate how the plan was generated.
- **Consumed state:** Once the meal time passes (or user marks eaten), the card shifts to a muted/completed visual — no edit lock, just a visual cue.

---

### Screen 5b — Today (Inventory Alert State) *(M2)*

**Purpose:** Show the Today screen with a bulk-order banner when ≥ 4 inventory items have breached their restock threshold.

**Banner placement:** Between the macro hero card and the first meal slot card.

**Banner anatomy:**
- Background: Amber tint `#FFFBEB`, border `#FDE68A`, corner radius 12px, padding 12×16px.
- Left: shopping cart icon (Lucide `shopping-cart`, 20px, amber `#D97706`).
- Center: headline "Time to stock up" (14px, 600 weight, `#92400E`) + sub-label "X items running low · next 3-day plan" (12px, 400 weight, `#B45309`).
- Right: "View →" text link (13px, 600 weight, `#D97706`).
- Tapping anywhere on the banner navigates to the **Shop screen**.

**Dismissal:** Banner disappears automatically once the restocked item count drops below 4 (after user marks items restocked in Shop screen). No manual dismiss option — the signal should stay visible until acted on.

**States to show:**
1. Banner absent (< 4 items low) — normal Today layout.
2. Banner present (≥ 4 items low) — banner inserted between macro card and meal slots, pushing slots down.

---

### Screen 13 — Shop *(M2)*

**Purpose:** Review the computed 3-day shopping list, check off bought items, and mark restocked quantities.

**Header:** "Shopping List" title + "Next 3 days · X items" sub-label (muted).

**List layout:** Grouped by tier in order — Fresh → Perishable → Staple → Frozen. Each group has a section header pill (e.g. "🥬 Fresh — restock every 3 days").

**Each item row:**
- Left: checkbox (unchecked = needs buying).
- Center: food name (15px, 600) + deficit quantity below (13px, muted, e.g. "Need 300g more").
- Right: reason badge — "Plan" (blue tint) or "Low stock" (amber tint), 11px pill.

**Footer CTA:** "Mark checked as restocked →" — opens a bottom sheet per checked item to enter restocked quantity (g or ml input). Confirms and updates inventory.

**Empty state:** "All stocked up for the next 3 days" with a green checkmark illustration.

---

### Screen 6 — Week View

**Purpose:** 7-day horizontal grid; tap a day to go to that day's plan.  
**Layout:** Week strip at top (Mon–Sun), each day shows: date, total kcal, and a simple "planned" or "done" indicator (not an approval status — just whether the day has a plan).  
**Selected day:** expands or links to Today-style slot list.  
**CTA:** "Generate New Plan" button.

---

### Screen 7 — Generate Plan

**Purpose:** Trigger plan generation; show source badge.  
**Content:** Brief summary of macro targets, date range picker (1 day vs 7 days), generate button.  
**States:** Idle → Loading (spinner with "Asking your home server…") → Done (success + redirect to Week).  
**Badge:** "AI via Home Server" or "Offline Rules" depending on connectivity.

---

### Screen 8 — Recipes List *(already exists)*

**Status:** Done in `nutri_pal_design.pen` (header + search + filter pills + empty recipe list).  
**Missing/TODO:** Populated recipe cards in the list, + Add Recipe FAB button.

---

### Screen 8a — Recipes List (Populated)

**Purpose:** Show actual recipe cards in the list.  
**Recipe card:** White card, recipe name (15px 600), slot tags (breakfast/lunch/snack/dinner as small colored pills), macro summary row (kcal, protein, carbs, fat in small chips), prep time.  
**FAB:** Green circular button bottom-right with "+" icon.

---

### Screen 9 — Recipe Editor

**Purpose:** Create / edit a recipe.  
**Sections:**
1. **Meta** — Name, Description, Slot types (multi-select pills), Servings, Prep time, Cook time, Tags, Active toggle
2. **Ingredients** — List rows: food name + amount (g or ml). Oils are added here like any other ingredient. "Add ingredient" row.
3. **Instructions** — Ordered step list. Add/remove steps.
4. **Macro Preview** — Sticky card at bottom: computed kcal / protein / carbs / fat per serving.

**CTA:** "Save Recipe" button.

---

### Screen 10 — Food Library

**Purpose:** Search and browse food items; add custom foods.  
**Layout:** Search bar, category filter row (All / Protein / Carb / Fat / Vegetable / Oil…), food list rows (name + per 100g macros).  
**FAB:** "Add Food" button.  
**Food row:** Name, category badge, macros in muted text.

---

### Screen 11 — Add / Edit Food Item

**Purpose:** Create a custom food or edit an existing one.  
**Fields:** Name, Unit type (solid/liquid toggle), Category, Calories per 100g/ml, Protein, Carbs, Fat, is Oil toggle.  
**CTA:** "Save Food"

---

### Screen 12 — Settings

**Purpose:** Profile, server config, macro overrides, sync status.  
**Sections:**
1. **Profile** — Name, photo/avatar, biometrics (height, weight, age, sex)
2. **Goal & Macros** — Activity, training days, surplus, macro overrides
3. **Meal Times** — Breakfast/lunch/snack/dinner time pickers
4. **Server** — URL input, connection status indicator (green dot / red dot)
5. **Sync** — Last synced timestamp, Manual sync button, Sync queue count
6. **Dietary** — Tags (vegetarian etc.), Allergies

---

## 3. Flow Map

```
App Launch
├── First time → Onboarding Step 1 → Step 2 → Step 3 (Macro Summary) → Today
└── Returning  → Login → Today

Today (Tab 1)
├── Tap meal card (before eating) → inline Swap / Skip actions
├── Plan is live by default — no approval step
├── Tap "Generate"   → Generate Plan screen
└── ≥4 items below threshold → Inventory alert banner appears
    └── Tap banner   → Shop screen (Screen 13)

Week (Tab 2)
├── Tap day          → Today view for that day
└── Generate Plan    → Generate Plan screen

Recipes (Tab 3)
├── Tap recipe       → Recipe Editor (edit mode)
└── Tap "+"          → Recipe Editor (create mode)
    └── Ingredient row → Food Library (pick food)

Settings (Tab 4)
└── Server URL       → edit + test connection

Food Library (nested within Recipe Editor)
└── "Add Food"       → Add Food Item screen
```

---

## 4. Build Order

| # | Screen | Priority | Notes |
|---|--------|----------|-------|
| 1 | Login | Done | Exists |
| 2 | Today (base) | Done | Exists |
| 3 | Recipes List (base) | Done | Exists |
| 4 | Today (Pre-Meal Edit State) | High | Swap/skip before eating; no approval flow |
| 5 | Week View | High | Core navigation |
| 6 | Generate Plan | High | Key user action |
| 7 | Recipe Editor | High | Core data entry |
| 8 | Recipes List (Populated) | High | Needs recipe cards |
| 9 | Settings | Medium | Required for server config |
| 10 | Food Library | Medium | Used inside Recipe Editor |
| 11 | Add / Edit Food Item | Medium | CRUD utility |
| 12 | Onboarding Step 1 | Medium | Personal details |
| 13 | Onboarding Step 2 | Medium | Goals & activity |
| 14 | Onboarding Step 3 | Medium | Macro summary |
| 15 | Today (Inventory Alert State) | M2 | Amber banner ≥4 items; links to Shop |
| 16 | Shop screen | M2 | 3-day shopping list; grouped by tier; restock flow |

---

*End of design plan. Update build order as screens are completed in Pencil.*
