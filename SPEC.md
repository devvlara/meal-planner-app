# Meal Planner App — MVP Technical Spec & Ship Plan

> **Goal:** Ship a web-first MVP that lets a user create an account, enter profile + macro targets, pick favorite foods, auto-generate a 7‑day plan of 3 meals/day that hits their daily targets, then approve and receive per‑meal grocery lists (and an optional aggregated weekly list).

---

## 1) MVP Scope (what ships)
- **Auth & Profile:** First name, last name, email (login), age, sex, **height (ft & in)**, **weight (lb)**.
- **Targets:** User enters daily **calories** and **macros** (protein, carbs, fat). We don’t compute TDEE in MVP; optional later.
- **Favorites:** User selects favorite foods (and optional recipes) from a curated database; each has per‑serving nutrition.
- **Planner:** Generate 7 days × 3 meals/day. Each day’s total ≈ user’s macro & calorie targets within a tolerance (e.g., ±5–10%).
- **Review & Approve:** User can approve all or regenerate a day/meal.
- **Grocery Lists:** Generate **per‑meal** grocery lists; also offer **weekly aggregated** list (deduped & unit‑normalized).
- **Persistence:** Save the plan and approvals.

**Out of scope (Phase 2+):** auto‑TDEE calc, dietary restrictions, advanced recipe builder, barcode scan, pantry tracking, substitutions marketplace, smart leftovers, cost optimization.

---

## 2) Success Criteria
- New user to grocery list in ≤ **5 minutes** (happy path).
- ≥ **90%** of plans hit targets within ±10% tolerance on all macros & calories.
- Planner runtime per user ≤ **2 seconds** for a full week (server‑side).

---

## 3) High-Level Architecture
- **Frontend:** Next.js (App Router) + TypeScript + Tailwind. Server Actions for form posts. Minimal state via React Query.
- **Backend:** Next.js API routes (or tRPC) with **Prisma** ORM.
- **DB:** Postgres (Supabase/Neon). Single region. Row‑level security for user data.
- **Auth:** NextAuth (email magic link or OAuth), or Supabase Auth.
- **Planner Engine:** Deterministic heuristic (TypeScript) with optional ILP solver later.
- **File Storage:** None in MVP.
- **Infra:** Vercel for app; DB on Supabase/Neon. Observability with Vercel Analytics + basic log drains.

---

## 4) Data Model (ERD sketch)

**Entities**
- `User` (id, firstName, lastName, email, age, sex, heightFt, heightIn, weightLb, createdAt)
- `NutritionTarget` (id, userId FK, calories, proteinG, carbsG, fatG, tolerancePct)
- `Food` (id, name, brand?, servingSizeG, servingDesc, caloriesPerServing, proteinG, carbsG, fatG)
- `Recipe` (id, name, instructions?, caloriesPerServing, proteinG, carbsG, fatG, defaultServings)
- `RecipeIngredient` (id, recipeId FK, foodId FK, qty, unit) — used to compute recipe macros
- `FavoriteFood` (id, userId FK, foodId FK)
- `MealPlan` (id, userId FK, weekStartDate)
- `Meal` (id, mealPlanId FK, dayIndex 0–6, slot ENUM['breakfast','lunch','dinner'])
- `MealComponent` (id, mealId FK, kind ENUM['food','recipe'], refId, servings Decimal(6,2))

**Notes**
- Store macros **per serving** for foods/recipes; planner chooses fractional servings.
- Keep **grams** as canonical for unit math; show friendly units on UI.

