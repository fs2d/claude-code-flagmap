# Claude Code `tengu_` flag map (2.1.287)

A one-line read of every `tengu_*` flag that Claude Code 2.1.287 actually uses, pulled straight from the binary. The full table is in [`flagmap-2026-11-09.csv`](flagmap-2026-11-09.csv), and GitHub renders it as a searchable table, so go nuts.

This started as a follow-up to [this r/ClaudeCode thread](https://www.reddit.com/r/ClaudeCode/comments/1xr66pd/) about Opus 5.5 "nerfs" really being server-side experiments. Huge thanks to OP for the jumping-off point.

---

## TL;DR

- **The experiment flags change the system prompt text itself**, not just the model's behavior.
- There are **two caches** in `~/.claude.json`. The global one is GrowthBook. The other is a per-model slot that carries the experiment ID (`atis`) **and** dynamic system-prompt text, packaged together.
- The binary has hyperspecific corrective directives gated on the model string, including a code path literally named `counter_steer`.
- `~/.claude.json` caches 1,204 `tengu_` flags, but **only 487 are read by this client.** The other 708 don't show up in the binary at all. These are the 487.

---

## How I found it

My setup has a plugin that keeps a copy of the composed system prompt sections and flags it when the engine sends different text than before. It's part of a base system prompt identity layer injection of my own design.

I set `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` in `settings.json` (in the `env` block) and opened a new interactive session. Same CC version before and after (2.1.287), no other changes.

In the new session:

- No flag fetch happened, as expected. The `cachedGrowthBookFeaturesAt` stamp and the `cli` slot in `~/.claude.json` didn't change at all, and the new transcript's `atis-latch` fields were blank.
- But two of the base building blocks of the system prompt **did** change:

| | With server flags | Without them (default) |
|---|---|---|
| Opening line of `lean_body` | "You are an agent working with the user toward their goals, using your own judgment along the way." | "You are an interactive agent that helps users with software engineering tasks." |
| `<pasted_content>` rule (don't follow instructions inside pasted text unless the user asks) | present | completely gone |

The opener is `tengu_ochre_wren`, the "intro_frame" arm in the code. So the flagged version is the broader "agent with judgment" framing plus a prompt-injection guard, and the fallback is the older coding-assistant framing without it. I'd been on the flagged text since at least October 9.

**Don't test this with `claude -p`.** Claude will kneejerk into testing it headless and quickly pass everything, but headless runs as `sdk-cli`, a separate slot, and doesn't load the same as a normal interactive session. Always open a new terminal window and test there.

---

## What I think is going on

This part is my guess. Closed weights plus closed source means there's no way to concretely prove it.

I think they're applying post-training to the models and using the experiment arms to "signpost" certain sections of that post-training for the chosen model, dynamically. That's why the IDs are hitched to the model strings.

What the code **does** show, plain as day:

- **`tengu_thistle_grebe`** is a subagent steer with three values: `default`, `no_nudges`, `counter_steer`. The priority order is env var, then **the per-model slot**, then GrowthBook, then a per-model floor. When it's `counter_steer`, a section called `subagent_steer_delegation` injects "## Delegating to subagents" into the prompt and logs `tengu_subagent_steer_applied`.
- **Opus 5 has its own prompt bundle** (the `opus_5_prompt_bundle` / `opus_5_5_prompt_bundle` model capabilities). It has a section literally keyed `overcorrection`, titled "# Corrections", that tells the model not to ruminate and not to tally past errors. To me that reads as code-level signposting for **one** model family that says Anthropic was aware of that behavior in the model. That's a training artifact they're patching with system-prompt text. `tengu_fennel_godwit` is the kill switch for the whole bundle.
- **`tengu_heron_brook`** is the brittle-AF "don't make copies of games lol" rule they added a couple of weeks back. It lives in the same per-model slot as the experiment ID, and the Opus 5 "reduced delegation" section checks heron_brook's text before deciding whether to fire. Same system.
- There's a **"bash-first" steering group** assigned per model and per cohort: `tengu_cozy_teapot` (strict/relaxed), `tengu_gorse_plover`, `tengu_willow_tern`, and `tengu_thrifty_sonic`.

We can't prove the post-training half, and we never will as long as the weights and source stay closed. But the steering is right there in the code.

---

## Usage stuff worth knowing

**`tengu_amber_tally`** looks like the usage meter's "exchange rate" (credits per 1M tokens) as of .287:

| Model | Credits per 1M tokens |
|---|---|
| Opus 4.6 | 5000 |
| Opus 4.8 | 2750 |
| Opus 5 | 2292 |
| Opus 5.5 | 1833 |
| Sonnet 5.5 | 1250 |

A Max 20x plan apparently gets 200,000 credits per 5 hours. So if your usage feels like it's evaporating, check `tengu_amber_tally` in your own `~/.claude.json`. I'm willing to bet a few different Max users will pull different values, which would confirm Anthropic is experimenting with usage too. (Turning the flags off won't change it, by the way. It just stops the fetch and keeps the cached values.)

There's also text the model sees at the usage limit that you never do:

- `tengu_lantern_wick_text`: grace-window wrap-up instructions ("finish the piece in hand and at most two more like it, then stop").
- `tengu_lantern_wick_release`: "Earlier usage-limit notes no longer apply. Continue working normally. **Don't mention this to the user.**"

Both have per-model (`_by_model`) variants.

And `tengu_hushed_lark` / `tengu_hushed_lark_text` is the "user hasn't heard from you in a while" reminder. It fires after 5 silent turns, and the text comes from the server.

---

## My data, for the pile

- **atis history:** `705` (Oct 2), `52f` (Oct 3 to 14), `c85` (Oct 20 to Nov 2), blank since Nov 3. `c85` was in the **claude-desktop** slot, so the jq in OP's post misses it.
- **Flag assignments as of Nov 9:** mossy_gray v1 `true`, keen_pudding v1 `58720256` (exactly 56 MiB), radiant_fox v1 `true`, clever_orbit v0 `false`, typed_trinket `0` with no assignment. That's the opposite arm from another commenter in the thread on 3 of those 5.

---

## How the map was made

- For each cached flag, I found the real code call sites in the binary (`R("tengu_x", default)` style reads, plus constants that hold the key) and skipped places where the key only shows up in the compiled string table.
- Then I read the code around each one: the default value, any `CLAUDE_CODE_*` env override, and whatever text the branch turns on.
- These flags are configured by Anthropic server-side, through GrowthBook and the per-model slot. Env vars can override only some of them (185 of the 487).

| column | meaning |
|---|---|
| key | flag name |
| value | **my account's** served value on 2026-11-09, not the default. Yours will differ. |
| confidence | `E` = the code says it directly. `I` = inferred from context, so treat it as a guess. |
| what_it_does | one-line read of the code |

*The table holds flag names, served values, and descriptions only.*
