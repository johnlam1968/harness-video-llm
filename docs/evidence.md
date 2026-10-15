# Evidence

Every claim in this repo that could be wrong is backed by something reproducible. Measured
2026-10-13 unless stated otherwise.

---

## 1. MiniMax accepts video and reads it correctly

Live probe against `https://api.minimaxi.com/anthropic/v1/messages`, credential resolved from
the running DSH process (value never printed).

### Fixture

A 4-second 320×240 clip with a distinctive **visual** and a distinctive **audio** track, so
a correct answer cannot be produced by guessing:

```bash
ffmpeg -hide_banner -loglevel error -y \
  -f lavfi -i "color=c=blue:s=320x240:d=4,drawtext=text='FRAME %{n}':fontcolor=yellow:fontsize=28:x=20:y=100" \
  -f lavfi -i "sine=frequency=880:duration=4" \
  -c:v libx264 -pix_fmt yuv420p -c:a aac -shortest probe.mp4
```

56 KB, `h264` video + `aac` 44.1 kHz mono audio.

### Results

| probe | result |
|---|---|
| text control | **200** — `"OK"` |
| image (png) | **200** — `"Red"` |
| **video (mp4)** | **200** — *"The background is bright blue, and the text reads **"FRAME 85"** in yellow."* |
| audio-only (m4a) | **400** — `unsupported content type 'audio' (2013)` |
| text control on `api.minimax.io` | **401** — `invalid api key` |

Three findings, all of which matter:

1. **Video works and is genuinely decoded.** The model reported the background colour and a
   specific frame number from the rendered text. It was not describing metadata.
2. **There is no audio content block.** Error 2013. A video *container* may carry an audio
   track, but audio cannot be submitted on its own.
3. **Host matters.** This credential is CN-only. `.com` returns 200, `.io` returns 401. The
   published quick-start snippet `ANTHROPIC_BASE_URL=https://api.minimax.io/anthropic`
   documents the international surface and does **not** apply to this key.

### Specs, from vendor docs and the canonical catalog

| field | value | source |
|---|---|---|
| context | `1_000_000` | `platform.minimax.io/docs/api-reference/text-anthropic-api` |
| max output | `512_000` | `sst/models.dev` `models/minimax/MiniMax-M3.1-Flash-Preview.toml` |
| modalities in | `text, image, video` | same TOML |
| thinking | forced on; `disabled` → 400 | doc "Thinking Control" |
| depth | `output_config.effort` ∈ `low\|medium\|high\|xhigh\|max`, default `max`; `none` → 400 | doc + TOML |
| cost | `0/0/0/0` | same TOML |
| availability | "M Plan and MiniMax Code for now" | doc banner |

---

## 2. Pi rejects video in `models.json` — runtime-verified

The published JSON schema is unambiguous:

```json
"input": { "type": "array", "items": { "enum": ["text", "image"] } }
```

in **both** `providers.*.models[].input` and `modelOverrides.*.input`.

Advisory schemas lie, so this was tested against the real binary with an isolated `HOME`:

```bash
mkdir -p /tmp/modtest/.pi/agent && cp ~/.pi/agent/models.json /tmp/modtest/.pi/agent/
HOME=/tmp/modtest pi --list-models
```

| mutation | outcome |
|---|---|
| baseline, `input: ["text","image"]` | 467 rows, no error |
| `input: ["text","video"]` | **`Invalid models.json schema: - providers.volcengine.models.0.input.1: must be equal to one of the allowed values`** |
| `input: ["text","banana"]` | same error — confirms the validator actually fires |
| `"video"` in a `modelOverrides` entry | **no error at all** |

Two incidental findings:

- **`modelOverrides` is not validated** the way `providers` is. Bad values there are silently
  ignored. Probably unintended upstream, but it makes that surface a silent no-op.
- **The existing override for M3.1 is dead.** `~/.pi/agent/models.json` already contains
  `modelOverrides["minimax-cn/MiniMax-M3.1-Flash-Preview"]`, yet the model never appears in
  `--list-models`. Pi ignores overrides for unknown IDs, and the model is not in the built-in
  catalog. **There is currently no M3.1 model in Pi at all.**

---

## 3. Source-level walls

```ts
// packages/ai/src/providers/model-schema.ts:5
export const MODEL_INPUT_MODALITIES = ["text", "image"] as const;
```

```ts
// packages/ai/src/api/anthropic-messages.ts  (current main)
// media_type is passed through; `type` is a literal
media_type: block.mimeType as "image/jpeg" | "image/png" | "image/gif" | "image/webp",
```

```ts
// deepseek-harness packages/llm/llm/src/types.ts:231 — DSH keeps its own copy
export interface ModelModalityMap {
  text: 'text'
  image: 'image'
}
```

```ts
// deepseek-harness packages/llm/llm-pi-ai/src/context.ts:71
): string | (TextContent | ImageContent)[] {
```

DSH does **not** inherit Pi's vocabulary. A Pi release adding video would supply one of four
layers; DSH's three remain its own.

---

## 4. Which labs actually take video

From `sst/models.dev` canonical records (`models/<lab>/<model>.toml`, `[modalities] input`),
552 records across 69 labs. Cross-checked against vendor docs where it mattered.

| lab | model | `input` | video |
|---|---|---|---|
| Google | Gemini 4-Argon | `text, image, video, audio, pdf` | ✅ + audio |
| Google | Gemini 3-Pro / 3.1-Pro / 3-Flash | `text, image, video, audio, pdf` | ✅ + audio |
| Xiaomi | MiMo-V2.6-Pro | `text, image, audio, video` | ✅ + audio |
| MiniMax | M3, M3.1-Flash-Preview | `text, image, video` | ✅ |
| Zhipu | GLM-4.6V | `text, image, video` | ✅ |
| Moonshot | Kimi K3 | `text, image, video` | ✅ |
| ByteDance | Seed 2.1 Pro | `text, image, video` | ✅ |
| Anthropic | Claude Opus 5.5 / Sonnet 5.5 | `text, image, pdf` | ❌ |
| OpenAI | GPT-6.1-Sol / 5.6-Terra / 4o | `text, image, pdf` | ❌ |
| Zhipu | GLM-5.3 | `text` | ❌ |
| DeepSeek | V4-Pro | `text` | ❌ (no multimodal) |