**Prisma-style schema snippets**
```sql
-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY,
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  age INT,
  sex TEXT CHECK (sex IN ('male','female','other')),
  height_ft SMALLINT CHECK (height_ft BETWEEN 3 AND 8),
  height_in SMALLINT CHECK (height_in BETWEEN 0 AND 11),
  weight_lb DECIMAL(6,2) CHECK (weight_lb > 0),
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Targets
CREATE TABLE nutrition_targets (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  calories INT NOT NULL,
  protein_g INT NOT NULL,
  carbs_g INT NOT NULL,
  fat_g INT NOT NULL,
  tolerance_pct INT DEFAULT 10
);

-- Foods (curated list)
CREATE TABLE foods (
  id UUID PRIMARY KEY,
  name TEXT NOT NULL,
  brand TEXT,
  serving_size_g INT NOT NULL,
  serving_desc TEXT,
  calories_per_serving INT NOT NULL,
  protein_g DECIMAL(6,2) NOT NULL,
  carbs_g DECIMAL(6,2) NOT NULL,
  fat_g DECIMAL(6,2) NOT NULL
);

-- Favorites
CREATE TABLE favorite_foods (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  food_id UUID REFERENCES foods(id) ON DELETE CASCADE,
  UNIQUE (user_id, food_id)
);

-- Plans & Meals
CREATE TABLE meal_plans (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  week_start_date DATE NOT NULL
);

CREATE TABLE meals (
  id UUID PRIMARY KEY,
  meal_plan_id UUID REFERENCES meal_plans(id) ON DELETE CASCADE,
  day_index SMALLINT NOT NULL CHECK (day_index BETWEEN 0 AND 6),
  slot TEXT NOT NULL CHECK (slot IN ('breakfast','lunch','dinner'))
);

CREATE TABLE meal_components (
  id UUID PRIMARY KEY,
  meal_id UUID REFERENCES meals(id) ON DELETE CASCADE,
  kind TEXT NOT NULL CHECK (kind IN ('food','recipe')),
  ref_id UUID NOT NULL,
  servings DECIMAL(6,2) NOT NULL
);
```sql
-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY,
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  age INT,
  sex TEXT CHECK (sex IN ('male','female','other')),
  height_ft SMALLINT CHECK (height_ft BETWEEN 3 AND 8),
  height_in SMALLINT CHECK (height_in BETWEEN 0 AND 11),
  weight_lb DECIMAL(6,2) CHECK (weight_lb > 0),
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Targets
CREATE TABLE nutrition_targets (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  calories INT NOT NULL,
  protein_g INT NOT NULL,
  carbs_g INT NOT NULL,
  fat_g INT NOT NULL,
  tolerance_pct INT DEFAULT 10
);

-- Foods (curated list)
CREATE TABLE foods (
  id UUID PRIMARY KEY,
  name TEXT NOT NULL,
  brand TEXT,
  serving_size_g INT NOT NULL,
  serving_desc TEXT,
  calories_per_serving INT NOT NULL,
  protein_g DECIMAL(6,2) NOT NULL,
  carbs_g DECIMAL(6,2) NOT NULL,
  fat_g DECIMAL(6,2) NOT NULL
);

-- Favorites
CREATE TABLE favorite_foods (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  food_id UUID REFERENCES foods(id) ON DELETE CASCADE,
  UNIQUE (user_id, food_id)
);

-- Plans & Meals
CREATE TABLE meal_plans (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  week_start_date DATE NOT NULL
);

CREATE TABLE meals (
  id UUID PRIMARY KEY,
  meal_plan_id UUID REFERENCES meal_plans(id) ON DELETE CASCADE,
  day_index SMALLINT NOT NULL CHECK (day_index BETWEEN 0 AND 6),
  slot TEXT NOT NULL CHECK (slot IN ('breakfast','lunch','dinner'))
);

CREATE TABLE meal_components (
  id UUID PRIMARY KEY,
  meal_id UUID REFERENCES meals(id) ON DELETE CASCADE,
  kind TEXT NOT NULL CHECK (kind IN ('food','recipe')),
  ref_id UUID NOT NULL,
  servings DECIMAL(6,2) NOT NULL
);
```

---

### 4.a) Unit & Conversion Policy (Updated)
- **User inputs**: height as **feet + inches**, weight in **pounds**.
- **Storage**: keep raw `height_ft`, `height_in`, `weight_lb` on the user.
- **Computation**: when needed, compute metric on the fly: `height_cm = height_ft*30.48 + height_in*2.54`, `weight_kg = weight_lb*0.45359237`.
- **Recipes & ingredients**: **all quantities stored and displayed in grams**. (Household measures like cups/tbsp are out-of-scope for MVP.)
- **Grocery lists**: per‑meal and weekly lists shown in **grams**; optional secondary display (e.g., kg in parentheses) can be a toggle later.

---

## 5) Planner Algorithm (v1 Heuristic)
**Objective:** For each day, choose components for 3 meals that sum to targets within tolerance. Prefer favorites; avoid repetition; respect per‑meal calorie split (default 30/40/30 breakfast/lunch/dinner).

**Inputs**
- Targets: `calories, proteinG, carbsG, fatG, tolerancePct`
- Favorites list: foods/recipes with per‑serving macros
- Constraints: per‑meal calorie ratios, max repeats per week (default 2), max components per meal (e.g., ≤3)

**Steps**
1. **Precompute densities**: protein per 100 kcal, carbs per 100 kcal, fat per 100 kcal.
2. **Breakfast**: seed from high‑protein favorites; scale serving to hit ~30% daily calories, then adjust with carb/fat add‑ins.
3. **Lunch/Dinner**: greedy fill to close gaps; choose next component that minimizes macro distance `(w_p*Δp + w_c*Δc + w_f*Δf)` after scaling serving within [0.5, 2.0] servings.
4. **Local refine**: small random or hill‑climb swaps to improve fit.
5. **Validate**: ensure totals within tolerance; otherwise rerun with different seeds up to N tries (e.g., 20).
6. **Variety rule**: no meal repeats on adjacent days; cap any single favorite to 2 occurrences/week.

**Pseudocode**
```ts
function planWeek(favorites, targets) {
  const days = Array.from({ length: 7 }).map((_, d) => planDay(favorites, targets));
  enforceVariety(days);
  return days;
}

