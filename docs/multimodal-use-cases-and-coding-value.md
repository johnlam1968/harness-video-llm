# Multimodal use cases, and what multimodal input is actually worth for coding

**Date:** 2026-10-15 · Method: vendor docs and OpenAPI (fetched), model card, published
benchmarks, local source read of `~/minimax-code` (**upstream `main` @ `9a93560`, pulled
2026-10-15**) and `~/mmc` (same commit, read-only), DSH source at `d743267388`.
Legend: **[V]** verified against a primary source · **[I]** inference.

This note answers three questions that were open: what a video-capable LLM is *for*, whether
coding actually benefits, and which harnesses already ship the plumbing. The capability
measurements live in [`evidence.md`](evidence.md); the harness-by-harness state lives in
`/home/john/research_reports/research/harness_study/minimax-code/37-video-input-landscape-across-harnesses.md`.

---

## 1. What a video-capable chat model is for

### 1.1 MiniMax-M3 / M3.1-Flash-Preview — the declared surface

**[V]** From `platform.minimax.io/docs/api-reference/text-anthropic-api`:

> "`MiniMax-M3.1-Flash-Preview` and `MiniMax-M3` support text, image, video, tool use, tool
> result, and thinking blocks. The M2.7, M2.5, M2.1, and M2 series support text and tool-call
> content blocks only; they do not support image or video input."

| property | M3 | M3.1-Flash-Preview |
|---|---|---|
| input modalities | text, image, video | text, image, video (**no audio, no PDF**) |
| context window | 1,000,000 **[V]** | 1,000,000 **[V]** |
| thinking | **off** by default; `{"type":"adaptive"}` enables | **always on** — `{"type":"disabled"}` → **400** (2013) **[V]** |
| effort control | `budget_tokens` | `output_config.effort` = low\|medium\|high\|xhigh\|max, default `max`; `none` → 400 **[V]** |
| price (≤512k in) | $0.30/M in · $1.20/M out · $0.06/M cache read **[V]** | **no pay-as-you-go price** — "available only through M Plan and MiniMax Code for now" **[V]** |
| price (>512k in) | $0.60 / $2.40 / $0.12; `service_tier:"priority"` = 1.5× **[V]** | — |

Three discrepancies worth carrying, all recorded rather than smoothed over:

1. **Max output is contradicted across three sources.** MiniMax documents none; `models.dev`
   says 512,000; `minimax-code`'s own catalog says `output: 128000` with
   `contextWindowOptions: [512000, 1000000]`. **[V]** for the conflict, unresolved for the truth.
2. **A third thinking mode exists only on the model card.** The HuggingFace card documents an
   `enabled` mode that the API reference does not list. **[V]**
3. **Video is first-party, not universal.** The `models.dev` rows for nano-gpt, siliconflow,
   wandb, huggingface, togetherai and others list `text, image` only. **[V]** So an
   OpenRouter-style route to the same model id may silently not take video — a second instance
   of the failure mode this whole work is about.

### 1.2 The wire block is confirmed by MiniMax's own OpenAPI

**[V]** `platform.minimax.io/docs/api-reference/text/api/openapi-chat-anthropic.json`:
`RequestContentBlock.type` enumerates `video`, and `MediaSource.type` is `["base64","url"]`.

```json
{"type":"video","source":{"type":"base64","media_type":"video/mp4","data":"<base64>"}}
```

| property | value | source |
|---|---|---|
| formats | MP4 → `video/mp4` · AVI → `video/x-msvideo` · MOV → `video/quicktime` (URL) or `video/mov` (base64) · MKV → `video/x-matroska` | **[V]** OpenAPI `MediaSource` |
| inline limits | **video ≤ 50 MB · image ≤ 10 MB · request body ≤ 64 MB** | **[V]** OpenAPI |
| Files API | `mm_file://{file_id}`, single video up to **512 MB** | **[V]** OpenAPI |
| **frame sampling** | `MediaSource.fps`, **range [0.2, 5], default 1** — "Higher values: more sensitive to scene changes, but higher token cost and slower response." | **[V]** OpenAPI |
| frame resolution | `max_long_side_pixel` (integer ≥ 1) | **[V]** OpenAPI |

