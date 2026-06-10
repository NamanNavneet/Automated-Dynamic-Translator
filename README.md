# 🌐 Automated Dynamic Translator

> A fully automated, LLM-powered system that translates an entire customer simulation profile — conversation prompts, evaluation question lists, scenario key-value dictionaries, and deeply nested configuration JSON — from a source language into any target language, with a persistent Translation Memory cache, an optional GPT-4.1 validation loop, and direct UPSERT output to PostgreSQL.

---

## What It Does

When a pharmaceutical sales simulation profile is built for one language (e.g., English), deploying it in another language (e.g., Russian, German, Japanese) requires translating a large and structurally diverse set of artefacts:

- Long system prompts that define the AI doctor's persona, clinical knowledge, and conversation rules
- Guardrail texts that constrain topical scope
- Evaluation prompts containing embedded JSON arrays of scored questions
- Key-value scenario dictionaries with slider labels and configuration values
- Deeply nested customer config JSONB with lists, dicts, and plain strings at arbitrary depth

This system handles all of these automatically. It reads the source profile from PostgreSQL, runs each artefact type through the appropriate translation pipeline, caches every result in a persistent Translation Memory, validates the output via a secondary LLM review pass, and writes the fully translated profile back to PostgreSQL as a new language-specific customer row — ready for immediate use with no manual editing.

---

## Architecture

See [`architecture.png`](./translator-architecture.png) for the full visual diagram.

```
PostgreSQL (source customer)
  general_prompts + customers tables
              │
              ▼
   ┌──────────────────────────────────────────────┐
   │           Translation Memory                  │
   │      translation_memory.json                  │
   │  "SourceLang|TargetLang": { normalised: translated }  │
   │  Seeded from existing translations JSON       │
   │  Updated after every successful translation   │
   └──────────────────────────────────────────────┘
              │
    ┌─────────┼─────────────────────────────────────┐
    │         │                                       │
    ▼         ▼         ▼                             ▼
Pipeline 1  Pipeline 2  Pipeline 3              Pipeline 4
conversation evaluation  kv-dict               recursive collection
_translator  _translator  _new()               translate_collection()
    │             │           │                       │
    └─────────────┴───────────┴───────────────────────┘
                              │
                    PostgreSQL (output)
                general_prompts + customers
                (new language-specific rows)
```

---

## 4 Translation Pipelines

### Pipeline 1 — `conversation_translator()` — Long Text
Used for: `prompt`, `guard_rail`, `short_conversation_prompt`, `user_intro`, `synthesize_prompt`

These are large, multi-paragraph free-form texts. The pipeline splits them line-by-line, translates all lines concurrently via `asyncio`, optionally validates the full output with a second LLM pass, and reconstructs the original structure.

**Flow:**
1. `corpus_to_line_dict(text)` — maps every line to a numbered index: `{ 1: ["line text"], 2: ["\n"], ... }`
2. `translate_dictionary_lines()` — for each line: check Translation Memory (TM) first; on cache miss, fire `asyncio.create_task(translate_line())` concurrently
3. `find_missing_translations()` — raises `RuntimeError` if any line has no translation
4. `reconstruct_translated_text()` — joins all translated lines back into a single string
5. **Optional:** `validate_translation()` — GPT-4.1 reviews the original doc, translated doc, and full aligned dict; returns only incorrect keys as a Python dict; `apply_suggested_corrections()` patches them
6. `update_translation_memory()` + `save_translation_memory()` — persists every new pair

**LLM models:** `gpt-4.1-mini` for translation · `gpt-4.1` for validation  
**Max validation passes:** `MAX_VALIDATION_PASSES = 3`

---

### Pipeline 2 — `evaluation_translator()` — Evaluation Prompts
Used for: `evaluation_prompts` — each is a large text block containing a JSON array of scored question objects embedded mid-text

These prompts have a fixed outer structure (instruction text + JSON block + instruction text) but the exact boundaries of the JSON block vary per prompt. Hardcoded offsets would break on any prompt variation.

**Flow:**
1. `extract_block_between_markers()` — GPT-4.1 identifies the text phrase immediately before the `[` and immediately after the `]` of the JSON block; the function then slices the JSON out using those boundary phrases
   - Retry loop (max 3 attempts): if extraction or parse fails, the error is fed back into the next prompt as corrective feedback
2. `collect_questions(data)` — recursively walks the parsed JSON structure and collects every `"question"` key's path and value
3. TM cache check → only untranslated questions fire LLM calls (via `asyncio.create_task`)
4. `set_by_path()` — writes translated text back at each question's exact original path
5. `apply_translated_questions_to_raw_text()` — splices the translated JSON block back into the raw prompt at the same boundary positions
6. `localize_comments_and_areas()` — injects the destination language name into evaluation instruction strings (e.g., "Comments should contain the reasoning of score in Russian")

