# Plan

Working plan for making native video input usable in [Pi](https://github.com/earendil-works/pi),
starting with MiniMax and generalizing to the other labs that offer it.

Last updated 2026-10-14. Status markers: ⬜ not started · 🟡 in progress · ✅ done · ⚠️ blocked

---

## 0. Shape of the problem

Three walls stand between a video file and a video-capable model in Pi. All three were
verified by reading source or running Pi, not inferred.

| # | wall | where | verified how |
|---|---|---|---|
| 1 | modality vocabulary is `{text, image}` | `packages/ai/src/providers/model-schema.ts` | `MODEL_INPUT_MODALITIES = ["text","image"]`; `models.json` runtime-validated, `"video"` rejected |
| 2 | Anthropic driver hardcodes `type:"image"` | `packages/ai/src/api/anthropic-messages.ts` | `media_type` passed through, `type` literal; `media_type` union is image MIME types only |
| 3 | the model is not in the generated catalog | `packages/ai/scripts/generate-models.ts` → `providers/data/*.json` | `MiniMax-M3.1-Flash-Preview` absent from `models/minimax/`; `modelOverrides` ignores unknown ids |

Wall 3 is broader than MiniMax: **DSH reads the same pi-ai catalog**, so both hosts lack the
model. That is why the work targets Pi and reaches DSH through the bridge.

**Scope of the walls (refined 2026-10-14).** Wall 1 binds `models.json` only; the
`registerProvider` path has no schema check. Wall 2's `media_type` is a TypeScript cast with
no runtime validation, so the video payload can ride an ordinary `ImageContent` all the way to
the request. Both are still real — but the only genuinely new code is the block-type rewrite
in §2.3. See §2.4 for the measurements.

### The one thing that does not need solving

`type: "audio"` does not exist. MiniMax rejects it with
`unsupported content type 'audio' (2013)`. A video file may carry an audio track and is
accepted as a video block, but there is no separate audio path, and none is planned here.
Audio-to-text is the separate `/v1/speech_to_text` endpoint.

---

## 1. Upstream issues ⬜ (drafted, not posted)

Two issues, filed separately because they are separable and have different audiences.

**Issue A — catalog gap.** `MiniMax-M3.1-Flash-Preview` missing from the generated catalog,
so it is unusable. Small, high-confidence, has precedent in #7326 which was closed unmerged.
Draft: [`docs/issues/pi-catalog-minimax-m3-1.md`](docs/issues/pi-catalog-minimax-m3-1.md)

**Issue B — video modality.** The broader ask, and the one with the wider audience. Every
Chinese lab frontier model we checked takes video; OpenAI and Anthropic take none. Draft:
[`docs/issues/pi-video-input-modality.md`](docs/issues/pi-video-input-modality.md)

**Ordering.** Post A and B together — A unblocks the model, B unblocks the capability.
If upstream engages, prefer their design over ours and treat the extension as a stopgap.

---

## 2. The extension ⬜

A Pi package providing native video input, installable with
`pi install npm:pi-minimax-video`, and portable to DSH through `pi2dsh`.

### 2.1 Scope

Capability-driven, not vendor-hardcoded: pass video through for **any** model that declares
the modality on a registered route, and refuse with a stable error otherwise. That keeps
the implementation reusable when other labs' routes are added in §4.

### 2.2 Shape — delegate, do not re-implement

**Corrected 2026-10-14.** An earlier draft of this section specified "a custom `streamSimple`
that owns the HTTP". That is the wrong shape and it is superseded.

Pi's `packages/coding-agent/docs/custom-provider.md` says the opposite, twice:

> Use one of Pi AI's API implementations whenever the provider protocol matches it.
> This is safer than copying a stream implementation because it preserves Pi's message
> conversion, tool handling, usage accounting, cancellation, and compatibility behavior.

and lists "Implement an unsupported wire protocol" as the *only* reason to reach for a custom
stream. MiniMax speaks Anthropic Messages, which Pi implements. So `streamSimple` delegates:

```ts
import { stream as anthropicStream } from "@earendil-works/pi-ai/..."; // packages/ai/src/api/anthropic-messages.ts
```

The whole extension reduces to one injected hook. `StreamOptions.onPayload` is typed
`(payload: unknown, model) => unknown | undefined | Promise<...>` — *use any replacement
payload it returns* — so we rewrite the outgoing body on its way out.

Owning the HTTP would mean re-implementing all six requirements of custom-provider.md
"Implement custom streaming", losing pi's tested tool/usage/cancel handling for no benefit.

### 2.3 What is actually new: one payload rewrite

Pi emits, for any `ImageContent` (line 168 of `anthropic-messages.ts`):

```ts
{ type: "image", source: { type: "base64", media_type: block.mimeType as "image/jpeg"|…, data: block.data } }
```

`mimeType` is a **cast, not a runtime check** — `ImageContent.mimeType` is a bare `string`
in `packages/ai/src/types.ts:433`. So a block carrying `mimeType: "video/mp4"` reaches the
payload hook already base64-encoded and correctly typed. The hook then rewrites the block
type. Lifted from ZCode's two pnpm patches (`@ai-sdk__anthropic@3.0.81.patch`,
`@ai-sdk__openai-compatible@2.0.60.patch`, 92 lines total):

