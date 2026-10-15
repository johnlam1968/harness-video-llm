# harness-video-llm

**Making video input reachable in coding harnesses — research, evidence, and the port plan.**

Several frontier models accept **video** as input today: `MiniMax-M3` and
`MiniMax-M3.1-Flash-Preview`, Gemini 3/4, Kimi K3, GLM-4.6V, MiMo-V2.6-Pro, Seed 2.1 Pro. Measured
against the live MiniMax API on 2026-10-13, `MiniMax-M3.1-Flash-Preview` returns **HTTP 200 and
correctly reads video frames** — it named the background colour *and* an on-screen frame number
burned into a test clip.

Neither of the harnesses people actually code in can deliver it. Pi's model layer declares exactly
two input modalities, `text` and `image`; DeepSeek Harness keeps its own copy of that vocabulary
and, on top of it, silently drops a block it does not recognise. Four harnesses *do* ship native
video ([`docs/multimodal-use-cases-and-coding-value.md`](docs/multimodal-use-cases-and-coding-value.md))
and their implementations are read here as templates.

This repo holds the plan, the reproducible measurements, the upstream issues, and the harness
landscape — including an open discussion thread on
[deepseek-harness #9647](https://github.com/deepseek-ai/deepseek-harness/discussions/9647).

**No code yet.** This is a research and design repository; the extension is
[planned](PLAN.md), not written.

Started 2026-10-14.

---

## Status

| track | state |
|---|---|
| Research + evidence | ✅ complete — [`docs/evidence.md`](docs/evidence.md) |
| Harness landscape (13 harnesses + 4 with native video) | ✅ complete — [`docs/multimodal-use-cases-and-coding-value.md`](docs/multimodal-use-cases-and-coding-value.md) |
| DSH discussion landscape (4 video asks, declaration-surface gap) | ✅ complete — [`docs/dsh-video-discussion-landscape.md`](docs/dsh-video-discussion-landscape.md) |
| Coding-value evidence review | ✅ complete — in the multimodal note |
| Retention-mechanism review (what "keeping the understanding" means) | ✅ complete — [`docs/video-retention-mechanism.md`](docs/video-retention-mechanism.md) |
| Video product / harness landscape | ✅ complete — [`docs/video-product-harness-survey.md`](docs/video-product-harness-survey.md) |
| Reply to [deepseek-harness #9647](https://github.com/deepseek-ai/deepseek-harness/discussions/9647) | 📝 drafted, not posted — [`docs/dsh-discussion-9647-reply.md`](docs/dsh-discussion-9647-reply.md) |
| Upstream issues to `earendil-works/pi` | 📝 drafted, **awaiting review** — [`docs/issues/`](docs/issues/) |
| Extension implementation | ⬜ not started — [`PLAN.md`](PLAN.md) |
| `pi2dsh` delivery to DSH | ⚠️ version gap, unverified — [`PLAN.md` §5](PLAN.md#5-delivery-to-dsh-via-pi2dsh) |


## The walls, per harness

**Pi** — three independent ones, each verified:

1. **Modality vocabulary.** `MODEL_INPUT_MODALITIES = ["text", "image"]` in
   `packages/ai/src/providers/model-schema.ts`. Not a type-level nicety — `models.json` is
   runtime-validated and `input: ["text","video"]` is **rejected with a schema error**.
2. **The Anthropic driver.** `packages/ai/src/api/anthropic-messages.ts` hardcodes
   `type: "image"` and passes only `media_type` through, so a video can never reach the wire
   as `type:"video"` even if it were representable. Zero occurrences of `video` in 1,690 lines.
3. **The model is not in the catalog at all.** `MiniMax-M3.1-Flash-Preview` is absent from
   pi's generated catalog, so it cannot be used *at all* — not through `models.json`, and not
   through `modelOverrides`, because Pi silently ignores overrides for unknown IDs.

Upstream is closed at both ends: [earendil-works/pi#8886](https://github.com/earendil-works/pi/issues/8886)
closed `not_planned` **9 seconds** after opening, [#7326](https://github.com/earendil-works/pi/issues/7326)
**7 seconds**, and no release since v0.80.7 adds video. [`#3200`](https://github.com/earendil-works/pi/issues/3200)
is still open.

**DeepSeek Harness** — its own copy of the vocabulary plus a fail-open branch, and a second
wall underneath that the video threads have not connected:

1. `ModelModalityMap { text, image }` in `packages/llm/llm/src/types.ts`, and
   `userContent()` in `packages/llm/llm-pi-ai/src/context.ts` ends in a documented
   `default: break` — an unknown block is discarded and **the request still succeeds**.
2. Even with a `video` block, a route could not *declare* it accepts one: the resolution chain
   ends at `DEFAULT_INPUT = ['text']` and the admission gate rejects a model whose resolved
   `inputModalities` lacks `image`. Reported for images in
   [#5837](https://github.com/deepseek-ai/deepseek-harness/discussions/5837),
   [#5367](https://github.com/deepseek-ai/deepseek-harness/discussions/5367),
   [#6049](https://github.com/deepseek-ai/deepseek-harness/discussions/6049) — and a patch-ready
   proposal exists in [#8870](https://github.com/deepseek-ai/deepseek-harness/discussions/8870).

Video has been requested four times in that repo: [#4725](https://github.com/deepseek-ai/deepseek-harness/discussions/4725),
[#6050](https://github.com/deepseek-ai/deepseek-harness/discussions/6050),
[#9647](https://github.com/deepseek-ai/deepseek-harness/discussions/9647), and
[#9729](https://github.com/deepseek-ai/deepseek-harness/discussions/9729).

## What the extension path looks like

Pi's `registerProvider` takes an extension-supplied `streamSimple`, so an extension owns its
whole wire format and never touches Pi's shared content union:

```ts
pi.registerProvider("minimax-cn-video", {
  baseUrl: "https://api.minimaxi.com/anthropic",
  apiKey: "$MINIMAX_CN_API_KEY",
  api: "minimax-cn-video-api",
  models: [{ id: "MiniMax-M3.1-Flash-Preview", input: ["text", "image", "video"], ... }],
  streamSimple: streamVideoCapable,
});
```

That `input` array is a literal in a models list the extension owns — wall 1 does not apply,
and wall 3 disappears because the provider *creates* the model. The rewrite is the same one
[`minimax-code`](https://github.com/MiniMax-AI/minimax-code) ships: video rides the existing
`ImageContent` with a `video/*` MIME and a payload-boundary pass mutates it to `type:"video"`,
recursing into nested tool results.

The same artifact then runs unmodified in DSH through
[`pi2dsh`](https://www.npmjs.com/package/pi2dsh), which implements Pi's extension ABI on top
of DSH's native services.

## Documents

- [`PLAN.md`](PLAN.md) — the plan, phases, risks, and open decisions
- [`docs/evidence.md`](docs/evidence.md) — every measurement, with reproduction steps
- [`docs/multimodal-use-cases-and-coding-value.md`](docs/multimodal-use-cases-and-coding-value.md)
  — what video-capable models are for, what multimodal input is worth for coding on the
  evidence, and which harnesses already ship the plumbing
- [`docs/vendor-minimax-report.md`](docs/vendor-minimax-report.md) — MiniMax modalities, wire
  block and limits from the vendor's own OpenAPI, and the open-source `mcode` video path
- [`docs/research.md`](docs/research.md) — architecture findings across Pi and DSH
- [`docs/dsh-discussion-9647-reply.md`](docs/dsh-discussion-9647-reply.md) — draft reply to
  deepseek-harness Discussion #9647
- [`docs/dsh-video-discussion-landscape.md`](docs/dsh-video-discussion-landscape.md) — every
  video/modality thread in the DSH repo: four video asks, and the declaration-surface gap
  (#5837, #5367, #6049, #8870) that sits beneath the vocabulary
- [`docs/video-retention-mechanism.md`](docs/video-retention-mechanism.md) — **what "the model
  keeps the video's understanding" actually means**: it does not; the frames stay in the context
  window as tokens, and three consequences follow (budget, sampling policy, silent loss)
- [`docs/video-product-harness-survey.md`](docs/video-product-harness-survey.md) — the video
  *product* landscape, tiered by whether an LLM reasons over footage, and what harness runs
  underneath each
- [`docs/issues/`](docs/issues/) — two drafted upstream issues, not yet posted

## Four harnesses already ship native video

Verified in source, 2026-10-15 (see the linked note):

| Harness | Mechanism |
|---|---|
| **MiniMax-Code** (MIT, open source) | video rides pi-ai's `ImageContent` with a `video/*` MIME, then a **payload-boundary rewriter** mutates it to `type:'video'`; recurses into tool results; never drops silently |
| **ZCode** (Apache-2.0) | first-class `type:"video"` block, per-model `supportsVideo`, both `video` and `video_url` wire shapes |
| **OpenClaw** | per-vendor `media-understanding-provider.ts` — native video for google, moonshot, qwen (**minimax is image-only**) |
| **Kimi Code CLI** | paste gated on per-model `video_in` flags |

The MiniMax-Code rewriter is the closest precedent to this plan: it is pi-based and reduces the
whole problem to one `onPayload`-style rewrite, which is what [`PLAN.md`](PLAN.md) §2.3 targets.

## Licence and attribution

Research and design material, MIT — see [`LICENSE`](LICENSE).

No third-party code is vendored here. [`PLAN.md`](PLAN.md) §2.3 notes that the reference payload
rewrite is *derived* from [`zai-org/zcode`](https://github.com/zai-org/zcode) (Apache-2.0); if that
code is ever implemented here it will ship under Apache-2.0 with a `NOTICE`, per the repo's own
open decision 6. Code quoted from other projects is marked inline and attributed.

## Related

- [earendil-works/pi#8886](https://github.com/earendil-works/pi/issues/8886) — `Add first-class
  video input support`, closed `not_planned` **9 seconds** after opening
- [earendil-works/pi#7326](https://github.com/earendil-works/pi/issues/7326) — `fix(ai):
  preserve MiniMax M3 video input capability`, closed **7 seconds** after opening, unmerged
- [earendil-works/pi#3200](https://github.com/earendil-works/pi/issues/3200) — `Support
  video/audio content in prompt command`, **still open**
- [deepseek-harness #4725](https://github.com/deepseek-ai/deepseek-harness/discussions/4725) —
  `support pdf/video passthrough`
- [deepseek-harness #5837](https://github.com/deepseek-ai/deepseek-harness/discussions/5837) —
  the `llm-pi-ai` declaration-surface gap that video inherits
- [MiniMax-AI/minimax-code](https://github.com/MiniMax-AI/minimax-code) — MIT, the shipping
  implementation this plan ports
