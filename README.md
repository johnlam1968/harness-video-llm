# pi-video-llm

**Native video input for video-capable LLMs in [Pi](https://github.com/earendil-works/pi).**

Pi's model layer declares exactly two input modalities — `text` and `image`. Several
mainstream models accept **video** as input today, and none of them are reachable through Pi.
This repo holds the plan, the measured evidence, and the upstream issues needed to close that
gap — plus the extension that unblocks video now, for anyone who needs it before upstream does.

Started 2026-10-14.

---

## Status

| track | state |
|---|---|
| Research + evidence | ✅ complete — see [`docs/evidence.md`](docs/evidence.md) |
| Upstream issues to `earendil-works/pi` | 📝 drafted, **awaiting review** — [`docs/issues/`](docs/issues/) |
| `pi-minimax-video` extension | ⏳ not started — see [`PLAN.md`](PLAN.md) |
| `pi2dsh` delivery to DSH | ⚠️ version gap, unverified — see [`PLAN.md` §5](PLAN.md#5-delivery-to-dsh-via-pi2dsh) |

## Why

Measured against the live MiniMax API on 2026-10-13: `MiniMax-M3.1-Flash-Preview` returns
**HTTP 200 and correctly reads video frames** — it named the background colour *and* the
on-screen frame number of a test clip. Pi cannot deliver that video to it.

Three independent walls, each verified:

1. **Pi's modality vocabulary.** `MODEL_INPUT_MODALITIES = ["text", "image"]` in
   `packages/ai/src/providers/model-schema.ts`. Not a type-level nicety — `models.json` is
   runtime-validated and `input: ["text","video"]` is **rejected with a schema error**.
2. **Pi's Anthropic driver.** `packages/ai/src/api/anthropic-messages.ts` hardcodes
   `type: "image"` and passes only `media_type` through, so a video can never reach the wire
   as `type:"video"` even if it were representable.
3. **The model is not in the catalog at all.** `MiniMax-M3.1-Flash-Preview` is absent from
   pi's generated catalog, so it cannot be used *at all* — not through `models.json`, and not
   through `modelOverrides`, because Pi silently ignores overrides for unknown IDs.

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
and wall 3 disappears because the provider *creates* the model. The custom `streamSimple`
emits a real `{"type":"video","source":{...}}` block, so wall 2 does not apply either.

The same artifact then runs unmodified in DSH through
[`pi2dsh`](https://www.npmjs.com/package/pi2dsh), which implements Pi's extension ABI on top
of DSH's native services.

## Documents

- [`PLAN.md`](PLAN.md) — the plan, phases, risks, and open decisions
- [`docs/evidence.md`](docs/evidence.md) — every measurement, with reproduction steps
- [`docs/research.md`](docs/research.md) — architecture findings across Pi and DSH
- [`docs/issues/`](docs/issues/) — two drafted upstream issues, not yet posted

## Related

- [earendil-works/pi#7326](https://github.com/earendil-works/pi/issues/7326) — `fix(ai):
  preserve MiniMax M3 video input capability`, closed 2026-07-30, unmerged
- [earendil-works/pi#8886](https://github.com/earendil-works/pi/issues/8886) — `Add
  first-class video input support`, closed `not_planned` after 14 minutes
- [earendil-works/pi#3200](https://github.com/earendil-works/pi/issues/3200) — `Support
  video/audio content in prompt command`, **still open**
- [deepseek-harness Discussion #4725](https://github.com/deepseek-ai/deepseek-harness/discussions/4725)
  — `support pdf/video passthrough`, unanswered since 2026-08-27