```ts
// input : {"type":"image","source":{"type":"base64","media_type":"video/mp4","data":…}}
// output: {"type":"video", "source":{"type":"base64","media_type":"video/mp4","data":…}}
```

and for the OpenAI-completions route, `image_url` → `video_url` with a `data:` URL.

**This rewrite is mandatory, not an optimisation.** Measured 2026-10-14 against
`MiniMax-M3.1-Flash-Preview` with the 4 s fixture:

| wire shape | result |
|---|---|
| `type:"video"` + `media_type:"video/mp4"` | **200** — *"A blue screen rapidly displays incrementing yellow-and-white frame numbers from 01 to 85."* |
| `type:"image"` + `media_type:"video/mp4"` | **400** — `image data url media type "video/mp4" not supported (2013)` |
| `type:"video"`, no accompanying text block | 200 but `stop_reason:"max_tokens"`, empty text |

The third row matters: pi's `convertContentBlocks` already unshifts `"(see attached image)"`
when a message carries no text, so that path stays safe — but the placeholder wording is
wrong for video and should be corrected in the same pass.

### 2.4 What is *not* a wall: the modality enum

`MODEL_INPUT_MODALITIES = ["text","image"]` (`packages/ai/src/providers/model-schema.ts:5`)
looks like a hard stop. It is not, on this path. That array feeds a TypeBox schema used to
validate **`models.json` at load**. The extension registration path is a different function:

- `validateExtensionProvider` → `applyExtension` (`provider-composer.ts:341,529`) maps
  `config.models` through `extensionModelFromDefinition` with **no schema validation**