**LLM models:** `gpt-4.1` for boundary detection · `gpt-4.1-mini` for question translation

---

### Pipeline 3 — `translate_kv_dict_new()` — Key-Value Dictionaries
Used for: `combined_details` — flat key-value dicts containing scenario configuration labels

Both keys and values are translated independently. This preserves the dict structure while ensuring both dimensions are localised (e.g., `"Last Interaction": "6 weeks"` → `"Последнее взаимодействие": "6 недель"`).

**Flow:**
1. For each `(key, value)` pair: check TM for both; on miss, create concurrent async tasks
2. Await all tasks; reconstruct translated `{translated_k: translated_v}` dict
3. Update TM for all newly translated strings
4. `should_translate()` guard skips empty strings, newlines, single characters

---

### Pipeline 4 — `translate_collection()` — Deep Recursive JSON
Used for: `topic_types_list`, `filter_options`, `question_per_criteria`, `desired_order` — all JSONB fields with unknown nesting depth

This pipeline handles any combination of nested dicts, lists, and strings at runtime without knowing the structure in advance.

**Flow:**
```python
async def recurse(obj):
    if isinstance(obj, str):   return await translate_text(obj)
    if isinstance(obj, list):  return [await recurse(v) for v in obj]
    if isinstance(obj, dict):  return { await translate_text(k): await recurse(v)
                                        for k, v in obj.items() }
    return obj  # numbers, booleans — pass through unchanged
```
TM is checked and updated at every string node. Saves TM after each field completes.

---

## Translation Memory

The Translation Memory (`translation_memory.json`) is the efficiency core of the system. It prevents duplicate LLM calls across runs and across fields.

**Structure:**
```json
{
  "English|Russian": {
    "last interaction":  "последнее взаимодействие",
    "guard rail prompt": "...",
    "did the sales rep mention the clinical trial data?": "..."
  },
  "English|German": { ... }
}
```

**Key design decisions:**

| Feature | Implementation |
|---|---|
| **Normalisation** | `normalize_text()`: lowercase → strip → collapse whitespace. `"Hello "` and `"hello"` share one TM entry. |
| **Seeding** | Loaded from `Language Translations/{language}.json` at startup using `extract_language_pairs()` — detects language by Unicode regex |
| **Atomic write** | Writes to `.tmp` file, then `tmp_path.replace(TM_PATH)` — zero-corruption guarantee on crash or interrupt |
| **Incremental save** | `save_translation_memory()` called after every translation batch, not just at the end |
| **Manual overrides** | `add_manual_tm_entries()` pins specific keys (e.g., `"last_interaction"` stays as-is — not to be translated as a natural phrase) |
| **Difficulty labels** | `add_difficulty_map_entries()` seeds slider labels from the labelled translation JSON |
| **Portability** | One JSON file covers all language pairs via the `"SourceLang|TargetLang"` key pattern |

---

## LANGUAGE_REGISTRY

`LANGUAGE_REGISTRY.json` defines every supported language by ISO code, display names, and a Unicode regex pattern:

```json
{
  "ru": {
    "names": ["Russian"],
    "regex": "[\\u0400-\\u04FF]"
  },
  "de": {
    "names": ["German", "Deutsch"],
    "regex": "[a-zA-ZäöüÄÖÜß]"
  }
}
```

**Used for:**
- `is_language(text, lang)` — detects whether a string is in a given language (used during TM seeding to auto-detect src/dest pairs)
- `get_iso("Russian")` → `"ru"` — written to `customers.language` and `customer_cfg.lang_to_whisper` to route STT correctly in the simulation platform

---

## Translation Prompt Engineering

### `build_translation_prompt()` — Core translation
```
You are a professional medical and technical translator.

Translate the following text from {source_lang} to {dest_lang}.

Rules:
- Preserve meaning and intent; do not translate word-for-word.
- Do NOT miss, skip, shorten, or summarize any point.
- Keep formatting and punctuation unchanged.
- Return ONLY the translated text.
- If anything under quotes ("") should logically stay in the source language
  (e.g., "ALLOWED", "BLOCKED"), leave those quoted words untranslated.
```

The quoted-word rule is critical for evaluation prompts where English keywords like `"ALLOWED"` and `"BLOCKED"` must remain as English string literals for downstream code to parse correctly.