**Not in public sources, so not claimed here:** any maximum duration cap, and whether the
audio track is consumed or ignored.

`fps` is the underused lever: it is the one client-side dial for the cost/latency/fidelity
trade-off, and no harness in our survey exposes it (see §3.5).

**Qualified 2026-10-15.** `fps` buys **cost and fidelity, not motion understanding.** MotionBlind
(arXiv:2609.09528) is explicit: *"Neither more frames nor smarter frame selection closes the gap,
because these change which frames are seen, not whether motion is read."* So this dial is worth
exposing as a **cost** control, and should not be sold as the fix for temporal reasoning. See
[`media-understanding-as-a-capability-seam.md`](media-understanding-as-a-capability-seam.md) §8.2.

### 1.3 Where the use cases actually are

Grouped by what the evidence supports, strongest first.

| use case | evidence | read |
|---|---|---|
| **Screen-recorded bug reproduction** | **ViScratch** ([arXiv:2509.11065](https://arxiv.org/abs/2509.11065)): gameplay video + block code beats an LLM baseline **without** video *and* beats human testers on bug identification and AST-verified repair, with no project description available | **[V]** the strongest case for video in coding — the video carries the reproduction |
| **Pixel-faithful front-end from a walkthrough** | **IWR-Bench** ([arXiv:2509.24709](https://arxiv.org/abs/2509.24709), ICLR 2026): 113 tasks, 1,001 actions, 28 LVLMs. Best **36.35%** overall — but **visual fidelity 64.25% vs functional correctness 24.39%** | **[V]** video copies *appearance*; it does not infer *behaviour* |
| **Cross-language visual bug-fixing** | **SWE-bench Multimodal** ([arXiv:2410.03859](https://arxiv.org/abs/2410.03859)): 617 instances, 17 JS libraries, ≥1 image each; best **12% vs 6%** | **[V]** but this is **image**, not video |
| **Screenshot-to-code at scale** | **MM-IssueLoc** ([arXiv:2607.15205](https://arxiv.org/abs/2607.15205)): 652 issue-PR pairs, 23 languages, paired text-only vs with-image; best agent 38.96 file Acc@5 | **[V]** the gap is real, and the fix is a still image |
| **Layout/visual-element recall** | **Design2Code** (NAACL 2025, [aclanthology](https://aclanthology.org/2025.naacl-long.199/)): 484 real webpages | **[V]** |
| **Long-video QA / summarisation** | **Video-MME (w/ subtitles) = 85.4** from MiniMax's own eval YAML | **[V] vendor-published, weakly pinned** — transcribed from a benchmark *image* in the model card, and the YAML itself flags variant ambiguity |
| Web/UI comprehension (general) | MMMU-Pro 78.1, Claw-Eval 74.5, SWE-bench Verified 80.5, SWE-bench Pro 59.0 (same source, same caveat) | **[V] vendor-published** |

Video-MMMU, MLVU and LongVideoBench: **not found in public sources** for this model.

### 1.4 The decisive number for planning

**[V]** Google documents **258 tokens per second of video**
([tokens](https://ai.google.dev/gemini-api/docs/tokens)), and states its own comparison
baseline is "1 FPS for static processing". Therefore:

> **258 tok/s ≡ 1 frame/s × 258 tok/frame.** A 30-second recording ≈ 30 frames ≈ **7,740
> tokens ≈ 15 full screenshots.**

Video input to a chat model is a **packaging convenience around frame sampling**, not
additional temporal signal. You can extract those frames yourself and send them as images —
which is exactly what Claude computer use and Playwright do (Anthropic documents ~1,000–1,800
input tokens **per screenshot**, a still-image loop by design, guidance "keep the last 3, prune
every 25 turns").

This does not make video useless. It relocates the claim: MiniMax's server-side sampler means
a client can hand over a file instead of running ffmpeg, and MiniMax's own eval puts Video-MME
at 85.4. But **"video in, better code out" is not a claim the benchmarks support**, and the
comparison a buyer would actually want — video vs. three well-chosen screenshots — **has not
been measured in any study found**.

### 1.5 The one measured win is doing *less* vision

**[V]** OSWorld-MCP ([arXiv:2608.03327](https://arxiv.org/abs/2608.03327)), 309 tasks: dropping
redundant screenshots and halving image history cut input tokens by ~⅓ at small accuracy cost;
the retrained agent scored **37.8% vs 33.0% at 53% of input cost**. Tool use: **+4.0pp** for a
reasoning model, **−5.9pp** for a non-reasoning one.

So the replicated, credible result in coding-agent vision is **context pruning**, not more video.

### 1.6 Claims with no evidence behind them

- **"Video-to-app from a walkthrough"** (v0 / Lovable / Figma Make / Builder.io) — no public
  benchmark found. IWR-Bench is the nearest serious measurement and it reports 24% functional
  correctness.
- **Vendor root-cause analysis** (Sentry Seer, Datadog Bits AI) — no independent measurement.
- **Google's "agentic video" savings** (tokens −88.0%, accuracy 87.5→88.5; cost "up to −66%")
  — vendor-selected benchmarks, no cost table behind the 66%, no independent replication.
  Notably **the largest saving yields the smallest accuracy gain**.
- **PDF/spec ingestion and whiteboard reading for coding** — no controlled measurement found.
  Do not assume.

### 1.7 What to prioritise, given the above

1. **Stills with aggressive pruning** — the only replicated win.
2. **Auto-attach a screenshot to every bug or failure** — cheap, and MM-IssueLoc proves the gap.
3. **Video only for genuinely temporal failures** — flaky tests, races, animation, gesture
   sequences — and then send **pre-extracted keyframes plus the file only when the harness
   can carry it**, because that is the case (ViScratch) the evidence actually supports.

---

## 2. Does coding benefit from multi-modality? — short answer

**Yes for images, narrowly and repeatedly. For video, yes in one well-defined case (reproducing
a runtime failure), and no in the case the marketing implies (learning a codebase or writing
features faster).**

The trap is that the *packaging* win and the *capability* win get confused. Video lets a client
skip ffmpeg. It does not, on current evidence, let a coding agent do things three screenshots
could not.

---

## 3. The harness landscape — what already works

### 3.1 Four implementations, three mechanisms

| Harness | Native video | Mechanism | Capability gate |
|---|---|---|---|
| **MiniMax-Code** (`@minimax-ai/code`) **[V, source]** | ✅ | video rides pi-ai's `ImageContent` as `{type:'image', mimeType:'video/*'}`, then a **payload-boundary rewriter** mutates it to `{type:'video'}` | per-model `support_video`, `max_video_bytes_inline`, `max_request_body_bytes` |
| **ZCode** (`zai-org/zcode`) **[V, source]** | ✅ | first-class `type:"video"` block; two wire encodings: anthropic `{type:'video',source:{…}}`, openai `{type:'video_url',video_url:{url:'data:…'}}` | per-model `ModelInputFormat{supportsImage,supportsPdf,supportsVideo}` + **typed** omission reason |
| **OpenClaw** **[V, source]** | ✅ for google / moonshot / qwen | per-vendor `extensions/<vendor>/media-understanding-provider.ts`, raw bytes inline | per-vendor capability lists; **18 providers shipped** |
| **Kimi Code CLI** **[V, docs]** | ✅ | `Ctrl-V`/`Alt-V` paste gated on `image_in`/`video_in` model flags; 20 MB/session | per-model booleans |
| **Pi** | ❌ | — | `MODEL_INPUT_MODALITIES = ["text","image"]`, `models.json`-validated |
| **DSH** | ❌ | — | `ModelModalityMap {text,image}`; **only** `llm-deepseek` and `llm-pi-ai` ship adapters |

### 3.2 MiniMax-Code in detail — the mechanism to copy

**[V]** `~/minimax-code`, **pulled to upstream `main` @ `9a93560` on 2026-10-15** (previously
pinned at v0.4.12 / `17b8ded`, 155 commits behind). Upstream `main` is also cloned read-only at
`~/mmc`.

`packages/agent-tools/src/shared/read-video.ts` states the design in its own header comment:

> "Video is not part of pi's built-in detection because pi does not own the host's multimodal
> capability declaration. […] The runtime-level provider video patchers then rewrite serialized
> provider image blocks with `video/*` media into `type:'video'` at the provider payload
> boundary."

The rewriter is now **readable in source** at
`packages/local-runtime-v2/src/service/turn-system/agent-host/assembly/local-turn-payload-transform.ts:475-499`:

```ts
function patchVideoBlock(value: unknown): boolean {
  if (!isRecord(value)) return false
  let mutated = false
  if (value.type === 'image' && isRecord(value.source)) {
    const mediaType = value.source.media_type
    if (typeof mediaType === 'string' && mediaType.startsWith('video/')) {
      value.type = 'video'; mutated = true
    }
  }
  if (Array.isArray(value.content)) {
    mutated = value.content.reduce((c, n) => patchVideoBlock(n) || c, mutated)
  }
  return mutated
}
```

Four properties — the fourth being the one that differs from every other harness:

- **Recurse into nested `content`** — tool results carry media too.
- **Return whether anything changed**, so the caller can report it.
- **Gated on `model.api === 'anthropic-messages'` and `capabilities.support_video === true`** —
  video is an Anthropic-protocol feature here; an OpenAI-completions route gets no rewriting.
- **Omission is decided at ingress, not at the wire.** `patchVideoPayload(payload, enabled)` with
  `enabled === false` returns `false` and does nothing. The visible statement is made earlier,
  by `read-video.ts` / `multimodal-attachments.ts`, as a text-only tool result carrying a typed
  reason — `unsupported_model`, `max_video_bytes_inline`, `max_request_body_bytes`,
  `max_attachments_count`, `unsupported_mime` — with `details.media.inline: false`.

ZCode and OpenClaw both decide at the wire; MiniMax-Code decides before the bytes are read,
which is cheaper and simpler. **For a DSH plugin the ingress decision is also the only one
reachable**, because the payload boundary is not extensible on an existing route (see §3.5).

Verified properties: **zero ffmpeg/ffprobe anywhere** in `packages/`; no client-side frame
sampling at all (raw bytes base64'd; sampling is the server's job); caps match MiniMax's
published limits exactly — `max_video_bytes_inline` 52,428,800 (50 MiB),
`max_image_bytes_inline` 10,485,760, `max_request_body_bytes` 67,108,864 (64 MiB),
`max_attachments_count` 4. MIME allowlist: `video/mp4`, `video/x-msvideo`, `video/quicktime`,
`video/x-matroska`, `video/mkv`.

**New since the previous read (2026-10-15):**

- **`support_files_api: true` + `files_api_upload_endpoint: "/v1/files/upload"`** — the
  `mm_file://{file_id}` path (up to 512 MB per the vendor OpenAPI) is **wired**, with
  `messages-file-api-patcher.ts` beside the video transform in the same assembly step. That is
  the answer to "how do you carry a video too large to inline", already implemented.
- A first-class `type: 'video'` now exists in the host's own vocabulary
  (`agent-core/src/tools/types.ts:47`), with video-aware token counters, and the image-dimension
  probe **deliberately skipping** video blocks: *"only the telemetry skips blocks projected from
  `video`, which keep their `video/*` mimeType and are unparseable by design."*
- Per-model thinking variants are explicit: M3 `switchable` with `none-thinking` / `thinking`
  (`disabled` / `adaptive`); M3.1 `effortOptions: [default, low, medium, high, xhigh, max]`.
- The caps were **relocated** into a shared `MINIMAX_M3_FILE_API_CAPABILITIES` constant, not
  changed — the numbers are identical.
- It still never sets `fps`, so sampling stays at the server default; `image-detection.ts` does
  read `max_long_side_pixel` back off the block for token accounting.

Fallback when the model has no native video: the tool `videos_understand`
(`/matrix/api/v1/mcp/videos_understand`, up to five videos per call), selected only when
`support_video !== true`.

This is the **same mechanism independently arrived at** in `PLAN.md` §2.3 — the `onPayload`
payload rewrite — now corroborated by readable source rather than inferred from a minified bundle.

### 3.3 OpenClaw — the best architecture, but **not** for MiniMax

**[V]** `main` carries **18** `media-understanding-provider.ts` files (`google`, `minimax`,
`moonshot`, `qwen`, `zai`, `anthropic`, `codex`, `openai`, `openrouter`, `ollama`, `groq`,
`mistral`, `deepgram`, `deepinfra`, `elevenlabs`, `senseaudio`, `opencode`, `opencode-go`).

This corrects the earlier note in two ways:

| vendor | declared capabilities |
|---|---|
| `google` (Gemini API) | image + audio + **video** |
| `moonshot` (Kimi) | image + **video** |
| `qwen` | image + **video** |
| **`minimax` / `minimax-portal`** | **image only**, routed to `MiniMax-VL-01` |

So OpenClaw *has* solved native video — but for **Moonshot/Kimi, Google and Qwen**, and it
routes MiniMax image understanding to a dedicated VL model rather than using M3's own vision.
The design lessons stand regardless: route-owned qualification, bounded `maxBytes`
(10/20/50 MB image/audio/video), and a **bounded model-visible marker** for every skipped
attachment with a five-marker budget and a typed disposition per candidate.

**Moonshot is the closest analogue to MiniMax** — same Anthropic-compatible protocol family —
so `extensions/moonshot/media-understanding-provider.ts` is the better template to read than
the MiniMax one.

### 3.4 ZCode — now settled, and the cleanest gate

The prior note recorded ZCode video as **"not established"** and marked the grep as pending. It
is done: `apps/zcode-cli/packages/adapters/tests/video-wire.test.ts` asserts both encodings,
and `packages/contracts/src/model/media-policy.ts` carries the capability gate:

```ts
export function getUnsupportedModelInputMediaKind(block, inputFormat) {
  if (block.type === "image" && !inputFormat.supportsImage) return "image input"
  if (isProviderVisiblePdfModelInputBlock(block) && !inputFormat.supportsPdf) return "PDF input"
  if (isProviderVisibleVideoModelInputBlock(block) && !inputFormat.supportsVideo) return "video input"
  return undefined
}
```

A first-class block, a per-model boolean triple, and a **typed** omission reason that feeds a
visible placeholder. This is the cleanest of the four designs and the one to imitate.

### 3.5 DSH — no hidden adapter, and what already exists to build on

**Verified at `d743267388` and in the installed runtime bundle:**

- Exactly **two** LLM adapter families ship: `llm-deepseek` and `llm-pi-ai`. Nothing else
  calls `registerAdapter`.
- Across `packages/llm` and `packages/attachment`, the **only** `video` hit is a test
  asserting the block assembler *rejects* an unhandled `blockType: 'video'`. There is no video
  reader, no video modality, nothing to discover.
- pi-ai's own `Model.input` is typed `("text" | "image")[]`, so DSH's `PiAiModality` is
  structurally `text | image` and a hand-declared `llm-pi-ai` route **cannot declare video**
  even with an updated catalog.

But three pieces of the answer already exist in-tree, and they are the right scaffolding:

1. **`fileHandleText()`** — the durable attachment + visible-handle-text pattern, already
   applied to every route: *"[File "x" (N bytes, sha256:…): verbatim read-only copy saved at
   …]"*. A video a route cannot carry degrades to exactly this shape, and it is already the
   only representation a provider ever receives for a file.
2. **The `llm/stream` waterfall** — the reachable seam for a visible-omission policy on any
   route, via re-dispatch (see the reply draft for the mechanics).
3. **The v4 session format's `plugin:<type>` namespacing** — a plugin-added block is a
   plugin-namespaced tag, which is how a video block survives the durable log.

### 3.6 The DSH plugin ecosystem: access, not native input

From the dsh.so registry index (21,067 plugins; CC BY 4.0, `generatedAt` 2026-10-15T01:15:17Z):

| plugin | what it does | native? |
|---|---|---|
| `dsh-video-lens` ★114, **L5**, audited | ffmpeg scene sampling → OpenAI-compatible VLM; ≤12 frames, audio discarded | sampler |
| `dsh-video-understand` ★11, L5 | Bilibili/local video → AVIS layer → summary + Q&A | sampler |
| `dsh-mimo-agent-tools` ★6, L4 | Xiaomi MiMo vision/video/audio/asr/tts as **tools** | tool bridge |
| `dsh-mmx-bridge` ★10, L5 | MiniMax multimodal via an external `mmx` binary | tool bridge |
| `dsh-directorx` ★34, L5 | configurable vision/video/audio model **tools** | tool bridge |

**No plugin registers a native video content block or a video-capable provider adapter.** The
ecosystem has solved *access to video understanding* through tools, repeatedly and competently.
It has not solved *video in the conversation*, which is exactly the gap a plugin would fill.

---

## 4. The plumbing question, answered three ways

> "pi is not interested; OpenClaw may have solved it; MiniMax-Code is open source; DSH is
> flexible — maybe adapters already exist?"

| route | answer |
|---|---|
| **Pi upstream** | Not viable. `#8886` closed `not_planned` 14 minutes after opening; `#7326` closed unmerged; `#3200` still open. Pi team has said no. |
| **Pi via extension** | Still viable and now better specified: `onPayload?: (payload: unknown, model: TModel) => unknown \| undefined \| Promise<…>` (`pi-ai/dist/types.d.ts:73`, "Return undefined to keep the payload unchanged") is exactly the seam `minimax-code` uses. `[V]` for 0.85.1; the 1.2.0 copy was not present on this host to check. |
| **OpenClaw** | Solved it, for google/moonshot/qwen — not MiniMax. Copy the *architecture* (route-owned qualification, bounded markers, typed dispositions), not the vendor list. |
| **MiniMax-Code** | Solved it, open source (MIT), and it is pi-based — so its rewriter is a direct port target for the Pi extension. |
| **DSH** | No hidden adapter exists. But the reachable shape is settled: a **new route with its own adapter** (the registry forbids taking over an existing one), a **`plugin:video` durable block** projected through the existing `fileHandleText` pattern when the route cannot carry it, and the **visible-omission policy** on `llm/stream`. |

**The recommendation does not change** — build the Pi extension as the portable artifact — but
the reason is now different and stronger: it is no longer "the only path that depends on nobody
else's roadmap", it is "the same one rewrite already runs in production at MiniMax, and pi2dsh
carries it to DSH".

---

## 5. What would make this worth building

Given §1, the honest framing for the plugin is not "native video input for coding agents". It
is:

> **Make a video attachment survive the harness at all, visibly.** Today a DSH user attaching a
> video gets a confident, invented answer with a clean transcript. After this plugin they get
> either a real answer or a visible statement that the video was not sent. That is true
> regardless of whether video ever beats three screenshots for most coding tasks — and it is
> the only property here that is both broken today and fixable without anyone's roadmap.

The `fps` dial ([0.2, 5], default 1 **[V]**) is the feature none of the four harnesses exposes.
If this gets built, that is where the cost control lives.
