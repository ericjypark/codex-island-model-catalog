# codex-island-model-catalog

Model prices for [CodexIsland](https://github.com/ericjypark/codex-island),
published as a plain JSON file so a new model does not require an app release.

**Endpoint:** https://ericjypark.github.io/codex-island-model-catalog/v1/models.json

```json
{
  "schemaVersion": 1,
  "generatedAt": "2026-07-26T04:16:19Z",
  "source": "litellm + overrides",
  "models": {
    "claude-opus-4-8": {
      "displayName": "Opus 4.8",
      "inputPerMillion": 5,
      "outputPerMillion": 25,
      "cacheCreationPerMillion": 6.25,
      "cacheReadPerMillion": 0.5
    }
  }
}
```

Prices are US dollars per million tokens. Keys are canonical model ids with
any 8-digit date suffix stripped (`claude-haiku-4-5-20251001` →
`claude-haiku-4-5`), matching how the app normalizes the ids it reads out of
local CLI session logs. Ids carrying a dashed date (`gpt-5-2025-08-07`) are
published as-is, because the app does not strip those either — collapsing them
would make the app look up an id this file never publishes and silently price
it at $0.

`schemaVersion` is 1 and will stay 1. A breaking change to this shape ships at
`/v2/models.json` so installed apps keep working.

## How a price gets here

A bot refreshes this file every six hours. You do not need its source to check
a value — both inputs are public and the rules are below.

1. Fetch [LiteLLM's price table][litellm].
2. Keep entries whose `mode` is `chat` or `responses`, whose id contains no
   `/` (that drops `azure/`, `bedrock/`, `vertex_ai/` re-listings of the same
   model), and whose id matches a pattern in [`config.json`](config.json).
   The `gpt-[0-9]*` pattern covers numbered GPT generations, including GPT-6
   and future generations, so a major-version launch needs no filter update.
3. Convert per-token prices to per-million by multiplying by 1,000,000, then
   round to 8 decimal places. The rounding exists so a price reads as `0.2`
   rather than `0.19999999999999998` — binary floating point leaves residue
   that would otherwise be published verbatim into a file meant to be read.
4. Where LiteLLM lists no cache-write rate, use the input rate: OpenAI bills
   cache writes at the standard input rate. Where it lists no cache-read rate,
   use 0.
5. Apply [`overrides.json`](overrides.json) on top, field by field.
6. Generate `displayName` from the id unless an override supplies one.

[litellm]: https://github.com/BerriAI/litellm/blob/main/model_prices_and_context_window.json

## Correcting a price

Open a PR against [`overrides.json`](overrides.json). An override beats
whatever upstream says, and only the fields you list are replaced:

```json
{ "gpt-5.6": { "cacheCreationPerMillion": 6.25 } }
```

An override can also introduce a model LiteLLM does not list at all — supply
all four rates and a `displayName` in that case.

The entries there now are deliberate: OpenAI began billing cache writes at
1.25× input starting with 5.6, and LiteLLM lists no cache-read rate for the
pro tier because those models have no prompt caching.

## Why you can trust the numbers

The app's release cycle used to be a human review gate on every price change.
This file removes that gate, so the bot carries its own:

- If the merged model count falls below half of what is already published, the
  whole run is rejected and nothing is committed.
- A negative rate, or any rate above $1000 per million, is rejected.
- A rate that moved by 10× or more from its published value is rejected.
- **A rejected model keeps its previously published value** rather than
  disappearing. A missing model prices to $0 in the app, which is the failure
  this whole arrangement exists to prevent.
- **Models are never removed.** If LiteLLM drops an entry, the published value
  stays until a human removes it deliberately.
- The file is committed only when a price actually changes, so `generatedAt`
  marks the last real change rather than the last time the bot ran.

Override values deliberately skip the absolute ceiling — that is what lets a
genuinely expensive model exceed it — so review of the PR is the control on
those. Negative values are still rejected.

## Repository contents

| Path | What it is |
|---|---|
| `v1/models.json` | Generated. The only file the app reads. |
| `overrides.json` | Hand-maintained. Beats LiteLLM. Open a PR here. |
| `config.json` | Which model id patterns are tracked. |
