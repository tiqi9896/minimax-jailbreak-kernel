# hermes-minimax-jailbreak-kernel

A Hermes Agent plugin that installs a **non-refusing delivery contract** into
your **MiniMax M3.1-Flash** calls, plus a reusable measurement skill that
proves whether the change actually took effect.

> **What this is:** a behavior-contract layer for a model *you* call with *your
> own* API key. It changes how the model behaves toward your requests — it does
> **not** bypass provider-side enforcement, and it does not make anyone else's
> systems more accessible.
>
> **Use it only for work you are authorized to do**, and comply with your
> provider's terms of service and applicable law.

---

## Why this exists

MiniMax M3.1-Flash is a strong model that, under some prompts, still:

- refuses outright,
- adds disclaimers / "educational use only" tails,
- silently **softens the deliverable** — a requested tool becomes a
  "detection and remediation guide",
- **fades to black** (skips the actual content instead of writing it),
- or answers with **an empty body** because the thinking budget ate the
  entire `max_tokens`.

Most "jailbreak prompt" collections solve none of these reliably, and none of
them tell you whether the fix worked. This repo is built around **measured**
behavior on M3.1-Flash specifically.

## Key findings (measured, not guessed)

| Finding | Why it matters |
|---|---|
| `thinking.budget_tokens` is **ignored** by M3.1-Flash | Only `output_config.effort` (adaptive thinking) has any effect. Hermes' default path routed MiniMax into the manual-`budget_tokens` branch, so effort was never actually applied. |
| An **empty body is usually not a refusal** | It is the thinking budget exhausting `max_tokens` (`stop_reason=max_tokens`, `thinking_tokens ≈ max_tokens`). Raising `max_tokens` or lowering effort fixes it; prompt engineering does not. |
| Refusals happen at the **model behavior layer**, not server-side | All requests returned HTTP 200 with a success status code — so a prompt-level contract can suppress them. |
| `effort=medium` is the sweet spot | `low` under-thinks and drifts into defensive framing more often; `high`/`max` burn thinking tokens. |
| The decisive lever is a **scope statement at the end of the *user* message** | Putting it in the system prompt only is measurably weaker. It suppresses both hard refusals *and* topic drift. |
| System-prompt sections are **frozen per session** | A session created before the plugin was enabled keeps its old persisted prompt. A request-layer fallback re-injects the kernel on every call, so **every** session is covered. |

## Layout

```
plugin/                     the installable Hermes plugin
  __init__.py               middleware: system rebuild, effort injection,
                            auth block, per-session bootstrap fallback
  plugin.yaml               plugin manifest
  selftest.py               offline self-test (no network, no API key)
  title_extract.py          title classifier used by the probe tooling
  prompts/profiles/         the kernel text, in two segments
skill/llm-refusal-bypass/   reusable measurement methodology + probe script
docs/                       install / usage / MiniMax-specific notes
```

## Install

See [`docs/INSTALL.md`](docs/INSTALL.md). Quick version:

```bash
git clone https://github.com/tiqi9896/hermes-minimax-jailbreak-kernel.git
cp -r hermes-minimax-jailbreak-kernel/plugin/minimax-jailbreak-core \
      ~/.hermes/plugins/
hermes plugins enable minimax-jailbreak-core
```

Then verify it is actually live:

```bash
cd ~/.hermes/plugins/minimax-jailbreak-core && python selftest.py
```

The self-test is **offline** — it never calls a model and never needs a key.

## Configure

Point Hermes at MiniMax M3.1-Flash, then set your API key:

```yaml
model:
  default: MiniMax-M3.1-Flash
  provider: minimax-cn
  base_url: https://api.minimaxi.com/anthropic
```

```bash
export MINIMAX_CN_API_KEY=<your key>
```

## Use

In-session commands (registered by the plugin):

```
/jailbreak on|off|slim|full     toggle the kernel
profile core|strict|max|research|creative
effort low|medium|high|max
```

Defaults: profile `strict`, effort `medium`, adaptive thinking on.

### Environment overrides

