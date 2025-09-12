# Meal Planner App (MVP)

This is the MVP spec and assets for a **Meal Planner App** that helps users hit daily calorie and macronutrient targets.  
The app generates **3 meals per day for a full week**, lets the user approve the plan, and then produces **per-meal and aggregated grocery lists** (all in grams).

---

## Features (MVP)

- **User Profiles**: First name, last name, email (Bubble built-in), age, sex, height (ft + in), weight (lb).  
- **Targets**: Daily calories and macros (protein, carbs, fat).  
- **Favorites**: Users select foods from a seeded Food table.  
- **Meal Plans**: Automatic 7-day × 3-meal planner that hits daily goals (±10%).  
- **Approvals**: Users can approve or regenerate their plan.  
- **Grocery Lists**: Generated in grams-only, per meal and weekly aggregate.  
- **Privacy**: Each user can only access their own profile, targets, meal plans, meals, and grocery lists. Food (and optionally Recipes) are globally readable.

---

## Repo Structure

```
meal-planner-app/
│
├── SPEC.md               # Full technical plan and app blueprint
├── DATA_DICTIONARY.md    # Bubble database schema (cleaned, grams-only)
├── FOODS_SEED.csv        # Seed food list (~120 items, grams-only serving sizes)
└── README.md             # This file
```

---


## Planner Logic (MVP Heuristic)

- Split daily calories 30/40/30 (breakfast/lunch/dinner).
- Start each meal with a protein-rich favorite.
- Add 1–2 foods to close macro gaps.
- Clamp servings to 0.5–2.0 per component.
- Retry until plan is within tolerance (±10% per macro).

(Full details in [`SPEC.md`](./SPEC.md).)

---

## Units & Conversions

- **Height**: stored in feet/inches.  
- **Weight**: stored in pounds.  
- **Conversions**:  
  - `height_cm = height_ft*30.48 + height_in*2.54`  
  - `weight_kg = weight_lb*0.45359237`  
- **Foods/Recipes/Grocery**: **grams-only** for storage and display.

---
