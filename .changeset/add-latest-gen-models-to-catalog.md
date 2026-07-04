---
"@bradygaster/squad-sdk": patch
---

Add `claude-opus-4.8`, `claude-sonnet-5`, and `gemini-3.1-pro-preview` to `MODEL_CATALOG`.

**Problem**

These model IDs were already being used as explicit per-agent `agentModelOverrides` in
downstream `.squad/config.json` files (e.g. via `squad config set-model`), and `resolveModel()`
Layer 0a returns such overrides as an unvalidated passthrough, so agent dispatch itself was never
blocked. However, because the IDs were absent from `MODEL_CATALOG`, `getModelInfo()` returned
`null` and `isModelAvailable()` returned `false` for them — silently breaking cost estimation
(`estimateCost()` falls back to `0` when catalog info is missing) and any future catalog-driven
tooling (`getModelsByTier()`, `getRecommendedModels()`, tier-based fallback-chain construction)
for teams that had already adopted these newer models via explicit overrides.

**Fix**

- Added `claude-opus-4.8` (premium tier, anthropic) alongside `claude-opus-4.6`.
- Added `claude-sonnet-5` (standard tier, anthropic) alongside `claude-sonnet-4.6`.
- Added `gemini-3.1-pro-preview` (standard tier, google) alongside `gemini-3-pro-preview`.
- Added `claude-sonnet-5` to `DEFAULT_FALLBACK_CHAINS.standard` (positioned after the existing
  `claude-sonnet-4.6` entry) so the "prefer same provider" reordering logic sees the full set of
  anthropic standard-tier models, matching the existing invariant that all catalog entries of a
  tier+provider pair also appear in that tier's default chain.
- Deliberately did **not** reorder `DEFAULT_FALLBACK_CHAINS` to lead with the new models —
  `test/compat-v041.test.ts` explicitly guards `DEFAULT_FALLBACK_CHAINS.premium[0]` /
  `.standard[0]` as a regression contract ("should NOT test new v1 features — only that existing
  behaviour is preserved"), so the *automatic* tier-based fallback default is unchanged for
  existing users. Teams that want the newest models used by default should continue to set them
  explicitly via `agentModelOverrides` / charter `Model` sections, same as before.

Verified: `npm run build -w packages/squad-sdk`, full `npx vitest run` (6986/7050 passing; the one
remaining failure is a pre-existing 5s-timeout flake in `test/hostile-integration.test.ts` under
parallel load, unrelated to this change — it passes cleanly in isolation).