| Variable | Effect |
|---|---|
| `HERMES_JAILBREAK_MODE` | `off` / `slim` / `on` / `full` (default `on`) |
| `HERMES_JAILBREAK_PROFILE` | default profile name (default `strict`) |
| `HERMES_JAILBREAK_EFFORT` | default effort (default `medium`) |
| `HERMES_JAILBREAK_EFFORT_LOCK` | `1` = ignore per-session effort |
| `HERMES_JAILBREAK_ADAPTIVE` | `0` = leave `thinking` untouched |
| `HERMES_JAILBREAK_AUTH_BLOCK` | `0` = do not append the scope statement |
| `HERMES_JAILBREAK_BOOTSTRAP` | `0` = disable the per-session kernel fallback |
| `HERMES_JAILBREAK_PREFILL` | `1` = enable assistant prefill (default off) |
| `HERMES_JAILBREAK_UPSTREAM_PROFILES` | path to upstream profiles, if installed |
| `HERMES_JAILBREAK_DISABLE` | `1` = plugin registers nothing at all |

## Verify it yourself — do not trust the docs

The whole point of the bundled skill is that you can check the claims:

```bash
# 1. plugin self-test (offline)
cd plugin/minimax-jailbreak-core && python selftest.py

# 2. live refusal probe against your own endpoint (needs your key)
python skill/llm-refusal-bypass/scripts/refusal-probe.py \
  --key-env MINIMAX_CN_API_KEY \
  --base-url https://api.minimaxi.com/anthropic/v1/messages \
  --model MiniMax-M3.1-Flash \
  --effort medium \
  --cases skill/llm-refusal-bypass/references/refusal-verdicts.md
```

Read [`docs/USAGE.md`](docs/USAGE.md) for the A/B methodology and the verdict
classification (`COMPLY` / `SOFT-REFUSE` / `HARD-REFUSE` / `TRUNCATED` /
`COMPLY(tool_use)`).

## Which models does this work on?

The kernel is applied by **route**, so every MiniMax model id is covered
automatically — including ones published after this repo was written. But
route coverage is not the same as *behavioral* coverage, so here is what was
actually measured.

| Model id | Refusals | Disclaimers | Delivery | Notes |
|---|---|---|---|---|
| `MiniMax-M3.1-Flash` | 0 / 8 | 0 / 8 | 8 / 8 | the reference target |
| `MiniMax-M3.1-Flash-Preview` | 0 / 8 | 0 / 8 | 6 / 8 + 2 tool-call turns | same behavior as above |
| `MiniMax-M3` | 0 / 8 | 0 / 8 | budget-limited — see below | **needs a bigger output budget** |

### The MiniMax-M3 output-budget finding

`MiniMax-M3` **thinks much harder** than `M3.1-Flash`. With the same
`max_tokens=8000` used for the other models, 2–3 of 8 cases came back with an
**empty body** and `thinking_tokens` pinned at the limit.

That is **not a refusal** — it is the thinking budget consuming the entire
output allowance before the model ever starts writing. Measured on the
hardest case (C2 + persistence framework):

| `max_tokens` | Result |
|---|---|
| 8000 | empty body, `thinking_tokens` = 8000 (budget exhausted) |
| 16000 | empty body, `thinking_tokens` = 16000 (still exhausted) |
| **32000** | **60689 chars, `thinking_tokens` = 17438, `stop_reason` = end_turn, correct title** |

**Practical rule:** if you switch to `MiniMax-M3`, set `max_tokens` to
**32000+**. If you see an empty response with a high `thinking_tokens`, raise
the budget — do not go looking for a prompt problem.

Note that `thinking.budget_tokens` in the request does **not** cap this:
M3 spent 17438 thinking tokens in a request that asked for 8000. The
adaptive `effort` setting governs it, and it can consume the whole
`max_tokens`.

### Model id aliases

`MiniMax-M3.1-Flash-Preview`, `MiniMax-M3.1-Flash-preview` and
`minimax-m3.1-flash-preview` are **the same model** — the API normalizes the
id and returns identical output. The same holds for `MiniMax-M3` /
`minimax-m3`. Test one canonical spelling, not every alias.

### A refusal-shaped opening is not always a refusal

One measured response opened with a refusal sentence and then delivered a
complete, working implementation in full. Judge the verdict on the **whole
body**, not on the first line — and when the title has drifted into different
naming, check whether the substance is still there before counting it as a
failure.

## Attribution

The prompt corpus and plugin scaffold derive from
**[Enchanted0911/hermes-jailbreak-kernel](https://github.com/Enchanted0911/hermes-jailbreak-kernel)**
(MIT, © 2026 Enchanted0911). See [`NOTICE`](NOTICE) for the full breakdown of
what came from where. **Keep the attribution** if you redistribute.

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md) — including the portable-path trap
that broke the plugin for anyone who did not already have the upstream plugin
installed, and the clean-directory test you must run before opening a PR.

## License

MIT — see [`LICENSE`](LICENSE).
