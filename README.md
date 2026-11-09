# Claude Code flag map (2.1.287)

What every `tengu_*` flag that Claude Code 2.1.287 actually reads appears to do, one line each. The full table is in [`flagmap-2026-11-09.csv`](flagmap-2026-11-09.csv).

Built on 2026-11-09 from the installed `claude.exe` (Windows, npm install) and one account's `~/.claude.json`. It's a follow-up to the r/ClaudeCode thread [I don't think Opus 5.5 was nerfed, they're just experimenting](https://www.reddit.com/r/ClaudeCode/comments/1xr66pd/).

## How it was made

- `~/.claude.json` caches 1,204 flags under `cachedGrowthBookFeatures`. **Only 487 of them are read anywhere in this binary.** The other 708 never appear in the binary at all, not even as fragments. They're presumably for other clients or versions.
- For each of the 487, I found the real code call sites (`R("tengu_x", default)` style reads, plus constants that hold the key) and skipped places where the key only shows up in the compiled string table. Then I read the surrounding code: the default value, any `CLAUDE_CODE_*` env override, and the text the branch turns on.
- **confidence**: `E` means the code says it directly. `I` means it's inferred from context, so treat it as a guess.
- **value**: this is **one account's served value**, not the default. Your assignments will differ.

## Columns

| column | meaning |
|---|---|
| key | flag name |
| value | served value for the account this was built on (2026-11-09) |
| confidence | E (from code) or I (inferred) |
| what_it_does | one-line read of the code |

## Findings relevant to the thread

- **There are two caches.** `cachedGrowthBookFeatures` is the global flag set. `clientDataCacheSlots` is a per-model, per-entrypoint, per-org payload that carries the `atis` experiment ID (sent as the `x-cc-atis` header) **together with** dynamic system-prompt text such as `tengu_heron_brook`. Several readers check the per-model slot **first** and only fall back to GrowthBook if it's missing.
- **`tengu_ochre_wren` picks the opening line of the system prompt** (the "intro_frame" arm). When true, the opener is "You are an agent working with the user toward their goals, using your own judgment along the way." When false, the default, it's "You are an interactive agent that helps users with software engineering tasks." I confirmed this directly: setting `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` stopped the flag fetch, and the opener flipped on the same CC version.
- **`tengu_thistle_grebe` is a subagent steer** with three values: `default`, `no_nudges` and `counter_steer`. Priority order: env var, then **the per-model clientData slot**, then GrowthBook, then a per-model floor. `counter_steer` injects a "## Delegating to subagents" section and logs `tengu_subagent_steer_applied`.
- **Opus 5 has its own prompt bundle** (`opus_5_prompt_bundle` / `opus_5_5_prompt_bundle` model capabilities) with sections named `overcorrection` ("# Corrections"), "# Delivering work" and `opus5_reduced_delegation`. **`tengu_fennel_godwit` is the kill switch** for that bundle.
- **A "bash-first" steering group** is assigned per model and cohort: `tengu_cozy_teapot` (strict/relaxed), `tengu_gorse_plover`, `tengu_willow_tern` (also readable from the per-model slot), and `tengu_thrifty_sonic` (a model capability).
- **Usage-limit text the model sees and you don't:** `tengu_lantern_wick_text` (wrap-up instructions during a grace window) and `tengu_lantern_wick_release` ("Earlier usage-limit notes no longer apply. Continue working normally. Don't mention this to the user."). Both have `_by_model` variants.
- **`tengu_hushed_lark` / `tengu_hushed_lark_text`:** the "user hasn't heard from you in a while" reminder, which fires after 5 silent turns. Its text is served from the server.
- **`tengu_amber_tally`:** the usage meter's exchange rates, i.e. 5-hour credits per plan and credits per million tokens per model.

*The table holds flag names, served values, and descriptions only.*
