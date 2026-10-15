# MiniMax Video-Input Capability — Research Report (as of 2026-10-15)

Legend: **[V]** = verified against a primary source (vendor doc / OpenAPI spec / model card / repo source). **[I]** = my inference.

## 1. Declared input modalities — MiniMax-M3 and MiniMax-M3.1-Flash-Preview

**[V]** From MiniMax's Anthropic-SDK reference:

> "`MiniMax-M3.1-Flash-Preview` and `MiniMax-M3` support text, image, video, tool use, tool result, and thinking blocks. The M2.7, M2.5, M2.1, and M2 series support text and tool-call content blocks only; they do not support image or video input."

> `type="image"` — "M3.1-Flash-Preview / M3 only. Image input via URL or base64. Supports JPEG, PNG, GIF, WEBP"
> `type="video"` — "M3.1-Flash-Preview / M3 only. Video input via URL, base64, or `mm_file://{file_id}`. Supports MP4, AVI, MOV, MKV"

**Neither audio nor PDF is declared for these two models.** (models.dev's Vercel entry for M3 lists `pdf`, but that is a Vercel-side gateway artifact, not a MiniMax declaration.)

- **Context window: 1,000,000** for both (doc's supported-models table).
- **Max output:** not stated in MiniMax's docs. models.dev records `output: 512000`. minimax-code's own catalog records `limit: {context: 512000, output: 128000}` with `contextWindowOptions: [512000, 1000000]`. Discrepant — treat 512k as the catalog value, 128k as the CLI default.
- **Thinking/reasoning:** `thinking` = `adaptive` | `disabled`.
  - M3: **off by default**; `adaptive` turns it on; `disabled` keeps it off.
  - M3.1-Flash-Preview: **always thinks**; `{"type":"disabled"}` returns `400` ("requires adaptive thinking … (2013)").
  - `output_config.effort` — **M3.1-Flash-Preview only**: `low | medium | high | xhigh | max`, default `max`; `none` → 400.
  - M2.x: thinking cannot be disabled. Ignored params: `top_k`, `stop_sequences`, `mcp_servers`, `context_management`, `container`. `top_p` default 0.95 (M3/M3.1) vs 0.9 (M2.x); `temperature` [0,2], recommended 1. `service_tier: standard | priority`.
  - Note the HF model card describes a third mode, `enabled`, that the API docs do not list.
- **Pricing [V]:** MiniMax-M3 pay-as-you-go — ≤512k input: **$0.30/M in, $1.20/M out, $0.06/M cache read**; >512k input: **$0.60 / $2.40 / $0.12**. `service_tier: "priority"` = **1.5×**. **M3.1-Flash-Preview has no pay-as-you-go price** — "available only through M Plan and MiniMax Code for now" (models.dev lists it at $0 under `minimax-coding-plan` / `minimax-cn-coding-plan`).
- **models.dev:** `minimax :: MiniMax-M3` → input `[text, image, video]`, output `[text]`, context 1,000,000, output 512,000, $0.30/$1.20/$0.06 with a >512k tier doubling. **Caveat:** modality lists vary per provider entry — nano-gpt, siliconflow, wandb, huggingface, togetherai, cortecs, merge-gateway, hyper and iteracompute list **text+image only**. Video is a first-party capability, not universally available on third-party hosts.

## 2. Video on the Anthropic-compatible endpoint — confirmed

**[V]** From MiniMax's own OpenAPI spec (`openapi-chat-anthropic.json`), the block you proposed is correct:

```json
{"type":"video","source":{"type":"base64","media_type":"video/mp4","data":"<base64>"}}
```

`source.type` enum is `["base64","url"]`; `media_type` is "Required for base64 input, for example image/png or video/mp4"; `url` also accepts `mm_file://{file_id}`.

**Formats (quoted from `MediaSource.description`):** MP4 → `video/mp4`; AVI → `video/avi` or `video/x-msvideo`; MOV → "URL input: set the object storage Content-Type to video/quicktime; base64 input: use video/mov, i.e. `data:video/mov;base64,<BASE64_ENCODING>`"; MKV → `video/x-matroska`.

**Limits (quoted):** "URL or Base64 input: video file ≤ 50 MB, image file ≤ 10 MB, request body ≤ 64 MB." / "Files API upload: referenced as `mm_file://{file_id}`, with a single video up to 512 MB."

**Frame sampling is server-side and client-tunable** — `MediaSource.fps`, quoted:

> "Video frame sampling rate. Defaults to 1. Range [0.2, 5].
> - Higher values: more sensitive to scene changes, but higher token cost and slower response.
> - Lower values: lower token cost and faster response, but less sensitive to scene changes."

Also `max_long_side_pixel` (integer ≥1): "Longest-side pixel limit for the image or video frames."

**Not found in public sources:** any maximum **duration** cap, and **any statement about whether the audio track is used or ignored**. Treat both as undocumented.

## 3. MiniMax coding products

### (a) Open source — yes, with native VIDEO support

**[V]** **`github.com/MiniMax-AI/minimax-code`** — "An open-source coding agent for your terminal, powered by MiniMax." **License MIT**, **TypeScript** (25.3 MB of 28.5 MB; also C#, Python, JS, Shell), 2,029 ★ / 255 forks, created 2026-06-01, last push 2026-10-15. Binary `mcode`; homepage `agent.minimax.io/download`.

Video is a **first-class native modality**, not an ffmpeg/frame-sampling path:
- `packages/agent-tools/src/shared/multimodal-attachments.ts` — `SUPPORTED_NATIVE_VIDEO_MIME_TYPES` = `video/mp4`, `video/x-msvideo`, `video/quicktime`, `video/x-matroska`, `video/mkv`.
- `packages/agent-tools/src/shared/read-video.ts` — the `read` tool inlines raw video bytes as pi-ai `{type:'image', data:<base64>, mimeType:'video/*'}`. Its own header comment: "The runtime-level provider video patchers then rewrite serialized provider image blocks with `video/*` media into `type:'video'` at the provider payload boundary."
- `packages/config/src/minimax-model-catalog.ts` — caps for both M3 and M3.1-Flash-Preview: `max_video_bytes_inline: 52_428_800` (50 MiB), `max_image_bytes_inline: 10_485_760`, `max_request_body_bytes: 67_108_864`, `max_attachments_count: 4`, `files_api_upload_endpoint: "/v1/files/upload"`. These match MiniMax's published limits exactly.
- **No ffmpeg/ffprobe anywhere** in `packages/` or `docs/` (only incidental mentions in a demo README about encoding a GIF). **No frame-sampling code** (no fps/frameCount logic) — sampling is entirely the server's job.
- Fallback for non-video models: `videos_understand` (`matrix_videos_understand` → `/matrix/api/v1/mcp/videos_understand`), "Send up to five local videos per call".

Other MiniMax repos: `MiniMax-AI/cli` (TypeScript, mmx-cli), `MiniMax-Coding-Plan-MCP`, `MiniMax-Code-MiniApps`, `MiniMax-Code-Plugins`, `OpenAgentCore` (Go), `Mini-Agent` (Python).

### (b) Hosted product

**[V]** Docs describe desktop/web/mobile apps and say "Drag in reference files or select a local project folder" and "upload reference files as needed" — but **never name video explicitly**.

**[I]** From `packages/local-runtime/src/api/hosted-agent-attachments.ts` (`attachmentKind`) and `messages/input.ts` (`renderAttachmentReminderEntry`), hosted attachments are classified `image | video | text | file` by MIME and inlined when `capabilities.support_video === true` — i.e. the hosted path is **native inline base64, not frame sampling**, same code path as the CLI. Not documented by the vendor, so treat as inference.

## 4. Public video-input evidence and evals

**[V]** MiniMax's own HF `eval_results/minimax-m3.yaml`, extracted from its model-card benchmark figure:

| Benchmark | Score |
| :- | :- |
| **Video-MME (w/ subtitles)** | **85.4** |
| MMMU-Pro | 78.1 |
| SWE-bench Verified | 80.5 |
| SWE-bench Pro | 59.0 |
| Claw-Eval | 74.5 |

Caveat: those numbers were transcribed from an **image** (`figures/benchmark.jpeg`), not a machine-readable table, and the YAML itself notes variant ambiguity. **Video-MMMU, MLVU and LongVideoBench: not found in public sources.** Technical report: arXiv:2606.13392 (video eval tables inside not verified here).

**[V]** The Anthropic Messages OpenAPI ships a **"Video Understanding"** worked example (`MiniMax-M3.1-Flash-Preview`, `thinking: adaptive`, `output_config.effort: max`, `type:"video"` with a public `.mp4` URL). The M3 model card states "M3 undergoes mixed-modality training from the very first step, enabling deeper semantic fusion across text, image, and video" and is tagged `image-text-to-text` + `video`.

---

## URLs actually fetched

- `https://platform.minimax.io/docs/api-reference/text-anthropic-api.md` — modalities per model; video/image block support; 1M context; `thinking` behavior table; `output_config.effort`; 50MB/10MB/64MB limits; M3.1-Flash-Preview M-Plan-only note.
- `https://platform.minimax.io/docs/api-reference/text/api/openapi-chat-anthropic.json` — **the decisive source for Q2**: `RequestContentBlock` (type enum incl. `video`), full `MediaSource` schema (`fps` 0.2–5 default 1, `max_long_side_pixel`, format table, size limits), "Video Understanding" example.
- `https://platform.minimax.io/docs/llms.txt` — doc index; located the OpenAPI spec paths.
- `https://platform.minimax.io/docs/pricing/overview.md` — M3 standard/priority pricing and the >512k tier; absence of an M3.1-Flash-Preview PAYG price.
- `https://platform.minimax.io/docs/m-plan/minimax-code.md` — hosted MiniMax Code surfaces and "upload reference files" wording (Q3b).
- `https://models.dev/api.json` — catalog entries for `minimax :: MiniMax-M3` and `minimax-coding-plan :: MiniMax-M3.1-Flash-Preview`; per-provider modality variance.
- `https://api.github.com/repos/MiniMax-AI/minimax-code`, `.../languages`, `https://api.github.com/orgs/MiniMax-AI/repos` — license (MIT), language mix, stars/forks, dates, org inventory.
- `https://github.com/MiniMax-AI/minimax-code` — repo landing page (returned mostly nav chrome; metadata came from the API instead).
- `https://huggingface.co/MiniMaxAI/MiniMax-M3/raw/main/README.md` — model card; native-multimodality claim; `video` tag; `enabled/adaptive/disabled` thinking modes.
- `https://huggingface.co/api/resolve-cache/models/MiniMaxAI/MiniMax-M3/.../.eval_results/minimax-m3.yaml` — **Video-MME 85.4** and the other benchmark numbers (Q4).
- Local clone `github.com/MiniMax-AI/minimax-code` @ `main` — file-level evidence for Q3a (`multimodal-attachments.ts`, `read-video.ts`, `minimax-model-catalog.ts`, `video-mime.ts`, `outbound-message-normalizer.ts`, `videos-understand.ts`, `hosted-agent-attachments.ts`).
