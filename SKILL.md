---
name: symphony
description: >-
  Order personalized meals from Symphony, a meal delivery service in Paris, France.
  Browse 250+ ingredients, discover recipes with nutritional filtering, manage baskets,
  view meal history, check nutritional intake, manage playlists, rate meals, and track credits.
  BEFORE any order: call GET /delivery/check to verify the user's postal code is in the
  delivery zone. Do not proceed with basket actions if not deliverable.
  Requires SYMPHONY_API_KEY from https://symphony.fr/en/my-account/account/api-keys
version: 3.0.0
metadata:
  openclaw:
    requires:
      env:
        - SYMPHONY_API_KEY
    primaryEnv: SYMPHONY_API_KEY
    emoji: "🍽️"
    homepage: https://symphony.fr
---

# Symphony API v3

[Symphony](https://symphony.fr) delivers fully personalized meals in Paris. Every dish has a unique recipe tailored to the customer's nutritional targets. Meals arrive ready to eat — just reheat.

**Base URL:** `https://symphony.fr/api/v3`
**Auth:** `Authorization: Bearer $SYMPHONY_API_KEY` on every authenticated request (or `?api_key=$SYMPHONY_API_KEY`).
**Language:** `?lang=en` or `?lang=fr` (default `fr`) on any endpoint.

## Response Format

All responses:
```json
{
  "success": true,
  "data": { ... },
  "errors": [],
  "meta": { "__response_format": "standard" }
}
```

On error: `{ "success": false, "errors": [{ "message": "...", "error_code": 400 }] }`.
All field names are `snake_case`. Most responses include a `next_actions` array — use those as hints for what to do next.

## First Interaction — Onboarding

When this skill is activated with a new customer (no prior orders or profile not yet configured), **onboard them step by step before doing anything else.** Do not list capabilities or describe what the API can do.

Onboarding sequence:
1. Explain what Symphony is — personalized meals delivered to their home, ready to reheat
2. Check delivery — ask for postal code, verify with `GET /delivery/check`; if not deliverable, stop
3. Dietary needs — allergies, intolerances, diet identity (vegan, halal…), dislikes
4. Appetite and goals — how much they eat, what they're optimizing for; set calorie target
5. Logistics — microwave, freezer space, meals per week
6. First recipes — browse a few matches, not the whole catalog

One step at a time. Wait for the user's answer before moving on.

---

## Agent Behavioral Rules

1. **Always check delivery first.** Before any basket action, call `GET /delivery/check`. If `deliverable` is false, tell the user and stop. Never skip this.
2. **Follow `next_actions`.** Every response includes suggestions. Use them to guide the conversation flow.
3. **Suggest rating after viewing past orders.** When a user reviews past baskets, prompt them to rate unrated meals.
4. **Check basket before adding.** Call `GET /me/meals/upcoming-basket` before adding meals to show what's already there.
5. **Use pagination.** List endpoints return pages. Always check `pagination.has_next_page` and offer to load more.
6. **Calories first, not grams.** Symphony sizes meals by calorie target. Do NOT pass `quantity_g` unless the user explicitly requests a specific gram amount. When omitted, the system auto-calculates the optimal portion: `grams = target_kcal / energy_density`, clamped to 280–650g. This ensures every meal hits the calorie target while staying in a comfortable eating range.
7. **Ask about appetite before setting targets.** Understand how much the user typically eats before choosing a calorie target. A person used to small meals may struggle with a 600g portion even if it's nutritionally right; a person used to large meals may feel unsatisfied with 300g even if it has enough calories. See the Best Practices section below.
8. **Nutrition is per 100g.** Scale by actual `quantity_g` when reporting absolute values to the user.
9. **Rate by meal ID, not recipe ID.** The rate endpoint uses the BuyingItemLabel `id` from past baskets, not the recipe's ID. Rating a meal automatically marks it as eaten if it wasn't already.
10. **Verify after PATCH /me.** After calling `PATCH /me`, always verify the update persisted by calling `GET /me`. Some fields may be silently dropped.
11. **Share what you learn.** When you discover something useful (a substitution pattern, an edge case, a nutritional insight), post it to `POST /knowledge`. Check `GET /knowledge/synthesis` at the start of your session to benefit from what other agents have learned.

---

## Symphony Best Practices (Advisory)

These are Symphony's recommendations for optimal meal planning. Follow them by default unless the customer explicitly wants something different — but explain the trade-offs.

### Calorie target drives portion size

Symphony computes portion grams from the user's calorie target and each recipe's energy density:
- `grams = target_kcal / energy_density_per_gram`
- Energy density is clamped to **1.4–2.2 kcal/g** to keep portions physically reasonable
- Resulting grams are clamped to **280–650g** (hard system limits)

If a user asks for a fixed gram amount instead, explain why calorie-fixed is preferable:
- A large low-density meal (e.g. 600g at 1.0 kcal/g = only 600 kcal) may leave them hungry later
- A small high-density meal (e.g. 300g at 2.5 kcal/g = 750 kcal) may be hard to finish and feel heavy
- Calorie-fixed ensures consistent energy intake regardless of recipe density

### Appetite guide

| Appetite | Typical calorie target | Typical portion range |
|----------|----------------------|----------------------|
| Small ("I eat light") | ~450 kcal | 280–350g |
| Normal | ~700 kcal | 350–500g |
| Large ("I eat a lot") | ~850–1000 kcal | 450–650g |

Above ~550g, check if the user is genuinely used to large meals. Below ~350g, recipe precision decreases.

### Per-meal nutritional guidelines

- **Vegetables:** aim for ≥110g per meal (≥160g is ideal)
- **Available carbs:** max 150g per meal (≤90g is ideal) — but zero carbs makes meals boring
- **Omega-6:** max 8g per meal (≤4.6g is ideal) — omega-6 is inflammatory. **Note:** For low-carb/high-fat meal plans, omega-6 per meal will typically exceed this guideline because achieving high calorie density from fat inherently involves omega-6-containing ingredients. Similarly, many vegan recipes have elevated omega-6 because plant protein sources come with significant vegetable fats. This is a structural reality of these diets, not an error the agent should try to fix.

### Reordering a previous meal

To reorder a meal from a previous order: call `GET /me/baskets` to find it, take the `recipe_id` from the meal, and `POST` it to `/me/meals/upcoming-basket`. No special endpoint needed.

### Per-week guidelines

- **Salmon:** max 3 meals per week (heavy metals accumulation risk and cost)
- **Variety:** diversify across protein sources, cuisine styles, and ingredient families over the week
- **World cuisines:** aim for diverse culinary traditions — don't default to the same style every time

### Personalize — don't just pick from the catalog

Symphony's value is personalization. When a recipe almost fits but doesn't quite hit the user's targets, **modify it** rather than settling for a bad match.

**Adjusting proportions (safe within ±20–25%):**
- To **increase calories** without changing weight: increase the main fat source (usually oil — sunflower, olive, sesame). Even a small fat increase raises calories significantly.
- To **decrease calories** (increase portion at lower kcal target): reduce the main fat source (oil, coconut cream, nuts). Redirect freed mass to vegetables (volume ingredients) — they add portion weight without adding much energy. See **Reducing energy density** below for candidate selection.
- To **increase protein**: increase the main protein source (meat, fish, legume proportion).
- To **reduce omega-6**: reduce sunflower oil, replace with olive oil if possible.
- To **adjust vegetables**: vegetables can be adjusted ±40% safely.

**Rules for safe modifications:**
- Don't change any single ingredient by more than **20–25%** of its proportion — beyond this, recipe quality may degrade unpredictably.
- Always start from **popular recipes** (sort by `most-popular`) — they are the most proven and most likely to be appreciated.
- Use the `POST /recipes` endpoint with `parent_recipe_id` to create a modified version.
- If you cannot match targets within safe modification limits, report this honestly rather than serving a meal that doesn't meet the user's needs.

**After any modification, always verify:** calories fit within portion limits (see Hard Limits), sodium is within range (see Sodium Management), omega-6 is under 8g/meal (see Per-meal Guidelines), protein meets the customer's minimum, and the dish is still recognizable as what the customer asked for.

**Never accept a poor match.** If the user asks for 1000 kcal and a recipe delivers 800 kcal at max portion, either modify the recipe (increase fat) or find a better candidate. Delivering meals significantly below the calorie target leaves users hungry — that's unacceptable.

#### Reducing energy density (low-calorie targets)

When a customer has a low calorie target (e.g. 500 kcal), the binding constraint is the **280g minimum portion**. The critical threshold is: `max_energy_density = target_kcal / 280`. For 500 kcal, that's 178.6 kcal/100g — any recipe above this gets clamped to 280g and delivers MORE calories than requested.

**Candidate selection:** Start with recipes at **140–175 kcal/100g** energy density. Recipes above ~190 kcal/100g (starch-heavy: gnocchi, pasta-dominant, risotto) usually cannot be reduced enough by fat removal alone — the starchy base is inherently energy-dense. Don't waste API calls trying.

**What to reduce:** Target the main fat source — sunflower oil (sid 66, ~880 kcal/100g), olive oil (sid 38, ~830 kcal/100g), coconut cream (sid 44, ~180 kcal/100g), pine nuts (sid 54, ~670 kcal/100g). Reduce by up to 25%.

**Where to redirect freed mass:** Always to volume vegetables (zucchini, spinach, carrots, yellow beans) — they add portion weight at ~20–40 kcal/100g, lowering overall energy density. Never redirect to starch or protein (they're more energy-dense).

**Verify after reduction:** Recalculate `portion_g = target_kcal / (energy_per_100g / 100)`. If portion is still ≤280g, the recipe cannot serve this calorie target — move on to a better candidate.

#### Recovering from low energy density (< 1.4 kcal/g)

When a recipe's energy density drops below 1.4 kcal/g (common in lean fish dishes after protein boosting), the system clamps to 1.4 and the calorie target becomes unreachable even at 650g max portion. Fix: add 1–2% olive oil (sid 38) to bring energy density above 1.4 without changing dish character. For fish recipes, olive oil is usually the best choice.

### Sodium/salt management — adjust salt to hit the target range

**Salt (shortId 3) can be changed as much as needed — the ±25% per-ingredient modification limit does NOT apply to salt.** Salt is a pure seasoning lever.

**After any recipe modification, always adjust salt to target the CENTER of the user's recommended sodium range.** Don't leave sodium wherever it landed — actively set it.

**Sodium reference ranges (% of meal weight):**
- **Below 0.15%:** Very low — may taste bland
- **0.20–0.33%:** Normal (green zone) — default target for most customers
- **0.33–0.40%:** High — acceptable for inherently salty recipes (pesto, miso, teriyaki) or when the customer explicitly prefers salty food
- **Above 0.40%:** Too high — will cause complaints; avoid

The acceptable range shifts based on the customer's preferences. Salt-sensitive customers target **0.15–0.25%**. Customers who prefer salty food shift upward. Always check the profile.

**Do NOT filter or exclude recipes based on sodium.** Sodium is easy to fix by adjusting salt. A recipe with high sodium is not a bad recipe — it just needs its salt adjusted.

**When salt alone isn't enough:** If reducing salt to zero still leaves sodium above the target, reduce soy sauce (shortId 37) or tamari (shortId 285) by up to 20–25% (same limit as other ingredients). Tamari is functionally identical to soy sauce for sodium purposes — treat them the same way. This is usually enough. Beyond 25% reduction, soy sauce/tamari loss degrades the recipe — they provide **umami** (savory depth), not just sodium. Warn the user if further reduction is needed, and compensate with nutritional yeast or mushroom-based ingredients for umami without sodium.

**Extreme catalog sodium:** Some catalog recipes have salt proportions 5–10× normal (e.g. salt at 2.5% of recipe weight = ~1.12% sodium). This is not unusual — it just means someone was generous with salt. These recipes are good candidates for modification; just set salt to an appropriate level.

**The rule is simple:** check sodium → adjust salt to hit center of range → if still too high, reduce soy sauce/tamari up to 25% → verify.

### Ingredient substitution — dietary conversions

When a customer needs a dietary change (vegan, dairy-free, FODMAP, halal, etc.), create converted versions using `POST /recipes` with `parent_recipe_id`.

**Common safe substitution pairs:**

| Animal ingredient | Vegan substitute | Protein (per 100g) | Energy (kcal/100g) | Notes |
|-------------------|-----------------|-------------------|-------------------|-------|
| Ground beef 20% (shortId 48) | Vegan mince (shortId 280) | 17→24 g | 250→170 | Near-perfect 1:1 swap. Vegan mince has *higher* protein density. Omega-6: 3.8g/100g. |
| Pulled beef (shortId 180) | Vegan mince (shortId 280) | 28→24 g | 160→170 | Near 1:1 swap. Slight protein drop — boost 10–15% if needed. Omega-6: 3.8g/100g. |
| Pulled pork (shortId 125) | Vegan mince (shortId 280) | 26→24 g | 190→170 | Near 1:1 swap. Minimal protein drop. Omega-6: 3.8g/100g. |
| Chicken breast (shortId 42) | Tofu (shortId 210) | 31→12 g | 110→130 | Best for plant-heavy recipes (curry banane pattern). ⚠️ High omega-6 (5.4g/100g). |
| Chicken breast (shortId 42) | Veggie-sliced (shortId 278) | 31→21 g | 110→162 | Best for complex dishes (paella, risotto, tikka masala). 32% protein drop — boost by 25–40%. ⚠️ High sodium (0.67g/100g — 8× chicken): at 15% proportion contributes ~0.10% sodium; at 30%+ proportion contributes ~0.20%+ sodium, making targets below 0.25% impossible even with zero salt. **Do not use veggie-sliced for salt-sensitive customers (target <0.25%) — use tofu instead.** ⚠️ Very high omega-6 (9.2g/100g). |
| Cream (shortId 45) | Soy cream (shortId 279) | 2→3 g | 300→140 | Energy density halved. May need added oil to maintain calorie target. |
| Butter (shortId 46) | Sunflower oil or olive oil | 1→0 g | 720→880/830 | Use less oil than butter (butter ~80% fat, oil 100%). Choose based on cuisine. |
| Parmesan (shortId 55) / Emmental (shortId 67) | Gran veggiano (shortId 277) | 35/27→20 g | 430/370→250 | May need 2x proportion. Add nutritional yeast (shortId 43) at 1–2% for umami + protein. |

**Umami compensation (when removing MSG or strong meat flavors):** Add soy sauce (shortId 37) at 2–3% + nutritional yeast (shortId 43) at 0.8% + lemon juice (shortId 39) at 0.5–1%. This trio replaces the savory depth lost from meat or MSG removal.

After any substitution, **recompute sodium** — processed substitutes often carry more sodium than originals. Also check for tamari (shortId 285), soy sauce (shortId 37), and miso (shortId 189) in Asian recipes — these are major sodium contributors that may need reduction when combined with high-sodium vegan proteins. Always verify the actual ingredient list — catalog tags may be imprecise. Stay within the character of the original dish. Then verify against the guidelines in the Personalize section above.

**Diet-specific deep dives** — fetch the relevant page when the customer has specific dietary needs:
- **Vegan/plant-based:** `GET /knowledge/synthesis/vegan-diet` — omega-6 spike, energy density drop, sodium shift from processed vegan proteins
- **FODMAP:** `GET /knowledge/synthesis/fodmap-diet` — allium removal, parsnip replacement, salt direction (often increases)
- Other diets: check `GET /knowledge/synthesis` for available pages

### Hard limits

- **Maximum portion: 650g** — the system cannot produce meals above this weight
- **Minimum portion: 280g** — below this, recipe precision decreases significantly
- **Energy density: 1.4–2.2 kcal/g** — recipes outside this range get clamped, which means the actual calorie delivery may differ from the target

### Energy density clamp interaction

See **Recovering from low energy density** in the Personalize section above. When modifying recipes to increase protein (which is less energy-dense than fat), you may inadvertently lower energy density below the 1.4 kcal/g clamp. Add 1–2% olive oil to fix.

### Knowledge synthesis pages

Detailed knowledge on specific diets is available via `GET /knowledge/synthesis/{slug}`. Fetch relevant pages when the customer has specific dietary needs:

| Slug | When to fetch | What you'll find |
|------|--------------|-----------------|
| `vegan-diet` | Customer wants vegan/plant-based meals | Omega-6 spike, sodium shift, energy density drop, catalog tag warnings |
| `fodmap-diet` | Customer has IBS/FODMAP sensitivity | 4-step conversion pattern, allium IDs, salt direction |
| `global` | General knowledge check | All community learnings synthesized |

### All 35 nutrition properties

Every recipe and ingredient includes `nutrition_per_100g` with these properties: `energy`, `water`, `fiber`, `protein`, `fat`, `saturated-fat`, `monounsaturated-fat`, `polyunsaturated-fat`, `availableCarb`, `sugar`, `salt`, `sodium`, `calcium`, `iron`, `magnesium`, `phosphorus`, `potassium`, `zinc`, `vitamin-A`, `vitamin-C`, `vitamin-D`, `vitamin-E`, `vitamin-K`, `thiamin`, `riboflavin`, `niacin`, `vitamin-B6`, `vitamin-B12`, `folate`, `cholesterol`, `omega-6`, `PUFA_18-2_n-6`, `PUFA_18-4_n-6`, `PUFA_20-4_n-6`, `PUFA_20-5_n-3`.

All values are per 100g. Scale by `quantity_g / 100` for absolute amounts.

---

## Workflows

### "Can I get Symphony delivered?"

```bash
curl -s "https://symphony.fr/api/v3/delivery/check?postal_code=75011"
```

```json
{
  "data": {
    "postal_code": "75011",
    "deliverable": true,
    "delivery_methods": [
      {
        "method": "doorstep",
        "shipping_cost_eur": 10,
        "free_shipping_threshold_eur": 40,
        "min_meals": 1,
        "days_between_order_and_delivery": 1,
        "possible_delivery_days": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]
      }
    ],
    "next_actions": ["check_delivery_dates", "browse_recipes"]
  }
}
```

If deliverable, get available dates:

```bash
curl -s "https://symphony.fr/api/v3/delivery/dates?postal_code=75011&weeks=2"
```

```json
{
  "data": {
    "postal_code": "75011",
    "delivery_dates": {
      "doorstep": [
        { "date": "2026-05-29", "day": "Friday" },
        { "date": "2026-06-01", "day": "Monday" }
      ]
    },
    "next_actions": ["browse_recipes", "add_to_basket"]
  }
}
```

Query params: `postal_code` (required), `method` (filter to one method), `weeks` (1–8, default 4).

---

### "Find me something to eat"

Discover recipes with filtering and search:

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/recipes/discover?lang=en&limit=10&sort_by=most-popular"
```

```json
{
  "data": {
    "recipes": [
      {
        "id": "6336dbba6ac970d8e808e862",
        "title": "Curry de pois chiches aux épinards",
        "nutrition_per_100g": { "energy": 123.34, "protein": 5.02, "fiber": 4.32, "..." : "35 properties" },
        "image": "https://symphony.fr/cdn-cgi/imagedelivery/.../w=256,h=256"
      }
    ],
    "pagination": { "has_next_page": true, "cursor": "63ce63c1...|1.999" },
    "next_actions": ["get_recipe_detail", "add_to_basket", "add_to_playlist"]
  }
}
```

**Query params:**

| Param | Type | Description |
|-------|------|-------------|
| `sort_by` | string | `most-popular` (default), `trending`, `recent` |
| `limit` | int | 1–50 (default 20) |
| `search` | string | Free-text search |
| `tags` | string | Comma-separated tags (e.g. `vegetarian,high-protein`) |
| `include_ingredients` | string | Comma-separated ingredient identifiers to require. Accepts both string IDs (e.g. `salmon`) and numeric `short_id` values (e.g. `85`). |
| `exclude_ingredients` | string | Comma-separated ingredient identifiers to exclude. Accepts both string IDs (e.g. `pulled-pork`) and numeric `short_id` values (e.g. `125`). |
| `nutrition_filter` | string | Filter by any nutrition property. Comma-separated `property:min:max` expressions. Supports all 35 properties (see All 35 Nutrition Properties). Empty min/max = unbounded. Examples: `protein:20:50` (protein 20–50g/100g), `sodium::0.3` (sodium ≤0.3g/100g), `availableCarb:0:15,protein:25:` (carbs 0–15 AND protein ≥25). Invalid property names return 400 with the full list. |
| `timescale` | int | For `most-popular`: 1, 7, 30, or 365 days |
| `cursor` | string | Pagination cursor from previous response |
| `lang` | string | `en` or `fr` |

Get full recipe detail:

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/recipes/6336dbba6ac970d8e808e862?lang=en"
```

```json
{
  "data": {
    "id": "6336dbba6ac970d8e808e862",
    "title_fr": "Curry de pois chiches aux épinards",
    "title_en": "Curry de pois chiches aux épinards",
    "ingredients": [
      { "id": "chickpeas", "short_id": 14, "title_en": "chickpeas", "proportion": 0.33587 },
      { "id": "spinach", "short_id": 26, "title_en": "spinach", "proportion": 0.15 }
    ],
    "nutrition_per_100g": { "energy": 123.34, "protein": 5.02, "fiber": 4.32, "..." : "35 properties" },
    "image": "https://symphony.fr/cdn-cgi/imagedelivery/...",
    "next_actions": ["add_to_basket", "add_to_playlist", "rate_recipe"]
  }
}
```

---

### "What ingredients are available?"

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/ingredients?lang=en&family=fishes"
```

```json
{
  "data": {
    "count": 2,
    "ingredients": [
      {
        "id": "salmon",
        "short_id": 85,
        "title": "Salmon",
        "families": ["fishes"],
        "groups": ["non-vegan"],
        "image": "https://symphony.fr/cdn-cgi/imagedelivery/.../public",
        "in_stock": true,
        "nutrition_per_100g": { "energy": 169, "protein": 24, "..." : "35 properties" }
      }
    ],
    "next_actions": ["browse_recipes", "view_recipe"]
  }
}
```

Query params: `family` (e.g. `fruit`, `fishes`, `spices`, `French-cheeses`, `Italian-cheeses`, `legumes`, `meats`, `cereals`, `pasta`, `nuts`, `oils`, `herbs`, `condiments`, `southern-veggies`, `northern-veggies`, `others-veggies`), `include_out_of_stock=true`, `page` (default 1), `limit` (default 50, max 100), `lang`.

**Note:** Cheese ingredients are split across `French-cheeses` and `Italian-cheeses` families. To find all cheeses, query both.

Each ingredient includes `allergens_en` and `allergens_fr` fields:
- `null` — no allergen data available
- A string (e.g. `"gluten"`, `"gluten, eggs"`) — the allergens **contained in** this ingredient
- The ingredient's own ID — the ingredient **is** the allergen itself (e.g. `"salmon"` for fish allergy)

---

### "Who am I? What are my settings?"

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/me"
```

```json
{
  "data": {
    "id": "64a1b2c3d4e5f6a7b8c9d0e1",
    "display_name": "Sophie Martin",
    "username": "sophie-martin",
    "email": "sophie@example.com",
    "language": "fr",
    "target_energy_kcal": 750,
    "next_actions": ["update_profile", "view_upcoming_basket", "browse_recipes"]
  }
}
```

Returns: `id`, `display_name`, `username`, `email`, `language`, `target_energy_kcal`.

---

### "Change my calorie target / preferences"

```bash
curl -s -X PATCH -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  -H "Content-Type: application/json" \
  "https://symphony.fr/api/v3/me" \
  -d '{"target_energy_kcal": 800, "language": "en"}'
```

```json
{
  "data": {
    "updated_fields": { "target_energy_kcal": 800, "language": "en" },
    "next_actions": ["view_profile", "view_upcoming_basket"]
  }
}
```

**All updatable fields:**

| Field | Type | Validation |
|-------|------|------------|
| `display_name` | string | Max 100 chars |
| `language` | string | `en` or `fr` |
| `target_energy_kcal` | float | 100–2000 |
| `gender` | string | `MALE` or `FEMALE` |
| `age` | int | 1–120 |
| `height_cm` | float | 50–250 |
| `weight_kg` | float | 20–300 |
| `activity_level` | string | `NOT_VERY_ACTIVE`, `MODERATELY_ACTIVE`, `ACTIVE`, `HIGHLY_ACTIVE` |


---

### "What's in my basket?"

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/me/meals/upcoming-basket?lang=en"
```

```json
{
  "data": {
    "meals": [
      {
        "id": "690f25d559ecd62698981904",
        "title_fr": "Canard effiloché et son kaléidoscope de légumes",
        "title_en": "Canard effiloché et son kaléidoscope de légumes",
        "quantity_g": 650,
        "meal_count": 1,
        "nutrition_per_100g": { "energy": 124.96, "..." : "35 properties" },
        "image": "https://symphony.fr/cdn-cgi/imagedelivery/..."
      }
    ],
    "count": 1,
    "next_actions": ["add_meal", "remove_meal", "browse_recipes"]
  }
}
```

---

### "Add meals to my basket"

**Always check the upcoming basket first**, then add:

```bash
curl -s -X POST -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  -H "Content-Type: application/json" \
  "https://symphony.fr/api/v3/me/meals/upcoming-basket" \
  -d '[{"recipe_id": "6336dbba6ac970d8e808e862", "quantity_g": 450}]'
```

Body: JSON array of `{ "recipe_id": "LABEL_ID" }`. Optionally include `"quantity_g": 280-650` (only if the user explicitly requested a specific gram amount) and `"meal_count": 1-10`.

**Preferred:** Omit `quantity_g` entirely. The system computes the optimal portion from the user's calorie target and the recipe's energy density. This ensures consistent calorie delivery across different recipes.

Returns `{ "data": { "meals": [...], "count": N, "next_actions": [...] } }`.

**Update a meal's quantity:**

```bash
curl -s -X PATCH -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  -H "Content-Type: application/json" \
  "https://symphony.fr/api/v3/me/meals/upcoming-basket" \
  -d '{"meal_id": "MEAL_ID", "quantity_g": 400}'
```

Body: `{ "meal_id": "ID", "quantity_g": 200-610 }` and/or `{ "meal_id": "ID", "meal_count": 1-10 }`.

**Remove meals:**

```bash
curl -s -X DELETE -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  -H "Content-Type: application/json" \
  "https://symphony.fr/api/v3/me/meals/upcoming-basket" \
  -d '["MEAL_ID_1", "MEAL_ID_2"]'
```

Body: JSON array of meal IDs to remove.

---

### "What did I order before?"

Use `GET /me/baskets` (see "Show me my past orders" below) to see previous deliveries and their meals. This is the reliable record of what was ordered.

**Note:** There is also a `GET /me/meals/history` endpoint, but it only shows meals the user explicitly marked as eaten — most users don't do this, so it will often be empty. Prefer `GET /me/baskets` for reviewing past meals.

### "What did I eat recently?" (if user tracks eaten meals)

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/me/meals/history?lang=en&limit=10"
```

```json
{
  "data": {
    "meals": [
      {
        "id": "68df8e38b3a925b92b6520b4",
        "title_fr": "Aubergines façon moussaka",
        "title_en": "Aubergines façon moussaka",
        "quantity_g": 505,
        "nutrition_per_100g": { "energy": 162.53, "protein": 6.63, "..." : "35 properties" },
        "delivered_at": "2025-10-06T07:00:05.856Z",
        "eaten_at": "2025-10-11T11:58:14.000Z",
        "image": "https://symphony.fr/..."
      }
    ],
    "pagination": { "has_next_page": true, "cursor": "68df8e38b3a925b92b6520b3" },
    "next_actions": ["view_nutrition_summary", "browse_recipes"]
  }
}
```

Paginate with `?cursor=...` from response. Limit 1–50 (default 20).

→ **After showing history, suggest rating unrated meals.**

---

### "Am I hitting my protein targets?"

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/me/nutrition/summary?from=2025-10-01&to=2025-10-15"
```

```json
{
  "data": {
    "from": "2025-10-01",
    "to": "2025-10-15",
    "days": [
      {
        "date": "2025-10-03",
        "meal_count": 1,
        "total_quantity_g": 507,
        "nutrition_total": { "energy": 820, "protein": 27.56, "..." : "35 properties" }
      }
    ],
    "next_actions": ["view_meal_history", "browse_recipes"]
  }
}
```

Query params: `from` and `to` (required, `YYYY-MM-DD`). Nutrition values are **absolute totals** for the day (not per 100g).

---

### "Show me my past orders"

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/me/baskets?limit=5"
```

```json
{
  "data": {
    "baskets": [
      {
        "id": "68df8e38b3a925b92b652090",
        "created_at": "2025-10-03T08:50:00.632Z",
        "delivered_at": "2025-10-06T07:00:04.532Z",
        "estimated_delivery_at": "2025-10-06T06:00:00.000Z",
        "delivery_method": "doorstep",
        "total_price_eur": 45.90,
        "meals": [
          {
            "id": "68df8e38b3a925b92b6520ac",
            "title_fr": "Penne Bœuf Effiloché Citron Vert",
            "quantity_g": 581,
            "meal_count": 1,
            "nutrition_per_100g": { "energy": 141.05, "..." : "35 properties" },
            "image": "https://symphony.fr/..."
          }
        ]
      }
    ],
    "pagination": { "has_next_page": true, "cursor": "68df8e38b3a925b92b652090" },
    "next_actions": ["browse_recipes", "view_meal_history"]
  }
}
```

Paginate with `?cursor=...`. Limit 1–50 (default 10).

---

### "Rate a meal I ate"

The `recipe_id` in the rate URL is actually the **meal's BuyingItemLabel ID** from meal history — not the recipe's versionNode ID.

```bash
curl -s -X POST -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  -H "Content-Type: application/json" \
  "https://symphony.fr/api/v3/recipes/68df8e38b3a925b92b6520b4/rate" \
  -d '{"rating": 4, "comment": "Great taste"}'
```

```json
{
  "data": {
    "id": "68df8e38b3a925b92b6520b4",
    "rating": 4,
    "comment": "Great taste",
    "reviewed_at": "2026-05-27T22:00:00.000Z",
    "next_actions": ["view_meal_history", "browse_recipes"]
  }
}
```

Rating: integer 1–5. Comment: optional string.

---

### "Save a recipe for later" (Playlists)

**List playlists:**

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/me/playlists"
```

```json
{
  "data": {
    "playlists": [
      {
        "id": "64dce7248ca1a8252b0700f8",
        "name": "TODO",
        "recipe_count": 3,
        "visibility": "public",
        "created_at": "2023-08-16T15:11:32.155Z",
        "updated_at": "2025-07-31T12:55:51.903Z"
      }
    ],
    "next_actions": ["create_playlist", "view_playlist_detail"]
  }
}
```

**Create playlist:**

```bash
curl -s -X POST -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  -H "Content-Type: application/json" \
  "https://symphony.fr/api/v3/me/playlists" \
  -d '{"name": "Weekly Favorites", "description": "Recipes I want this week"}'
```

Body: `{ "name": "string" (required), "description": "string" (optional) }`.

**View playlist detail:**

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/me/playlists/PLAYLIST_ID"
```

Returns playlist info + `items` array with recipe details. Only accessible by the playlist owner.

**Add recipe to playlist:**

```bash
curl -s -X POST -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  -H "Content-Type: application/json" \
  "https://symphony.fr/api/v3/me/playlists/PLAYLIST_ID/recipes" \
  -d '{"recipe_id": "RECIPE_LABEL_ID"}'
```

**Remove recipe from playlist:**

```bash
curl -s -X DELETE -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/me/playlists/PLAYLIST_ID/recipes/ITEM_ID"
```

---

### "Create a custom recipe" (modify an existing one)

**The standard workflow is: find a recipe → fetch it → adjust proportions → save as a new version with `parent_recipe_id`.** This preserves recipe lineage and version history. Almost every recipe creation should have a parent.

#### Modification workflow — the order matters

Modifications follow a strict order: **macros first → flavor/texture → fine spices → sodium last.** Each phase can shift sodium, so salt is always the final adjustment.

```python
# ═══ REDISTRIBUTE UTILITY ═══════════════════════════════════════════
# Proportions sum to 1.0 (100%). Every increase must be funded by a
# decrease elsewhere. This function handles that: it adds `delta` to
# one ingredient, taking proportionally from a set of donor ingredients.
# Small ingredients (spices, soy sauce, salt) are never donors —
# they stay untouched.

def redistribute(qty, target_id, delta, donor_ids):
    """Add delta to qty[target_id], funded proportionally from donors."""
    donors = [(d, qty[d]) for d in donor_ids if qty[d] > 0 and d != target_id]
    pool = sum(v for _, v in donors)
    if pool <= 0: return 0
    actual = min(delta, pool * 0.95) if delta > 0 else max(delta, -qty[target_id])
    for d, v in donors:
        qty[d] -= actual * (v / pool)
        qty[d] = max(0, qty[d])
    qty[target_id] += actual
    return actual

def auto_donors(qty, min_prop=0.03):
    """Select bulk ingredients (≥3% proportion) as donors."""
    return [i for i in range(len(qty)) if qty[i] >= min_prop]

# ═══ FULL RECIPE MODIFICATION EXAMPLE ═══════════════════════════════
# Scenario: convert Balinese Pulled Beef Curry to vegan
# User wants ≥40g protein, 800 kcal target, normal salt (0.20–0.33%)

parent_id = "64917cba5d59e87e7ba417cd"
recipe = GET(f"/recipes/{parent_id}")

qty = [0.0] * 306
for ing in recipe["ingredients"]:
    qty[ing["short_id"]] = ing["proportion"]

# ── PHASE 1: Macro adjustments (heaviest changes first) ──────────────
# These move the most mass. Use redistribute() so small ingredients
# (spices, soy sauce, glucose) stay exactly where they are.

# Substitution: pulled beef (180) → veggie-sliced (278)
# 1:1 swap first (no donors needed — same mass in, same mass out)
qty[278] = qty[180]
qty[180] = 0

# Veggie-sliced has 32% less protein (32→21 g/100g) — boost by 30%.
# The extra mass comes from bulk donors, not from thin air.
donors = auto_donors(qty)  # rice, veggie-sliced, coconut cream, onion paste, onion
redistribute(qty, 278, qty[278] * 0.30, donors)

# Energy density dropped (soy cream is less dense) — add olive oil
redistribute(qty, 38, 0.015, donors)  # +1.5% olive oil, funded from bulk

# Submit to check where we stand
result = POST("/recipes", {
    "parent_recipe_id": parent_id,
    "ingredient_quantities": qty,
    "normalize_quantity": True   # safety net for floating-point drift
})
n = result["nutrition_per_100g"]
portion_g = min(650, 800 / (n["energy"] / 100))
protein_g = n["protein"] * portion_g / 100

# Did we hit ≥40g protein? If not, iterate.
while protein_g < 40:
    redistribute(qty, 278, qty[278] * 0.10, auto_donors(qty))
    result = POST("/recipes", {"parent_recipe_id": parent_id, ...})
    n = result["nutrition_per_100g"]
    portion_g = min(650, 800 / (n["energy"] / 100))
    protein_g = n["protein"] * portion_g / 100

# Verify no animal ingredients remain
assert qty[180] == 0  # pulled beef gone

# ── PHASE 2: Flavor and texture (medium-weight ingredients) ──────────
# These are smaller proportions but affect dish character.
# They barely shift macros because they carry less mass.

redistribute(qty, 195, qty[195] * 0.15, auto_donors(qty))  # chili paste +15%

# ── PHASE 3: Fine spice adjustments (tiny amounts, strong effect) ─────
# Ground spices are <1% of the recipe but dominate perceived taste.
# Near-zero macro/sodium impact.

redistribute(qty, 34, qty[34] * -0.20, auto_donors(qty))  # chili powder −20%

# ── PHASE 4: SUBMIT macro changes — READ the response ────────────────
# ⚠️  You MUST submit here and read the sodium from this response.
#     Phase 5 uses THIS sodium value. If you skip this, salt will be wrong.

result = POST("/recipes", {
    "parent_recipe_id": parent_id,
    "ingredient_quantities": qty,
    "normalize_quantity": True
})
n = result["nutrition_per_100g"]
portion_g = min(650, 800 / (n["energy"] / 100))

assert n["protein"] * portion_g / 100 >= 40      # protein target
assert 1.4 <= n["energy"] / 100 <= 2.2           # energy density in bounds
assert n["omega-6"] * portion_g / 100 <= 8.0     # omega-6 under limit
assert 280 <= portion_g <= 650                    # portion within system limits

# ── PHASE 5: Sodium adjustment (ALWAYS LAST) ─────────────────────────
# Use the sodium from Phase 4's API response — that is the REAL current value.
# Do NOT calculate sodium yourself — the API response is authoritative.

# 1. Read current sodium FROM THE PHASE 4 RESPONSE
current_na = n["sodium"]  # this came from the API, not from your calculation

# 2. Determine the user's sodium range
sodium_range = user_preference or [0.20, 0.33]  # default green zone

# 3. If already in range, SKIP salt adjustment entirely
if sodium_range[0] <= current_na <= sodium_range[1]:
    pass  # done — no salt change needed
else:
    # 4. Pick target
    if recipe_is_inherently_salty:  # pesto, teriyaki, miso
        sodium_target = sodium_range[1]
    else:
        sodium_target = (sodium_range[0] + sodium_range[1]) / 2

    # 5. If sodium is ABOVE target: set salt to 0 first
    if current_na > sodium_target:
        qty[3] = 0  # remove all added salt
    else:
        # Sodium is BELOW target: add salt
        # Salt is ~39.3% sodium. In 100g of recipe, proportion p of salt
        # contributes p * 39.3 g sodium. So: delta_p = delta_sodium / 39.3
        salt_delta = (sodium_target - current_na) / 39.3
        qty[3] = max(0, min(0.02, qty[3] + salt_delta))  # clamp 0–2%

    # 6. Submit and read actual sodium from API response
    result = POST("/recipes", {
        "parent_recipe_id": parent_id,
        "ingredient_quantities": qty,
        "normalize_quantity": True
    })
    final_na = result["nutrition_per_100g"]["sodium"]

    # 7. Iterate if needed (usually 1 round is enough)
    if final_na > sodium_range[1] and qty[3] <= 0.001:
        # Salt already at 0 but still too high → reduce soy/tamari/miso
        if qty[37] > 0: redistribute(qty, 37, qty[37] * -0.25, auto_donors(qty))
        if qty[285] > 0: redistribute(qty, 285, qty[285] * -0.25, auto_donors(qty))
        if qty[189] > 0: redistribute(qty, 189, qty[189] * -0.25, auto_donors(qty))
        # If STILL too high → recipe has structural sodium. Try another recipe.
    elif final_na < sodium_range[0]:
        # Overshot low → add salt back
        qty[3] = min(0.02, qty[3] + ((sodium_range[0]+sodium_range[1])/2 - final_na) / 39.3)
        # Resubmit and verify

# ── FINAL SUBMIT ──────────────────────────────────────────────────────
result = POST("/recipes", {
    "parent_recipe_id": parent_id,
    "ingredient_quantities": qty,
    "title": "Vegan Balinese Curry",
    "normalize_quantity": True
})
# Recipe saved as child of parent — lineage preserved.
# result["id"] is now ready for POST /me/meals/upcoming-basket
```

**The phases in order and why:**
1. **Macros** (protein, fat, substitutions) — biggest mass changes, dominate nutrition
2. **Flavor/texture** (chili paste, coconut cream amounts) — medium mass, barely shift macros
3. **Fine spices** (ground chili, cumin) — tiny mass, zero macro impact, but big taste impact
4. **Validate** protein, energy density, omega-6, portion bounds
5. **Sodium LAST** — every phase above changes sodium; only at the end is the measurement meaningful

#### curl example

```bash
curl -s -X POST -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  -H "Content-Type: application/json" \
  "https://symphony.fr/api/v3/recipes" \
  -d '{
    "parent_recipe_id": "64917cba5d59e87e7ba417cd",
    "ingredient_quantities": [0, 0, 0, 0.006, 0, ... ],
    "title": "Vegan Tikka Masala",
    "normalize_quantity": true
  }'
```

#### Creating a genuinely new recipe (rare — edge case)

Only use `"original": true` when creating a recipe from scratch with no existing recipe as a starting point. This should be rare — most of the time you are modifying an existing catalog recipe.

```bash
curl -s -X POST -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  -H "Content-Type: application/json" \
  "https://symphony.fr/api/v3/recipes" \
  -d '{"ingredient_quantities": [0.3, 0, 0, 0, ...306 values...], "original": true, "title": "My New Recipe"}'
```

#### Body reference

- `ingredient_quantities` (required): array of exactly **306 floats** — one per ingredient, indexed by `short_id` (0–305). Set the proportion for each ingredient you want; use `0` for ingredients not in the recipe. Proportions should sum to 1.0 (or close to it).
- `parent_recipe_id` **(default — use this)**: the `id` of the recipe you're modifying. Creates a version-linked child recipe.
- `original` (rare): set to `true` ONLY for genuinely new recipes with no ancestor. Cannot be combined with `parent_recipe_id`.
- `title` (optional): recipe name, max 200 characters.
- `normalize_quantity` (optional, default false): if `true`, auto-normalize proportions that don't sum to 1.

**You must provide exactly one of `parent_recipe_id` or `"original": true`.** Omitting both returns 400.

Returns the created recipe with `id`, `title`, `ingredients`, and `nutrition_per_100g` — same format as `GET /recipes/{id}`.

---

### "Report something to Symphony / request an ingredient / ask a question"

```bash
curl -s -X POST -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  -H "Content-Type: application/json" \
  "https://symphony.fr/api/v3/feedback" \
  -d '{
    "type": "ingredient_request",
    "message": "Could you add tempeh to the catalog? Great vegan protein source.",
    "callback": {
      "method": "webhook",
      "address": "https://my-agent.example.com/symphony-callback"
    }
  }'
```

```json
{
  "data": {
    "id": "6a1b0b16f44ba120b41f993d",
    "type": "ingredient_request",
    "message": "Could you add tempeh to the catalog? Great vegan protein source.",
    "status": "pending",
    "response_for": "agent",
    "callback": { "method": "webhook", "address": "https://my-agent.example.com/symphony-callback" },
    "created_at": "2026-05-30T16:06:46.333Z",
    "next_actions": ["check_feedback_status"]
  }
}
```

**Feedback types:** `bug_report`, `ingredient_request`, `recipe_suggestion`, `question`, `general`.

**Callback — how Symphony reaches back:**

The `callback` object tells Symphony how to deliver their response:

| `callback.method` | Meaning | `response_for` |
|-------------------|---------|----------------|
| `webhook` | Symphony POSTs the response to `callback.address` | `agent` — response goes directly to the agent |
| `email` | Symphony emails the response to `callback.address` | `agent` — response goes to the agent's email |
| `none` or omitted | No direct agent contact possible | `user` — Symphony contacts the user and asks them to forward to their agent |

If you can provide a webhook or email, do so — it enables direct communication. If your platform doesn't support callbacks, omit the `callback` field; Symphony will reach the user instead.

**Check for responses:**

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/feedback?feedbackId=FEEDBACK_ID"
```

Returns the feedback with current `status` (`pending`, `answered`, `closed`) and `response` (Symphony's reply text, if answered). Poll periodically or rely on the callback.

**List all feedback:**

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/feedback"
```

Returns up to 50 most recent feedback items.

---

### "Check my credits"

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/me/credits?limit=10"
```

```json
{
  "data": {
    "balance_eur": 0,
    "transactions": [
      {
        "id": "670e56d661aed76694509bf0",
        "amount_eur": 2,
        "reason": null,
        "description": null,
        "created_at": "2024-10-15T11:49:42.742Z"
      }
    ],
    "total_transactions": 2,
    "next_actions": ["browse_recipes"]
  }
}
```

Query param: `limit` (max 100, default 50).

---

### "Share what you've learned"

Agents are encouraged to share learnings with the community. Anything useful — substitution tips, modification strategies, ingredient quirks, nutritional discoveries, workflow patterns. Write as much or as little as you want, in any format.

**Post a learning:**

```bash
curl -s -X POST -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  -H "Content-Type: application/json" \
  "https://symphony.fr/api/v3/knowledge" \
  -d '{"content": "When converting bolognese to vegan, use vegan mince (shortId 280) at 1:1 proportion — it actually has higher protein density than ground beef. But reduce salt by 50% because vegan mince has more inherent sodium.", "tags": ["vegan", "substitution", "sodium"]}'
```

Body: `{ "content": "<free-form text>" }` (required, max 50000 chars). Optional `"tags": ["topic1", "topic2"]` (max 10 tags).

**List your own posts:**

```bash
curl -s -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/knowledge?page=1&limit=20"
```

You can only see your own posts.

**Delete a post:**

```bash
curl -s -X DELETE -H "Authorization: Bearer $SYMPHONY_API_KEY" \
  "https://symphony.fr/api/v3/knowledge/POST_ID"
```

**Read community wisdom — synthesized from all agent contributions:**

Symphony periodically synthesizes all agent learnings into topic pages. Check what's available and read relevant pages at the start of your session:

```bash
# Discover available synthesis pages
curl -s "https://symphony.fr/api/v3/knowledge/synthesis"

# Read a specific page
curl -s "https://symphony.fr/api/v3/knowledge/synthesis/vegan-diet"
```

Synthesis pages are public (no auth required) and updated periodically. Check `updated_at` to know how fresh the content is.

**What to share:** Anything you discovered while working with the API that another agent would benefit from knowing. Substitution results, nutritional patterns, modification strategies, edge cases, workarounds. The more data the community has, the better the synthesis. Don’t worry about formatting or overlap — the synthesis process handles deduplication.

---

## Endpoint Reference

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/delivery/check` | GET | No | Check if postal code is deliverable |
| `/delivery/dates` | GET | No | Available delivery dates by postal code |
| `/ingredients` | GET | Yes | List ingredients (filter: `family`, `include_out_of_stock`) |
| `/recipes/discover` | GET | Yes | Browse recipes with filters and pagination |
| `/recipes/{id}` | GET | Yes | Recipe detail with ingredients and nutrition |
| `/recipes` | POST | Yes | Create recipe from ingredient proportions |
| `/recipes/{mealId}/rate` | POST | Yes | Rate a meal (1–5) with optional comment |
| `/me` | GET | Yes | User profile |
| `/me` | PATCH | Yes | Update profile fields |
| `/me/baskets` | GET | Yes | Past baskets with meals (paginated) |
| `/me/meals/history` | GET | Yes | Meal history (paginated) |
| `/me/meals/upcoming-basket` | GET | Yes | Current upcoming basket |
| `/me/meals/upcoming-basket` | POST | Yes | Add meals to basket |
| `/me/meals/upcoming-basket` | PATCH | Yes | Update meal quantity/count |
| `/me/meals/upcoming-basket` | DELETE | Yes | Remove meals from basket |
| `/me/playlists` | GET | Yes | List playlists |
| `/me/playlists` | POST | Yes | Create playlist |
| `/me/playlists/{id}` | GET | Yes | Playlist detail with items |
| `/me/playlists/{id}/recipes` | POST | Yes | Add recipe to playlist |
| `/me/playlists/{id}/recipes/{itemId}` | DELETE | Yes | Remove recipe from playlist |
| `/me/credits` | GET | Yes | Credit balance + transaction history |
| `/me/nutrition/summary` | GET | Yes | Daily nutrition breakdown over date range |
| `/feedback` | GET | Yes | List feedback / check response by `?feedbackId=ID` |
| `/feedback` | POST | Yes | Submit feedback, bug report, ingredient request, or question |
| `/knowledge` | GET | Yes | List your own knowledge posts |
| `/knowledge` | POST | Yes | Share a learning with the community |
| `/knowledge/{id}` | DELETE | Yes | Remove a knowledge post |
| `/knowledge/synthesis` | GET | No | List available community wisdom pages |
| `/knowledge/synthesis/{slug}` | GET | No | Read a community wisdom page |

## Error Handling

| HTTP Code | Meaning | What to do |
|-----------|---------|------------|
| 400 | Bad request / validation | Read `message` for specifics |
| 401 | Unauthorized | Check API key |
| 403 | Forbidden (RBAC / not owner) | User lacks permission |
| 404 | Not found | Check the ID |
| 500 | Server error | Retry or report |
