# DATA_DICTIONARY.md
Meal Planner App — Bubble Database Schema

---

## Table 1: User
**Privacy Rules:** Only when Current User = This User

| Field Name  | Field Type | Constraints/Notes                  |
|-------------|------------|------------------------------------|
| first_name  | text       | Required                           |
| last_name   | text       | Required                           |
| age         | number     | Optional                           |
| sex         | option set | Options: male, female, other       |
| height_ft   | number     | Min: 3, Max: 8                     |
| height_in   | number     | Min: 0, Max: 11                    |
| weight_lb   | number     | Min: 0, Decimal places: 2          |
| Created Date| date       | Auto-generated                     |

**Note:** Do *not* add an `email` field — Bubble’s built-in User type already has one.

---

## Table 2: NutritionTarget
**Privacy Rules:** Only when Current User = This NutritionTarget's user

| Field Name   | Field Type | Constraints/Notes                          |
|--------------|------------|--------------------------------------------|
| user         | User       | Required, Delete when parent is deleted    |
| calories     | number     | Required                                   |
| protein_g    | number     | Required                                   |
| carbs_g      | number     | Required                                   |
| fat_g        | number     | Required                                   |
| tolerance_pct| number     | Default: 10                                |

---

## Table 3: Food
**Privacy Rules:** Everyone can view this (seed data)

| Field Name          | Field Type | Constraints/Notes                |
|---------------------|------------|----------------------------------|
| name                | text       | Required                         |
| brand               | text       | Optional                         |
| serving_size_g      | number     | Required (grams only)            |
| serving_desc        | text       | Optional (use only if descriptive in grams, e.g. “per 170 g container”) |
| calories_per_serving| number     | Required                         |
| protein_g           | number     | Required, Decimal places: 2      |
| carbs_g             | number     | Required, Decimal places: 2      |
| fat_g               | number     | Required, Decimal places: 2      |

---

## Table 4: Recipe
**Privacy Rules:** Everyone can view this (for MVP seed recipes)

| Field Name       | Field Type | Constraints/Notes          |
|------------------|------------|----------------------------|
| name             | text       | Required                   |
| instructions     | long text  | Optional                   |
| default_servings | number     | Default: 1                 |

**Note:** Do not store calories/protein/carbs/fat here. These are computed from ingredients.

---

## Table 5: RecipeIngredient
**Privacy Rules:** Everyone can view this

| Field Name | Field Type | Constraints/Notes                               |
|------------|------------|-------------------------------------------------|
| recipe     | Recipe     | Required, Delete when parent is deleted         |
| food       | Food       | Required                                        |
| qty_g      | number     | Required, in grams only                         |

---

## Table 6: FavoriteFood
**Privacy Rules:** Only when Current User = This FavoriteFood's user

| Field Name | Field Type | Constraints/Notes                          |
|------------|------------|--------------------------------------------|
| user       | User       | Required, Delete when parent is deleted    |
| food       | Food       | Required                                   |

**Unique Constraint:** user + food — enforce in workflow (Bubble doesn’t support composite keys).

---

## Table 7: MealPlan
**Privacy Rules:** Only when Current User = This MealPlan's user

| Field Name      | Field Type | Constraints/Notes                       |
|-----------------|------------|-----------------------------------------|
| user            | User       | Required, Delete when parent is deleted |
| week_start_date | date       | Required                                |
| is_approved     | yes/no     | Default: no                             |
| Created Date    | date       | Auto-generated                          |

---

## Table 8: Meal
**Privacy Rules:** Only when Current User = This Meal’s meal_plan’s user

| Field Name | Field Type | Constraints/Notes                                |
|------------|------------|--------------------------------------------------|
| meal_plan  | MealPlan   | Required, Delete when parent is deleted          |
| day_index  | number     | Required, Min: 0, Max: 6                         |
| slot       | option set | Options: breakfast, lunch, dinner                |

**Unique Constraint:** meal_plan + day_index + slot.

---

## Table 9: MealComponent
**Privacy Rules:** Only when Current User = This MealComponent’s meal’s meal_plan’s user

| Field Name | Field Type | Constraints/Notes                                      |
|------------|------------|--------------------------------------------------------|
| meal       | Meal       | Required, Delete when parent is deleted                |
| kind       | option set | Options: food, recipe                                  |
| ref_food   | Food       | Optional, required if kind = food                      |
| ref_recipe | Recipe     | Optional, required if kind = recipe                    |
| servings   | number     | Required, Decimal places: 2                            |

---

## Option Sets

### 1. Sex
- male
- female
- other

### 2. MealSlot
- breakfast
- lunch
- dinner

### 3. ComponentKind
- food
- recipe

---

## Key Notes

- **All recipes and grocery lists are grams-only.** No cups/tbsp in MVP.  
- **Heights/weights**: stored as ft/in and lb; convert to metric on the fly for calculations.  
- **Favorites**: enforce uniqueness (user + food) in workflow.  
- **Privacy**:  
  - User, NutritionTarget, MealPlan, Meal, MealComponent, FavoriteFood → only accessible by their owner.  
  - Food (and optionally seed Recipes) → public read.  
- **Backups**: export Food and RecipeIngredient via CSV for safety.  
- **Search performance**: index Food `name`; MealPlan `user + week_start_date`.  

---
