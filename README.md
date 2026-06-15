# FitFindr — Starter Kit

This starter kit contains everything you need to begin Project 2.

## What's Included

```
ai201-project2-fitfindr-starter/
├── data/
│   ├── listings.json          # 40 mock secondhand listings
│   └── wardrobe_schema.json   # Wardrobe format + example wardrobe
├── utils/
│   └── data_loader.py         # Helper functions for loading the data
├── planning.md                # Your planning template — fill this out first
└── requirements.txt           # Python dependencies
```

## Setup

```bash
pip install -r requirements.txt
```

Set your Groq API key in a `.env` file (get a free key at [console.groq.com](https://console.groq.com)):
```
GROQ_API_KEY=your_key_here
```

## The Mock Listings Dataset

`data/listings.json` contains 40 mock secondhand listings across categories (tops, bottoms, outerwear, shoes, accessories) and styles (vintage, y2k, grunge, cottagecore, streetwear, and more).

Each listing has: `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, and `platform`.

Load it with:
```python
from utils.data_loader import load_listings
listings = load_listings()
```

## The Wardrobe Schema

`data/wardrobe_schema.json` defines the format your agent uses to represent a user's existing wardrobe. It includes:

- `schema`: field definitions for a wardrobe item
- `example_wardrobe`: a sample wardrobe with 10 items you can use for testing
- `empty_wardrobe`: a starting template for a new user

Load an example wardrobe with:
```python
from utils.data_loader import get_example_wardrobe
wardrobe = get_example_wardrobe()
```

## Where to Start

1. **Read `planning.md` and fill it out before writing any code.**
2. Verify the data loads correctly by running `python utils/data_loader.py`.
3. Build and test each tool individually before connecting them through your planning loop.

Your implementation files go in this same directory. There's no required file structure for your agent code — organize it however makes sense for your design.

---

## Tool Inventory

### `search_listings(description, size, max_price)`
Scans the mock listings dataset and returns items matching the user's keywords, with optional size and price filters.

| Parameter | Type | Description |
|-----------|------|-------------|
| `description` | `str` | Keywords describing the item (e.g. "vintage graphic tee") |
| `size` | `str \| None` | Size filter, case-insensitive. Pass `None` to skip. |
| `max_price` | `float \| None` | Maximum price inclusive. Pass `None` to skip. |

**Returns:** `list[dict]` — matching listings sorted by keyword overlap score, highest first. Each dict has `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, `platform`. Returns `[]` on no match, never raises.

---

### `suggest_outfit(new_item, wardrobe)`
Calls the Groq LLM to suggest 1–2 complete outfit combinations using the thrifted item and the user's wardrobe.

| Parameter | Type | Description |
|-----------|------|-------------|
| `new_item` | `dict` | A listing dict for the item the user is considering |
| `wardrobe` | `dict` | Wardrobe dict with an `'items'` key containing wardrobe item dicts |

**Returns:** `str` — a non-empty outfit suggestion with specific styling details. If the wardrobe is empty, returns general styling advice using universal staples instead of crashing.

---

### `create_fit_card(outfit, new_item)`
Turns an outfit suggestion into a short, authentic Instagram/TikTok OOTD caption using the Groq LLM at high temperature so outputs vary.

| Parameter | Type | Description |
|-----------|------|-------------|
| `outfit` | `str` | The outfit suggestion string from `suggest_outfit` |
| `new_item` | `dict` | The listing dict for the thrifted item |

**Returns:** `str` — a 2–4 sentence casual caption mentioning the item name, price, and platform once each. If `outfit` is empty or whitespace, returns a safe template fallback without crashing.

---


## How the Planning Loop Works

The loop in `run_agent()` is a conditional pipeline — it checks each tool's output before deciding whether to continue or stop:

1. Parse the query with regex to extract `max_price` (looks for "under $X")
2. Call `search_listings()` with the parsed parameters
3. **If results are empty → set `session["error"]` and return early.** `suggest_outfit` and `create_fit_card` are never called.
4. Take the top result and store it as `session["selected_item"]`
5. Call `suggest_outfit()` with the selected item and wardrobe
6. Call `create_fit_card()` with the outfit suggestion and selected item
7. Return the completed session

The agent does not call all three tools unconditionally. If step 3 finds nothing, the interaction ends there with a helpful message.

---

## State Management

All state lives in a single `session` dict initialized at the start of each `run_agent()` call:

```python
session = {
    "query": str,             # original user input
    "parsed": dict,           # extracted description / max_price
    "search_results": list,   # full list from search_listings
    "selected_item": dict,    # results[0], passed into suggest_outfit
    "wardrobe": dict,         # user's wardrobe, passed into suggest_outfit
    "outfit_suggestion": str, # output of suggest_outfit, passed into create_fit_card
    "fit_card": str,          # final caption from create_fit_card
    "error": str | None,      # set on early exit, None on success
}
```

The `selected_item` that flows into `suggest_outfit` is the exact same dict that flows into `create_fit_card` — nothing is re-fetched or re-entered between steps.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| `search_listings` | No listings match the query | Returns `[]`. Planning loop detects this, sets `session["error"]` to "No listings found. Try different keywords or a higher price." and exits before calling any LLM tools. |
| `suggest_outfit` | Wardrobe `items` list is empty | Detects the empty list and switches to a fallback LLM prompt asking for general styling advice using universal wardrobe staples. Returns a non-empty string — never crashes. |
| `create_fit_card` | `outfit` input is empty or whitespace | Guards with `if not outfit.strip()` before calling the LLM. Returns a template string like `"just picked up this {title} on {platform} for ${price} ✨ breakdown coming soon!"` |

**Concrete example:** Running the query `"designer ballgown size XXS under $5"` triggers the `search_listings` failure mode and returns: No listings found. Try different keywords or a higher price."

The agent stops there — `suggest_outfit` and `create_fit_card` are never called.

---

## Spec Reflection

**One way the spec helped:** Writing out the conditional logic in `planning.md` before touching `agent.py` made the branching structure clear upfront. Knowing that an empty result list meant an immediate return — not a fallback search — made the implementation straightforward.

**One way implementation diverged:** The planning.md spec included a `current_step` field in the session dict to track pipeline progress. In practice it wasn't needed — the Python call stack already handles sequencing, and the `error` field covers the only case where step tracking mattered. Removed it to keep the session dict clean.

---

## AI Usage

**Instance 1 — `search_listings` scoring logic**
I gave Claude the Tool 1 spec from `planning.md` (inputs, return value, scoring approach, failure mode) and the function stub from `tools.py`. It generated a working implementation using `load_listings()` with price/size filtering and keyword overlap scoring. I changed the size filter from exact string equality to a case-insensitive substring match (`size_lower in item_size.lower()`) to handle formats like `"S/M"` that appear in the real dataset.

**Instance 2 — `run_agent()` planning loop**
I gave Claude the Architecture diagram and Planning Loop section from `planning.md`. The draft it produced called `get_example_wardrobe()` directly inside `run_agent`, hardcoding the wardrobe. I changed it to accept `wardrobe` as a parameter so `app.py` could pass either the example or empty wardrobe based on the user's selection in the UI.

