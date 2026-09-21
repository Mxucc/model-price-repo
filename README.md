# model-price-repo

Filtered model pricing data for CRS and sub2api projects. Syncs from the upstream [litellm](https://github.com/BerriAI/litellm) pricing file on a schedule, applying configurable prefix filters to keep only the models you actually use.

## How it works

A GitHub Actions workflow runs every 10 minutes (and on manual trigger):

1. Downloads the full `model_prices_and_context_window.json` from litellm
2. Filters models by the prefix rules in `config.json`
3. Merges new models into the existing output (additive — never removes)
4. Applies alias mappings and custom model definitions
5. Writes the output JSON + SHA-256 hash, commits only if content changed

## Configuration

All settings live in [`config.json`](config.json):

| Field | Description |
|---|---|
| `upstream_url` | URL to the upstream litellm pricing JSON |
| `output_file` | Output filename (default: `model_prices_and_context_window.json`) |
| `hash_file` | SHA-256 hash filename for change detection |
| `sync_mode` | `"additive"` (only add new) or `"full"` (replace each run) |
| `update_existing` | Whether to update pricing data for models already in the output |
| `prefix_filters` | List of prefixes — a model key must start with one to be included |
| `exclude_patterns` | Substring patterns to exclude (applied before prefix matching) |
| `aliases` | Map alias model keys to existing source models (deep copy pricing) |
| `custom_models` | Manually defined pricing objects, always injected |

### Adding new model prefixes

Edit the `prefix_filters` array in `config.json`:

```json
{
  "prefix_filters": [
    "claude-",
    "gpt-",
    "your-new-prefix/"
  ]
}
```

### Adding aliases

Aliases create copies of an existing model's pricing under a new key:

```json
{
  "aliases": {
    "claude-opus-4-6-thinking": {
      "source": "claude-opus-4-6",
      "description": "Thinking variant, same pricing"
    }
  }
}
```

If the source model doesn't exist in the filtered data, the alias is skipped with a warning.

## Hand-maintained prices (`custom_models`)

`custom_models` in `config.json` is the place to pin prices that must not follow
upstream. Entries there are written **last**, so they win over both the existing
output and the upstream file, and `update_existing: false` keeps a model that is
already published from being repriced by an upstream sync. Two entries currently
exist for that reason:

- `codex-auto-review` — an internal Codex model. It is aliased from `gpt-5.6-luna`,
  and this entry cancels the alias's inherited service-tier, cache-write and
  long-context fields (a `null` value removes a field) so no public GPT-5.6 tier
  pricing is inferred for it.
- `gemini-3.6-flash` — pinned at 2× the currently published upstream rate. Upstream
  re-sourced this model from the Gemini API docs to the Gemini Enterprise Agent
  Platform docs, which quote exactly half. The pin is deliberate; delete the entry
  to adopt upstream pricing.

## Declarative billing expressions (`billing_expr`)

sub2api supports an optional `billing_expr` field on any model entry: a single
expression that replaces the per-token rates for that model and can encode
time-of-day pricing (peak/off-peak windows), context-length tiers and cache/image
rates. Put it in `custom_models` and the sync preserves it like any other field.

Example — DeepSeek bills double rate on weekdays 09:00-12:00 and 14:00-18:00
Beijing time, and half rate at all other times:

```json
{
  "deepseek-flash": {
    "billing_expr": "v1:(weekday(\"Asia/Shanghai\") >= 1 && weekday(\"Asia/Shanghai\") <= 5 && ((hour(\"Asia/Shanghai\") >= 9 && hour(\"Asia/Shanghai\") < 12) || (hour(\"Asia/Shanghai\") >= 14 && hour(\"Asia/Shanghai\") < 18))) ? tier(\"peak\", p * 0.30 + cr * 0.006 + c * 1.20) : tier(\"off_peak\", p * 0.15 + cr * 0.003 + c * 0.60)"
  }
}
```

Coefficients are real USD per million tokens; `p` is input, `c` is output, `cr`
is cache read, `len` is the full input context length (use it, not `p`, for tier
conditions). Group and channel pricing configured in sub2api still overrides the
expression. Full grammar: `docs/MODEL_BILLING_EXPRESSIONS.md` in the sub2api
repository.

Note: sub2api ships a built-in DeepSeek expression, and its `deepseek-v4-pro` →
Flash routing switch lives on the built-in side. Adding a `billing_expr` for
`deepseek-v4-pro` here would disable that switch, so don't unless you mean to.

## Running locally

```bash
python3 scripts/sync_prices.py --config config.json --repo-root .
```

No pip dependencies — uses Python standard library only.

## CRS integration

Point CRS to the raw output file from this repo:

```
MODEL_PRICES_URL=https://raw.githubusercontent.com/<owner>/model-price-repo/main/model_prices_and_context_window.json
```

The output JSON structure is identical to what litellm produces (model key -> pricing object), so CRS `pricingService.js` works without changes.

## License

[MIT](LICENSE)