- `MODEL_INPUT_MODALITIES` is otherwise referenced **nowhere** in `packages/ai/src`
- `registerProvider` *is* called eagerly on registration ("validate eagerly so
  registration/reload reports structural errors immediately") — but that eagerness runs the
  compose pipeline, not a TypeBox check

So `input: ["text","image","video"]` in an extension is a TypeScript error and a working
runtime value. It is written with a cast and a comment saying why.

The single runtime consumer of the vocabulary is `downgradeUnsupportedImages`
(`api/transform-messages.ts:35`), which gates on `model.input.includes("image")` and replaces
unsupported blocks with a placeholder, for user messages and tool results alike. Declaring
`"video"` in `input` is exactly the flag that switch reads.

### 2.5 The capability-downgrade pattern already exists in Pi

`transform-messages.ts` is ZCode's design, already shipped, for images:

```ts
const NON_VISION_USER_IMAGE_PLACEHOLDER  = "(image omitted: model does not support images)";
const NON_VISION_TOOL_IMAGE_PLACEHOLDER  = "(tool image omitted: model does not support images)";
```

compared with ZCode's `createUnsupportedModelInputMediaText`:

```
"[Media omitted from provider request because the selected model does not support video input.]"
```

Same three properties: per-model capability gate, **visible** placeholder rather than a
silent drop, applied on both the user-message and tool-result paths. We do not design a
mechanism here; we extend an existing one to one more modality.

### 2.6 Non-negotiable contracts from Pi's docs

Because we delegate rather than re-implement, `custom-provider.md`'s four instrumentation
requirements are satisfied by forwarding `options` through to the built-in stream. They
become a test, not an implementation:

- call `options.onPayload` before sending, honour a replacement payload
- call `options.onResponse` after receiving, before consuming the body
- await `options.onProviderStreamEvent?.(providerEvent, model)` per parsed event
- pass through the abort signal and provider-scoped environment

Plus the mandated test matrix: ordinary and empty text · tool calls and results · **image
input and image tool results** · usage and cost accounting · abort · context overflow ·
malformed/partial streams · Unicode boundaries · cross-provider handoff · auth refresh and
cancellation.

### 2.7 Limits

Configurable, with MiniMax's documented values as defaults — video 50 MiB, request body
64 MiB, image 10 MiB. Enforce **before** provider I/O so an oversized video fails with a
stable error code, never an opaque upstream 400. Enforcing before provider I/O means sizing
in the `onPayload` hook, since that is the last point before the request leaves.

### 2.8 Known risk

MiniMax streaming quirks. The community patch `pi-minimax-m3-caching-fix` exists because M3
emitted duplicated thinking blocks and alternated `reasoning_content`/`reasoning` across
chunks. Research now shows **one of those two bugs is fixed upstream** (the driver takes the
first non-empty reasoning field and never switches, and `ensureThinkingBlock` guards against
a second block), while the inline `<think>` duplication is **still unaddressed upstream**.
Delegation makes this strictly less risky than the old plan: we inherit whatever the built-in
driver does rather than freezing a copy of it.

---

## 3. Publishing ⬜

- npm package with the `pi-package` keyword (required for gallery eligibility)
- either conventional `extensions/` discovery or an explicit `pi.extensions` manifest
- README documenting the limits above, so nobody mistakes a 200 response for proof that
  every frame was observed

---

## 4. Generalisation to the other video-native labs ⬜

Once one route works, the same `streamSimple` serves any anthropic-messages route that
declares video. Candidate routes, from the measured capability table in
[`docs/evidence.md`](docs/evidence.md):

| lab | models declaring video input | notes |
|---|---|---|
| MiniMax | M3, M3.1-Flash-Preview | first target; Anthropic-protocol |
| Zhipu | GLM-4.6V | video, no audio |
| Moonshot | Kimi K3 | video, no audio |
| ByteDance | Seed 2.1 Pro | video, no audio |
| Xiaomi | MiMo-V2.6-Pro | video **and** audio |
| Google | Gemini 4-Argon / 3.x Pro / 3.x Flash | video **and** audio; different wire protocol |

Gemini is a separate wire protocol (`google-generative-ai`), so it is its own adapter rather
than a configuration change — deliberately out of scope for the first cut.

---

## 5. Delivery to DSH via pi2dsh ⚠️

`pi2dsh` runs unmodified Pi extensions as native DSH plugins, so the extension above would
also serve DSH sessions. **But the versions do not line up**, and this is unverified:

| | pi2dsh targets | this deployment runs |
|---|---|---|
| Pi | 0.84.1 | **1.2.0** |
| DSH | 0.1.1-rc.2, 0.1.5-rc.1, 0.1.5-rc.2 | **0.1.7-rc.2** (clone at 0.2.1-alpha.2) |
| pi2dsh itself | — | installed 0.25.2, npm latest 0.26.0 |

Evidence that the path works in principle: pi2dsh's Level-1 verified list includes
`pi-provider-alibaba` — "Alibaba Token Plan (CN) … live catalog, cold-start dynamic model,
complete tool loop and restart" — which is a custom-provider extension running a real tool
loop inside DSH.

**Action before relying on it:** run `npx pi2dsh inspect <pkg>@<version>` for our package, and
read pi2dsh's `docs/posting-kit/support-matrix.md` to understand what each verification level
proves. Note pi2dsh is currently wired only into the `pi-cs-test` profile, not `docdrift`.

---

## 6. Success criteria

- [ ] `pi install npm:pi-minimax-video` exposes `MiniMax-M3.1-Flash-Preview` with video input
- [ ] a real video reaches the model as a `type:"video"` block — verifiable against the live
      API with the fixture in `docs/evidence.md`
- [ ] Pi's mandated test matrix passes
- [ ] oversized video fails with a stable code before provider I/O
- [ ] the extension's reasoning output is clean, with no duplicated or orphan thinking blocks
- [ ] DSH delivery confirmed via `pi2dsh inspect` and one real DSH session
- [ ] Issues A and B posted upstream

---

## 7. Open decisions

1. **Post the issues, or wait?** Posting under this account is public and awkward to retract.
2. **If upstream engages** — defer to their design and retire the extension? `dsh-design-qa`
   sets the precedent with its own tagline: *"Retires itself when DeepSeek ships vision."*
3. **Naming.** `pi-minimax-video` is accurate but narrow; the plan generalises. Decide
   whether to name for the first provider or for the capability.

### 7.1 Blocking — needed before code

4. **Provider id: shadow or sibling?**
   - *Sibling* — register `minimax-cn-video` with its own model list. Pi's catalog is
     untouched, `pi.unregisterProvider()` cleanly reverts, and the user's model choice is
     explicit. Cost: the user must switch models.
   - *Shadow* — `registerProvider("minimax-cn", { models })` to redefine the built-in
     provider's models in place. Costless to use, but it overrides a shipped provider and the
     revert path is less obvious.
   Recommendation: **sibling**, for reversibility. Worth confirming — this is a UX call, not
   a technical one.
5. **Ingress: how does a video get in?** Pi's `UserMessage.content` is
   `(TextContent | ImageContent)[]`, and `ImageContent.mimeType` is a free string, so a
   `read_video` tool or a `/video <path>` command emitting `{type:"image", mimeType:"video/mp4"}`
   is sufficient. The open question is whether to ship **only** a tool (model calls
   `read_video` like any other tool, so nothing changes for existing workflows) or **also** a
   command/drag-and-drop path (user attaches directly).
   Recommendation: **tool first**, command as a thin follow-up — the tool is the smaller
   surface and matches what ZCode does for `Read`.
6. **Licence and attribution.** ZCode is Apache-2.0 and the lifted payload rewrite is a
   derived work of it. Confirm we ship the `NOTICE` attribution and keep the lifted hunks
   marked as derived, rather than re-typing them from the spec.