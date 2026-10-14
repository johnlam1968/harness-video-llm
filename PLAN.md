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

### 2.2 Shape

Follows Pi's own guidance in `packages/coding-agent/docs/custom-provider.md`, which says to
prefer a complete provider for integrations owning more than static metadata. Register:

- a provider id (e.g. `minimax-cn-video`) with `apiKey: "$MINIMAX_CN_API_KEY"`
- models declaring `input: ["text","image","video"]`, `contextWindow: 1000000`,
  `maxTokens: 512000`, zero cost
- `reasoningEfforts`-equivalent mapping for `output_config.effort` ∈
  `low|medium|high|xhigh|max` — **no `off`**, because thinking is forced on for M3.1 and
  `disabled` returns 400
- a custom `streamSimple` that owns the HTTP and emits real `type:"video"` blocks

### 2.3 Non-negotiable contracts from Pi's docs

Pi's `custom-provider.md` states that omitting these "makes the provider behave differently
from Pi's built-in providers":

- call `options.onPayload` before sending, honour a replacement payload
- call `options.onResponse` after receiving, before consuming the body
- await `options.onProviderStreamEvent?.(providerEvent, model)` per parsed event
- pass through the abort signal and provider-scoped environment

Plus the mandated test matrix: ordinary and empty text · tool calls and results · **image
input and image tool results** · usage and cost accounting · abort · context overflow ·
malformed/partial streams · Unicode boundaries · cross-provider handoff · auth refresh and
cancellation.

### 2.4 Limits

Configurable, with MiniMax's documented values as defaults — video 50 MiB, request body
64 MiB, image 10 MiB. Enforce **before** provider I/O so an oversized video fails with a
stable error code, never an opaque upstream 400.

### 2.5 Known risk

MiniMax streaming quirks. The community patch `pi-minimax-m3-caching-fix` exists because M3
emitted duplicated thinking blocks and alternated `reasoning_content`/`reasoning` across
chunks. Research now shows **one of those two bugs is fixed upstream** (the driver takes the
first non-empty reasoning field and never switches, and `ensureThinkingBlock` guards against
a second block), while the inline `<think>` duplication is **still unaddressed upstream**.
Budget for hitting a variant of this during implementation.

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