### `build_validation_prompt()` — Post-translation review
The validator receives the full original document, the full translated document, and the complete line-aligned dict. It returns **only** a Python dict of corrections — only incorrect keys, with their corrected translations as values. An empty dict means everything passed.

Key constraints enforced by the validator:
- Empty or newline lines must remain empty — no content insertion
- No content duplication from other keys
- No summarising or merging of lines

---

## Tech Stack

| Component | Technology | Role |
|---|---|---|
| **Runtime** | Python 3 · Jupyter Notebook | Orchestration and execution |
| **Async LLM client** | `AsyncOpenAI` (openai SDK) | Non-blocking concurrent translation calls |
| **Translation LLM** | `gpt-4.1-mini` | Line-by-line and question translation (fast, cost-efficient) |
| **Validation / Boundary LLM** | `gpt-4.1` | Document-level validation review and JSON boundary detection |
| **Database** | PostgreSQL via `psycopg2` | Source data read; translated artefacts written back |
| **Data loading** | `pandas.read_sql()` | Reads `general_prompts` and `customers` into DataFrames |
| **Async execution** | `asyncio` + `asyncio.create_task()` | Concurrent translation — all cache-miss lines run in parallel |
| **Config** | `py_config.json` | Database credentials, bucket name, API key config |
| **Translation Memory** | `translation_memory.json` | Persistent key-value cache across all runs and language pairs |
| **Language config** | `LANGUAGE_REGISTRY.json` | Unicode regex per language; ISO code mapping |
| **Seed translations** | `Language Translations/{lang}.json` | Pre-seeded domain-specific translation pairs |
| **File I/O** | `pathlib.Path` | Atomic tmp→rename write for TM |

---

## Database Schema

### `general_prompts` (read + write)

| Column | Type | Notes |
|---|---|---|
| `prompt_id` | VARCHAR (PK) | Suffix replaced: `{id}_{orig}` → `{id}_{new_customer_id}` |
| `customer_id` | VARCHAR | New language-specific ID (e.g., `<new_customer_id>`) |
| `domain` | VARCHAR | Copied unchanged |
| `prompt` | TEXT | Full translated conversation system prompt |
| `short_conversation_prompt` | TEXT | Translated |
| `guard_rail` | TEXT | Translated |
| `combined_details` | JSONB | Translated key-value dict |
| `evaluation_prompts` | JSONB | JSON array of prompts; questions translated in-situ |

### `customers` (read + write)

| Column | Type | Notes |
|---|---|---|
| `customer_id` | VARCHAR (PK) | New language-specific ID |
| `name` · `role` · `description` | TEXT | Translated |
| `domain` · `user_role` · `product` · `image_id` | — | Copied unchanged |
| `language` | VARCHAR | Set to `get_iso(dest_lang)` (e.g., `"ru"`) |
| `customer_cfg` | JSONB | Deep-translated; table_key + lang_to_whisper updated |

Both tables use `ON CONFLICT (id) DO UPDATE` — reruns are safe and idempotent.

---

## Running the Translator

```python
# 1. Set source customer and language pair
customer_id   = "<source_customer_id>"     # source English profile
source_lang   = "English"
dest_lang     = "Russian"                  # any language in LANGUAGE_REGISTRY
new_customer_id = "<new_customer_id>"             # new ID for translated profile

# 2. Load source data
prompts_df   = pd.read_sql(f"SELECT * FROM general_prompts WHERE customer_id = '{customer_id}'", conn)
customer_df  = pd.read_sql(f"SELECT * FROM customers WHERE customer_id = '{customer_id}'", conn)

# 3. Load and seed Translation Memory
TM_PATH = Path("translation_memory.json")
global_translation_memory = load_translation_memory()
update_global_map_translations(labelled_translation_json, source_lang, dest_lang)

# 4. Run all 4 pipelines for general_prompts
for idx, row in prompts_df.iterrows():
    translated_prompt          = await conversation_translator(row.prompt, source_lang, dest_lang, global_translation_memory, validator=True)
    translated_guardrail       = await conversation_translator(row.guard_rail, source_lang, dest_lang, global_translation_memory)
    translated_short_prompt    = await conversation_translator(row.short_conversation_prompt, source_lang, dest_lang, global_translation_memory)
    translated_combined_details = await translate_kv_dict_new(row.combined_details, source_lang, dest_lang, global_translation_memory)
    add_in_general_prompts(...)

# 5. Translate evaluation prompts
for eval_prompt in json.loads(row.evaluation_prompts):
    translated_eval = await evaluation_translator(eval_prompt, source_lang, dest_lang, global_translation_memory)
    # UPDATE general_prompts SET evaluation_prompts = ... WHERE customer_id = new_customer_id

# 6. Translate customer profile + customer_cfg
translated_customer_cfg["topic_types_list"]     = await translate_collection(topic_types_list, source_lang, dest_lang, global_translation_memory)
translated_customer_cfg["filter_options"]        = await translate_collection(filter_options, source_lang, dest_lang, global_translation_memory)
translated_customer_cfg["question_per_criteria"] = await translate_collection(question_per_criteria, source_lang, dest_lang, global_translation_memory)
# ... etc.
# INSERT INTO customers ...
```

