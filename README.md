# FitFindr

FitFindr is a multi-tool AI agent that helps users find secondhand clothing and figure out how to style it. You describe what you're looking for in natural language, and the agent searches a dataset of mock listings, suggests an outfit based on your existing wardrobe, and generates a shareable caption for the look.

---

## Setup

```bash
pip install -r requirements.txt
```

Set your Groq API key in a `.env` file (get a free key at [console.groq.com](https://console.groq.com)):
```
GROQ_API_KEY=your_key_here
```

Run the Gradio UI:
```bash
python app.py
```

Or run the agent directly:
```bash
python agent.py
```

---

## Tools

FitFindr uses three tools, each implemented as a standalone function in `tools.py`.

### `search_listings(description, size, max_price)`

Searches the mock listings dataset (`data/listings.json`) for items matching a natural language description, with optional size and price filters.

- **description** (str): Keywords describing what the user wants, e.g. `"vintage graphic tee"`
- **size** (str | None): Size string to filter by, e.g. `"M"`. `None` skips size filtering.
- **max_price** (float | None): Maximum price in USD (inclusive). `None` skips price filtering.

Returns a list of matching listing dicts sorted by relevance score (highest first), or an empty list if nothing matches. Each listing has: `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, `platform`.

Scoring works by counting keyword overlap between the user's description and each listing's title, description, category, and style tags. Listings with a score of 0 are dropped.

---

### `suggest_outfit(new_item, wardrobe)`

Given a candidate thrifted item and the user's wardrobe, uses an LLM (Groq) to suggest 1–2 complete outfits.

- **new_item** (dict): A listing dict — the item the user is considering buying.
- **wardrobe** (dict): A wardrobe dict with an `"items"` key containing a list of wardrobe item dicts. May be empty.

Returns a non-empty string with outfit suggestions. If the wardrobe has items, suggestions name specific pieces the user already owns. If the wardrobe is empty, returns general styling advice for the item instead.

---

### `create_fit_card(outfit, new_item)`

Uses an LLM (Groq) to generate a 2–4 sentence Instagram/TikTok-style caption for the outfit.

- **outfit** (str): The outfit suggestion string from `suggest_outfit()`.
- **new_item** (dict): The listing dict for the thrifted item.

Returns a casual caption that mentions the item name, price, and platform once each. Uses a higher LLM temperature (0.9) so the output sounds different for different inputs. Never raises an exception — returns an error message string if the outfit input is missing.

---

## Planning Loop

The agent runs a fixed three-step pipeline in `agent.py`. The planning loop does not use an LLM to decide which tool to call — it always calls all three tools in order, but checks state between steps to decide whether to continue.

**Step 1 — Parse the query.** The agent uses regex to extract `max_price` (e.g. `$30`) and `size` (e.g. `M`) from the user's message. The full message is used as the `description` for keyword scoring.

**Step 2 — Call `search_listings`.** The agent calls `search_listings` with the parsed parameters and stores the results in `session["search_results"]`. It then checks: if the list is empty, it sets `session["error"]` to a helpful message and returns immediately. It does not proceed to `suggest_outfit` with empty input — that would produce a nonsensical outfit suggestion for no item.

**Step 3 — Pick the top result.** The agent selects `session["search_results"][0]` (the highest-scoring listing) as `session["selected_item"]`.

**Step 4 — Call `suggest_outfit`.** Passes `selected_item` and `session["wardrobe"]` to `suggest_outfit`. Stores the result in `session["outfit_suggestion"]`.

**Step 5 — Call `create_fit_card`.** Passes `outfit_suggestion` and `selected_item` to `create_fit_card`. Stores the result in `session["fit_card"]`.

**Step 6 — Return.** Returns the completed session dict. The caller checks `session["error"]` to determine whether the run succeeded.

---

## State Management

All state lives in a single session dict initialized in `_new_session()` at the start of each `run_agent()` call. No global variables or external storage are used. Each tool reads the fields it needs and writes its output back to the dict:

```
session = {
    "query":              # original user message
    "parsed":             # extracted description, size, max_price
    "search_results":     # list of matching listings (set by search_listings)
    "selected_item":      # top listing dict (set after search step)
    "wardrobe":           # loaded once before run_agent is called
    "outfit_suggestion":  # string from suggest_outfit
    "fit_card":           # string from create_fit_card
    "error":              # set if the run ended early; None on success
}
```

---

## Error Handling

| Tool | Failure mode | What the agent does |
|------|-------------|---------------------|
| `search_listings` | No listings match the query | Sets `session["error"]` to a message asking the user to broaden their search (different keywords, no size filter, higher price). Returns immediately — does not call `suggest_outfit` or `create_fit_card`. |
| `suggest_outfit` | Wardrobe is empty | Detects `wardrobe["items"] == []` before calling the LLM and switches to a general styling prompt. Returns general advice instead of wardrobe-specific outfits. Pipeline continues normally. If the LLM call itself fails (API error), returns a fallback string: `"Outfit suggestions are unavailable right now, but this item is worth checking out!"` |
| `create_fit_card` | Outfit string is empty or whitespace | Returns `"Could not generate a fit card: outfit description was missing."` without calling the LLM. If the LLM call fails, returns a template fallback: `"Found on [platform] for $[price] — this one speaks for itself."` |

**Concrete example from testing:** Running `search_listings("designer ballgown size XXS under $5", ...)` returns an empty list because no listing in the dataset costs under $5. The agent catches this immediately and returns: `"No listings matched your search. Try different keywords, remove the size filter, or raise your price ceiling."` — the outfit and fit card panels stay blank.

---

## Spec Reflection

**One way the spec helped:** Writing out the complete interaction in `planning.md` before touching any code made it clear that `search_listings` needed to return early and block the rest of the pipeline — not just return an empty list and let the next tool figure it out. Spelling that out in the planning doc made the conditional logic in `run_agent` obvious to implement.

**One divergence and why:** The spec suggested the agent could use an LLM to parse the user's query (extracting size and price from natural language). In practice, regex was simpler and faster — prices always appear as `$30` and sizes as `M` or `size M`, so there was no need to add a third LLM call just for parsing. This also makes the agent faster since each query already makes two LLM calls.

---

## File Structure

```
ai201-project2-fitfindr-starter/
├── data/
│   ├── listings.json          # 40 mock secondhand listings
│   └── wardrobe_schema.json   # Wardrobe format + example wardrobe
├── utils/
│   └── data_loader.py         # Helper functions for loading data
├── tools.py                   # Three tool implementations
├── agent.py                   # Planning loop and session state
├── app.py                     # Gradio UI
├── planning.md                # Agent design spec
└── requirements.txt           # Python dependencies
```

---

## AI Usage

**Instance 1 — Implementing `search_listings`**
I designed the scoring logic in `planning.md` first — deciding it should match keywords across title, description, category, and style tags, drop zero-score listings, and sort by score. I then gave Claude that spec and asked it to implement it. After it generated the code I tested it against three queries myself: a keyword match, a price-filtered query, and a nonsense string to confirm it returned empty. I accepted it after all three passed.

**Instance 2 — Debugging the Groq model**
Claude generated `suggest_outfit` and `create_fit_card` using the specs I wrote. When I ran the code I noticed both tools were returning fallback strings instead of real output. I recognized the silent `except` block was hiding the real error, so I stripped it out to expose the exception. The error showed the model `llama3-8b-8192` had been decommissioned. I asked Claude to update both tools to `llama-3.3-70b-versatile` and re-ran to confirm the LLM calls worked.

**Instance 3 — Implementing `run_agent`**
I designed the session dict structure and the conditional early-exit logic in `planning.md` before asking Claude to implement it. When Claude generated `agent.py` I checked that the early-exit condition specifically stopped the pipeline when `search_results` was empty — rather than passing empty data forward — which matched what I had written in the spec. I ran both the happy path and the no-results test to verify.