**Attribution correction (2026-10-15).** An earlier draft of the harness comparison in the
deepseek-harness Discussion #9647 post attributed the placeholder string

```
[Media omitted from provider request because the selected model does not support video input.]
```

to OpenClaw. **That string is ZCode's**, from `zai-org/zcode`
`apps/zcode-cli/packages/contracts/src/model/media-policy.ts`:

```ts
return `${placeholder}\n[Media omitted from provider request because the selected model does not support ${unsupportedKind}.]`
```

OpenClaw's own visible-omission string is different — verified in `openclaw/openclaw`
`packages/ai/src/provider-transcript-transform.ts:12`:

```ts
const VIDEO_OMISSION = "(video omitted: provider does not support video input)"
```

Both harnesses omit **visibly**, which was the substance of the claim and remains true; only the
attribution was wrong. The ZCode placeholder is the better one to cite, because it is templated
over a typed reason — `UnsupportedModelInputMediaKind = "image input" | "PDF input" | "video input"`.

Confirmed directly against vendor docs:

- **Anthropic** — `platform.claude.com/docs/en/build-with-claude/vision` documents only
  `image` content blocks. No video type anywhere on the page.
- **OpenAI** — the API docs navigation has an "Images and vision" guide and no video guide.
- **Gemini** — video is tokenised per second (a 60-second clip ≈ 6,000 tokens), which is why
  frame-sampling tools exist as a cost workaround.

**The sharp point:** video input is present at every Chinese frontier lab checked, and absent
at both Western labs. This is not "catch up with everyone" — it is a capability the Western
frontier has not adopted.

---

## 5. `pi-minimax-m3-caching-fix` is half obsolete

The upstream commit its README cites, `b85b91c9` ("route MiniMax-M3 to openai-completions for
passive caching", 2026-06-10), is **not merged**:

```
compare/b85b91c9...main  =>  status=diverged  ahead_by=2419  behind_by=1
```

`skipThinkingBlock` returns **zero hits** across all of pi's source.

But the behaviour was fixed differently, in `packages/ai/src/api/openai-completions.ts`:

```js
// Use the first non-empty reasoning field to avoid duplication
for (const field of ["reasoning_content", "reasoning", "reasoning_text"]) {
  if (typeof deltaFields[field] === "string" && deltaFields[field].length > 0) {
    foundReasoningField = field;
    break;                    // ← first non-empty wins; never alternates
  }
}
const block = ensureThinkingBlock(thinkingSignature);
```

and `ensureThinkingBlock` guards with `if (!thinkingBlock)`.

| bug the extension fixes | upstream status |
|---|---|
| alternates `reasoning_content`/`reasoning` → orphan thinking block | **fixed** |
| inline `<think>…` duplicated into visible text | **still open** — zero `<think>` handling in pi src except `llama-cpp-classify.ts` |

The extension has shrunk from two jobs to one. The remaining job is a vendor-agnostic
streaming defect: content carrying inline reasoning tags *while also* streaming
`reasoning_content`. Worth its own upstream issue.

---

## 6. Existing ecosystem (so we do not rebuild it)

| project | what it does | relevance |
|---|---|---|
| [`dsh-video-lens`](https://www.npmjs.com/package/dsh-video-lens) ★114, L5 | `video_probe` / `video_analyze` / `video_ask`; ffmpeg scene sampling → OpenAI-compatible VLM | frame sampling, not native video; discards audio |
| [`pi-web-access`](https://www.npmjs.com/package/pi-web-access) v0.40.0 | YouTube + local video understanding via Gemini | proves the tool surface, not native input |
| [`@getpipher/vision`](https://www.npmjs.com/package/@getpipher/vision) v0.5.3 | delegates only when the active model is text-only; native passthrough otherwise | the capability-aware routing shape |
| [`pi2dsh`](https://www.npmjs.com/package/pi2dsh) 0.25.2 | runs unmodified Pi extensions as native DSH plugins | delivery path to DSH |

`pi2dsh`'s Level-1 verified list includes `pi-provider-alibaba` — "Alibaba Token Plan (CN) …
live catalog, cold-start dynamic model, complete tool loop and restart" — proving custom
provider extensions run a real tool loop inside DSH.

None of these delivers native video to a chat model. They are all bridges and samplers.

---

## 7. Upstream tracker

| repo | item | state |
|---|---|---|
| `earendil-works/pi` | [#8886](https://github.com/earendil-works/pi/issues/8886) Add first-class video input support | closed `not_planned` **9 seconds after opening**, label `no-action` |
| `earendil-works/pi` | [#7326](https://github.com/earendil-works/pi/issues/7326) fix(ai): preserve MiniMax M3 video input capability | closed **7 seconds** after opening, 2026-07-30, unmerged |
| `earendil-works/pi` | [#3200](https://github.com/earendil-works/pi/issues/3200) Support video/audio content in prompt command | **open** |
| `deepseek-harness` | [#4725](https://github.com/deepseek-ai/deepseek-harness/discussions/4725) support pdf/video passthrough | open, unanswered since 2026-08-27 |

On DSH: its `CONTRIBUTING.md` states *"We are sorry that we cannot accept external pull
requests at the moment."* The sanctioned routes are Discussions and plugins.