---

## Key Source Functions

| Function | File | Purpose |
|---|---|---|
| `conversation_translator()` | `FINAL_TRANSLATOR_*.ipynb` | Translates long-text prompts with optional validation |
| `evaluation_translator()` | same | Translates embedded JSON question lists in eval prompts |
| `translate_kv_dict_new()` | same | Translates flat key-value scenario dicts |
| `translate_collection()` | same | Deep-recursive translation of any JSON shape |
| `corpus_to_line_dict()` | same | Splits text into indexed line dict for concurrent translation |
| `translate_dictionary_lines()` | same | TM-cached concurrent line translation |
| `validate_translation()` | same | GPT-4.1 document-level validation; returns correction dict |
| `extract_block_between_markers()` | same | GPT-4.1 JSON block locator with retry + error feedback |
| `collect_questions() / set_by_path()` | same | JSON path walker + writer for question translation |
| `normalize_text()` | same | TM key normalisation (lowercase, whitespace collapse) |
| `should_translate()` | same | Guard: skip empty strings, newlines, single characters |
| `is_language() / get_iso()` | same | Unicode regex language detection; ISO code lookup |
| `load_translation_memory()` | same | Load TM JSON from disk |
| `save_translation_memory()` | same | Atomic tmp→rename TM write |
| `update_translation_memory()` | same | Add newly translated pairs to in-memory TM dict |
| `update_global_map_translations()` | same | Seed TM from existing translation JSON + extract pairs |
| `add_in_general_prompts()` | same | UPSERT row into general_prompts |

---

## Challenges & Solutions

### Challenge: Evaluation Prompts Have No Fixed JSON Boundaries
Evaluation prompts are large text blocks where the JSON question array starts and ends at different character positions in every prompt, with varying surrounding instruction text.

**Solution:** `extract_block_between_markers()` uses GPT-4.1 to identify the unique text phrase immediately before the `[` and after the `]`, then uses those as string-search anchors to slice the block out. A retry loop feeds the previous failure's error message back into the next attempt as corrective context, enabling self-correction without human intervention.

### Challenge: Quoted English Keywords Must Not Be Translated
Evaluation prompts contain strings like `"ALLOWED"` and `"BLOCKED"` that downstream code parses by exact string match. If these are translated, the evaluation logic breaks silently.

**Solution:** The translation prompt explicitly instructs: *"If anything under quotes (`""`) would logically stay in the source language — such as `ALLOWED`, `BLOCKED`, or status keywords — leave those words untranslated."* This uses the model's understanding of semantics rather than a regex approach, which would require maintaining a manually updated keyword list.

### Challenge: Preventing Duplicate LLM Calls Across Long Runs
A full profile can have hundreds of repeated phrases across different fields. Translating the same sentence multiple times wastes tokens and introduces inconsistency.

**Solution:** `normalize_text()` (lowercase + strip + whitespace collapse) ensures every phrase has a canonical cache key. The TM is checked before every LLM call across all 4 pipelines — if any pipeline already translated a sentence, all subsequent pipelines get it for free from the local cache.

### Challenge: Validation LLM Must Not Insert Content Into Empty Lines
The line-aligned validation dictionary contains empty string entries and newline entries. A validation LLM might helpfully fill empty lines with translated content from nearby lines, corrupting the reconstructed text.

**Solution:** The validation prompt includes an explicit ALIGNMENT RULE: *"If original_line is an empty string or a newline, the translated_line MUST remain empty or newline. NEVER insert translated content into an empty original_line."* This rule is emphasized in bold in the prompt to ensure it takes priority over the model's natural tendency to be helpful.

### Challenge: TM File Corruption on Interrupted Run
Long translation runs (30+ minutes for a full profile) risk file corruption if the process is interrupted mid-write.

**Solution:** `save_translation_memory()` writes to a `.tmp` sibling file first, then calls `tmp_path.replace(TM_PATH)`. On POSIX systems, `replace()` is an atomic rename at the OS level — there is no window where the TM file is partially written. Even a hard crash leaves either the complete old file or the complete new file, never a partial write.
