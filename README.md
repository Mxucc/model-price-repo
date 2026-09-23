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
already published from being repriced by an upstream sync. Entries currently exist
for these reasons:

- `codex-auto-review` — an internal Codex model. It is aliased from `gpt-5.6-luna`,
  and this entry cancels the alias's inherited service-tier, cache-write and
  long-context fields (a `null` value removes a field) so no public GPT-5.6 tier
  pricing is inferred for it.
- `gemini-3.6-flash` — pinned at 2× the currently published upstream rate. Upstream
  re-sourced this model from the Gemini API docs to the Gemini Enterprise Agent
  Platform docs, which quote exactly half. The pin is deliberate; delete the entry
  to adopt upstream pricing.
- `deepseek-flash` / `deepseek-v4-flash` / `deepseek-v4-flash-vision-exp` /
  `deepseek-v4-pro` / `deepseek-chat` / `deepseek-reasoner` — DeepSeek's rates and
  its peak/off-peak rule are defined here as `billing_expr` (see below), so the
  published card and the charged amount can no longer drift apart.
- `glm-5` / `glm-5-turbo` / `glm-5.1` / `glm-5.2` / `glm-5.3` / `kimi-k2.5` /
  `kimi-k2.6` / `kimi-k2.7-code` / `kimi-k3` / `minimax-m2.7` / `grok-4.3` /
  `grok-4.5` / `grok-4.6` / `grok-4.7` / `grok-4.20-*` / `grok-build-0.1` — these
  models are not covered by `prefix_filters` (litellm has no GLM/Kimi/MiniMax/bare-grok
  entry for them), so a hand-maintained entry is the only way they get a published
  price. `glm-5`, `glm-5-turbo`, `glm-5.2`, `glm-5.3`, `kimi-k2.7-code` and `kimi-k3`
  came from upstream `Wei-Shaw/model-price-repo` (PR #17, official reference prices);
  `grok-4.7` was added to match sub2api v0.2.8's own audited card (identical to
  `grok-4.6`: <200k $2.00 / $0.50 cached / $6.00, >=200k $4.00 / $1.00 / $12.00); the
  rest were added here earlier.

## Merging upstream (`Wei-Shaw/model-price-repo`)

This repository is a fork of upstream `Wei-Shaw/model-price-repo` (common ancestor
`10706f3`), so upstream syncs and hand-written reference prices can be pulled in with
a normal `git merge upstream/main`. Resolve it like this:

1. `config.json` merges cleanly — keep every local `custom_models` / `aliases` entry,
   because that is where the audited pins live.
2. For the generated catalog take **upstream's** snapshot (it is the fresher one:
   newer `*_batches` fields and newly published models), then re-run
   `scripts/sync_prices.py` on it with the merged config
   (`python3 scripts/sync_prices.py --config config.json --repo-root .` in a scratch
   copy first). Because `custom_models` is written last and
   `update_existing: false` never reprices a published model, the result only **adds**
   fields/models — no published rate moves. Verify that before committing: the model
   count must not shrink, `deepseek-*` must still carry `billing_expr`, and
   `gemini-3.6-flash` must still be 2× upstream.

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

An expression the entry does **not** mention is not silently free: a variable the
expression never references stays inside `p` and is billed at the input rate. That
is why DeepSeek's expressions carry a trailing ` + cc * 0 + cc1h * 0` — DeepSeek
charges nothing for a cache write, so the term has to be written out to keep those
tokens out of `p`.

### DeepSeek

Official card (<https://api-docs.deepseek.com/zh-cn/quick_start/pricing>): the live
models are `deepseek-flash` (V4.1-Flash) and `deepseek-v4-pro` (V4-Pro-0813); peak
is Beijing Mon-Fri 09:00-12:00 & 14:00-18:00 (excluding Chinese public holidays),
everything else — including weekends and holidays — is off-peak, and off-peak is
exactly half.

`config.json` defines both cards here, in USD per MTok at the rate sub2api has
always billed (flash 1/4/0.02 CNY per MTok → $0.15/$0.60/$0.003; pro
4.5/13.5/0.15 CNY → $0.66/$1.98/$0.022):

- `deepseek-flash` — peak `p*0.30 + cr*0.006 + c*1.20`, off-peak
  `p*0.15 + cr*0.003 + c*0.60`
- `deepseek-v4-pro` — peak `p*1.32 + cr*0.044 + c*3.96`, off-peak
  `p*0.66 + cr*0.022 + c*1.98`
- the retired aliases `deepseek-v4-flash`, `deepseek-v4-flash-vision-exp` (model
  offline, requests served by V4.1-Flash) and `deepseek-chat`, `deepseek-reasoner`
  bill at Flash rates, per footnote 1 of the official card

The per-token rate fields on these entries are the **off-peak baseline** and are
used for display only; the expression is what charges a request. `deepseek-v4-pro`
was routed to V4.1-Flash by upstream between 2026-09-14 and the card's restoration
— the hardcoded switch for that in sub2api is gone, so the Pro card above is what
now applies.

`deepseek-v3-2-251201` (provider `volcengine`) deliberately has **no** expression
here and its upstream rates are 0. sub2api treats any `deepseek-` prefix as
DeepSeek's own card, so to bill it from Volcengine's numbers you must first fill in
real rates and then add `"billing_expr": ""` (an empty string means "bill from this
entry's own rates, do not let the built-in rule take over") — declaring the empty
string without rates would make the model free.

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