function planDay(F, T) {
  const splits = { breakfast: 0.3, lunch: 0.4, dinner: 0.3 };
  const meals = ['breakfast','lunch','dinner'].map(slot =>
    seedAndGreedy(F, T, splits[slot])
  );
  const refined = localRefine(meals, T);
  if (!withinTolerance(refined, T)) retry with different seeds;
  return refined;
}

function seedAndGreedy(F, T, calShare) {
  let meal = [];
  const targetCal = T.calories * calShare;
  // seed with high-protein favorite if protein gap is large
  meal.push(bestSeed(F, T));
  scaleServing(meal[0], targetCal);
  // add components to reduce macro distance
  while (components < 3 && calories(meal) < 0.95*targetCal) {
    const cand = argmin(F, f => macroDistance(T, currentTotalsWith(f)));
    meal.push(scaleToFit(cand, targetCal - calories(meal)));
  }
  return meal;
}
```

**Why heuristic first?** Fast, deterministic, and easy to ship. **Phase 2:** switch to **ILP** to guarantee optimality with constraints (GLPK/OR‑Tools via server worker) and to support more rules (dietary restrictions, cost minimization).

---

## 6) Grocery List Generation
**Goal:** After approval, output per‑meal lists and an optional weekly aggregated list.

**Rules (Updated)**
- **All recipe ingredients and grocery outputs are in grams.** No household measures in MVP.
- Convert every component to **grams** first.
- For recipes, expand `RecipeIngredients` → sum per meal × servings.
- Aggregate by `foodId` across the week.
- Display grams; optional secondary `(kg)` in UI can be a later toggle.
- Round to sensible granularities (e.g., nearest 5 g) and optionally to package sizes (Phase 2).

**Example per‑meal output**
```
Lunch (Tue):
- Chicken breast — 180 g
- Jasmine rice — 150 g (cooked)
- Olive oil — 10 g
```

**Example aggregated weekly output**
```
Chicken breast — 1260 g
Jasmine rice — 1050 g (cooked)
Olive oil — 140 g
Greek yogurt — 980 g
Blueberries — 420 g
```

---

## 7) API Design (REST)
**Auth**
- `POST /api/auth/register` → {firstName,lastName,email,password}
- `POST /api/auth/login`

**Profile & Targets**
- `GET /api/me` → user profile (returns `heightFt`, `heightIn`, `weightLb`)
- `PUT /api/me` → update profile fields `{ age, sex, heightFt, heightIn, weightLb }`
- `GET /api/targets` / `PUT /api/targets`

**Foods & Favorites**
- `GET /api/foods?search=chicken&page=1`
- `POST /api/favorites` → {foodId}
- `DELETE /api/favorites/:id`

**Planner**
- `POST /api/plan` → {weekStartDate}
  - Returns `mealPlanId` + plan payload
- `POST /api/plan/:id/approve`
- `GET /api/plan/:id/grocery?scope=meal|week`

**Response shape (excerpt)**
```json
{
  "mealPlanId": "...",
  "days": [
    {
      "date": "2025-09-08",
      "totals": {"cal": 2450, "p": 185, "c": 250, "f": 80},
      "meals": [
        {"slot":"breakfast","components":[{"type":"food","id":"chicken","servings":1.2}]},
        {"slot":"lunch","components":[...]},
        {"slot":"dinner","components":[...]}
      ]
    }
  ]
}
```

---

## 8) Frontend Flow (screens)
1. **Sign up / Login**
2. **Profile** (age, sex, **height ft/in pickers**, **weight lb** with number input)
3. **Targets** (calories & macros; show preview of macro split)
4. **Favorites Picker** (search foods, add to list)
5. **Review Plan** (7×3 grid; per‑day totals; regenerate a day)
6. **Approve** (CTA)
7. **Grocery Lists** (tab: per‑meal | aggregated week; **all grams**; print/export)

**UX notes**
- Always show **gap meters** (how close the day is to targets).
- Provide **one‑tap regenerate** for a day.
- Keep the favorites picker tight; sorting by protein density helps.

---

## 9) Seed Data Strategy
- Ship with a curated seed list of ~150 common foods & simple recipes (chicken, salmon, eggs, oats, rice, pasta, yogurt, nuts, fruits, oils, etc.) with per‑serving nutrition.
- Keep a **content JSON** that can be iterated quickly by non‑engineers.
- Add a minimal **admin import** for foods via CSV.

---

## 10) Validation & Edge Cases
- **Targets sanity check:** If user macros don’t sum near calories (`4*P + 4*C + 9*F`), warn & offer auto‑fix.
- **Favorites too small:** If < 10 favorites, prompt to add more for variety.
- **Impossible targets:** If the engine fails after N attempts, report the smallest adjustment needed to make targets feasible (e.g., +15g fat or −150 kcal).
- **Units:** Always compute in grams; display nicely.
- **Leftovers (Phase 2):** Allow intentional repeats to simplify grocery list.

---

## 11) Security & Privacy
- Store only necessary PII; encrypt at rest; RLS by `user_id`.
- No health claims or medical advice.
- Allow full account deletion (GDPR‑style) in settings.

---

## 12) QA & Metrics
**Tests**
- Planner: property‑based tests to ensure totals within tolerance.
- API: contract tests on endpoints.
- E2E: sign‑up → targets → favorites → plan → approve → grocery (Playwright).

**Metrics**
- Time to plan generation, plan approval conversion rate, regenerate count per day, macro hit rate.

---

## 13) Delivery Timeline (7‑Day Sprint)
**Day 1** – Project scaffold, auth, DB schema, seed foods
**Day 2** – Profile & targets UIs + APIs; validation
**Day 3** – Favorites picker + search
**Day 4** – Planner v1 (heuristic) + unit tests
**Day 5** – Plan review UI (7×3 grid), regenerate per day
**Day 6** – Approvals + grocery list (per‑meal & weekly aggregation)
**Day 7** – Polish, PDF/print export, deploy, basic analytics

---

## 14) Example Day (hits a 2400 kcal / 180P / 240C / 80F target ±5%)
- **Breakfast**: Greek yogurt (**300 g**), blueberries (**150 g**), granola (**60 g**)
- **Lunch**: Chicken breast (**180 g**), jasmine rice cooked (**200 g**), olive oil (**10 g**)
- **Dinner**: Salmon (**170 g**), roasted potatoes (**250 g**), mixed salad + dressing (**40 g**)

**Totals (illustrative)**: 2425 kcal; P 182 g; C 238 g; F 81 g → all within ±5%.

---

## 15) Stretch Goals (Phase 2)
- ILP/Linear solver for optimal macro fit with **cost/price** minimization.
- Dietary filters: vegan, vegetarian, halal/kosher, gluten‑free, nut‑free, lactose‑free.
- Pantry & leftovers modeling (reduce waste, batch cooking toggles).
- Price API to estimate weekly cost & allow budget constraints.
- Mobile app (Expo) with offline grocery lists.
- Barcode scanner; import from photo; wearable sync.

---

## 16) Dev Notes: Heuristic Details
- **Macro distance**: `D = w_p*(Δp)^2 + w_c*(Δc)^2 + w_f*(Δf)^2`, emphasize the **scarcest** macro (usually protein): set `w_p=2, w_c=1, w_f=1`.
- When scaling a candidate, clamp servings to [0.5, 2.0]. Compute fractional servings with calories as the pivot: `servings = clamp((targetMealCal - currentCal)/candCal, .5, 2)`.
- To enforce variety, keep a weekly counter per `foodId`; skip candidates above the cap.
- For speed, pre‑sort favorites by **protein density** and **calorie density**; cache.

---

## 17) Tiny TypeScript Module (planner core – sketch)
```ts
export type Macro = { cal: number; p: number; c: number; f: number };
export type Food = Macro & { id: string; name: string };

export function planDay(favs: Food[], T: Macro, splits = {b:0.3,l:0.4,d:0.3}){
  const meals = [splits.b, splits.l, splits.d].map(share => planMeal(favs, T, share));
  return meals;
}

function planMeal(favs: Food[], T: Macro, share: number){
  const target: Macro = { cal: T.cal*share, p: T.p*share, c: T.c*share, f: T.f*share };
  let comps: {food: Food; servings: number}[] = [];
  comps.push(seed(favs, target));
  scaleToCalories(comps[0], target.cal);
  while (comps.length < 3 && sumCal(comps) < 0.95*target.cal){
    const cand = bestNext(favs, target, comps);
    comps.push(scaleToFit({food:cand, servings:1}, target.cal - sumCal(comps)));
  }
  return comps;
}
```

---

## 18) Print/Export
- Print‑friendly CSS for grocery list.
- Export as PDF/CSV.

---

## 19) Open Questions (for later)
- Do we need household measures (cups/tbsp) vs grams only?
- Cost awareness: show estimated total? Budget constraint?
- How strict should variety rules be by default?

---

**That’s the whole plan.** This is deliberately biased for speed-to-ship while leaving clear upgrade paths for precision and features.

