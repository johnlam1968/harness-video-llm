# Research notes

Architecture findings gathered while scoping native video input. Kept separate from
[`evidence.md`](evidence.md), which holds the reproducible measurements; this file is the
reading that informed the plan, including the dead ends.

---

## How Pi's model layer is put together

Read on `main` at `d743267388` era, and cross-checked against the installed 1.2.0 build.

| concern | location |
|---|---|
| modality vocabulary | `packages/ai/src/providers/model-schema.ts` |
| content union (`TextContent`/`ThinkingContent`/`ImageContent`/`ToolCall`) | `packages/ai/src/types.ts` |
| Anthropic wire protocol | `packages/ai/src/api/anthropic-messages.ts` |
| OpenAI-compatible wire protocol | `packages/ai/src/api/openai-completions.ts` |
| catalog generation | `packages/ai/scripts/generate-models.ts` → `providers/data/*.json` |
| catalog validation | `packages/ai/scripts/model-data.ts` |
| `models.json` loading + validation | `packages/coding-agent/src/core/model-config.ts` |
| extension authoring | `packages/coding-agent/docs/extensions.md`, `docs/custom-provider.md`, `docs/packages.md` |
| checklist for core-level providers | `.pi/skills/add-llm-provider.md` |

Pi reads its catalog from **`https://models.dev/models.json`** at generation time. That is the
single upstream source for which models exist and what they cost — which is why an absent entry
is a hard block rather than a config issue.

## The two dead ends worth remembering

**Pi's `MODEL_INPUT_MODALITIES` is not the only constraint**, and fixing it alone would not
have been enough. Even with video in the vocabulary, the Anthropic driver serialises
`type` as a literal, so a video block would still not reach the wire. Both have to move
together. That is why the extension approach — owning `streamSimple` outright — sidesteps the
problem rather than half-solving it.

**`models.json` cannot create a model.** `modelOverrides` only adjusts metadata for models Pi
already knows about; unknown ids are dropped without a warning. Worth knowing because it is a
natural first thing to try, and it fails silently.

## DSH shares the catalog but not the vocabulary

DSH bundles `@earendil-works/pi-ai` for the model data, but defines its **own** modality type
and its own content projection:

```ts
// deepseek-harness packages/llm/llm/src/types.ts:231
export interface ModelModalityMap { text: 'text'; image: 'image' }

// deepseek-harness packages/llm/llm-pi-ai/src/context.ts:71
): string | (TextContent | ImageContent)[]
```

So a Pi release adding video would supply **one of four** layers. The remaining three are
DeepSeek's. This is why a Pi-side fix does not automatically reach DSH, and why the plan
targets Pi and uses `pi2dsh` for DSH delivery rather than assuming inheritance.

Version position, for context:

| | installed in this deployment | newest known |
|---|---|---|
| `@earendil-works/pi-ai` | 0.85.1 (DSH's bundle) | **1.2.0** (2026-10-12) |
| `deepseek-harness` | 0.1.7-rc.2 | 0.2.1-alpha.2 |
| `pi2dsh` | 0.25.2 | 0.26.0 |
| `pi2dsh` targets | Pi 0.84.1 · DSH 0.1.1-rc.2 / 0.1.5-rc.* | — |

## The extension seam

`packages/coding-agent/docs/custom-provider.md` is the authority, and it routes decisions
explicitly:

| requirement | use |
|---|---|
| Add models behind a supported API | `models.json` |
| Discover models dynamically | a provider with `refreshModels` |
| Add a `/login` flow | native or legacy OAuth config |
| **Implement an unsupported wire protocol** | **a provider with `stream` or `streamSimple`** |

It also prefers *"a complete provider for new integrations that own more than static endpoint
and model metadata"* — which is this case.

Contracts that are easy to miss and will cause subtle divergence if skipped:

- `options.onPayload` before sending, honour a replacement payload it returns
- `options.onResponse` after receiving, before consuming the body
- `await options.onProviderStreamEvent?.(providerEvent, model)` per parsed event

The doc's own wording: *"Omitting them makes the provider behave differently from Pi's
built-in providers."*

## Why an extension rather than a patch

`dsh-llm-pi-ai` and `@deepseek-ai/dsh-llm` are first-party DeepSeek packages, and
`CONTRIBUTING.md` says plainly: *"We are sorry that we cannot accept external pull requests at
the moment."* Their enumerated alternatives are Discussions, plugins under the `dsh-plugin`
topic, guides, and Q&A.

So the realistic routes are: (a) an upstream Discussion, and (b) a plugin or an extension. The
Pi extension route additionally gets native execution in Pi and portability to DSH via
`pi2dsh`, which a DSH plugin does not. That is the deciding factor for the plan.

## Prior art read, not just listed

- **`dsh-video-lens`** — read its source layout (`frames.js`, `vlm.js`, `asr.js`, `probe.js`).
  Scene-change detection with `ffmpeg scdet`, midpoint fallback, ASR strictly additive, ≤12
  frames. Deliberately a **sampler**, not a native path. Its default vision model is
  `Qwen/Qwen3-VL-8B-Instruct` via an OpenAI-compatible endpoint.
- **`dsh-mmx-bridge`** — shells out to an external `mmx` binary; model ids are `MiniMax-H3`,
  `MiniMax-Hailuo-2.3`. Video **generation**, plus a `describe` action via a separate VLM
  endpoint. Zero references to M3.1 or the chat route.
- **`@getpipher/vision`** — delegates only when the active model is text-only, otherwise
  passes through natively. The routing shape we want, done for images.
- **`pi-provider-alibaba`** — pi2dsh's Level-1 verified list includes it as a custom provider
  with a live catalog and complete tool loop in DSH. The strongest evidence that the
  extension route reaches DSH.

## Open questions

1. Does `pi2dsh` work on Pi 1.2.0 / DSH 0.1.7-rc.2? Both outside its declared window.
   `npx pi2dsh inspect <pkg>@<version>` exists for exactly this.
2. Does an extension declaring `input: ["text","image","video"]` get a video affordance in
   Pi's TUI composer, or is that gated on the core vocabulary too? Probably the latter, and
   it would mean video arrives via tool input rather than paste.
3. Is the first-cut package named for MiniMax or for the capability? See `PLAN.md` §7.