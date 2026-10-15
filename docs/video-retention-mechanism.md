# Does an LLM "keep the understanding" of a video?

**Question.** The claim worth testing: *"the key point of video understanding for an LLM is that
instead of outputting a text description of a video, the LLM keeps the underlying 'understanding'
of the video, which could be used for subsequent conversation."*

**Verdict: half right, wrong mechanism.** The claim is right that a multi-turn video conversation
retains video-derived capability, and right that this is the point of video support. But "keeps the
underlying understanding" describes an implementation that **does not exist in any shipping
system**. An autoregressive transformer stores nothing between turns. What survives is the
*content of the request*: sampled frames encoded to video tokens that sit in the context window and
are re-attended on every later turn.

Better phrasing: **the video stays in the context window as tokens, and every turn re-reads them.**

Date: 2026-10-15 · method: vendor docs, model weights config, arXiv metadata, four vendor
caching docs, three harness bug reports, two benchmarks. Every claim below is cited; the two
places where MiniMax's own material corrects a natural assumption are called out.

---

## 1. MiniMax-M3's own artefacts settle it

The model card's *entire* multimodal claim is one sentence:

> "M3 undergoes mixed-modality training from the very first step, enabling deeper semantic fusion
> across text, image, and video."

It says nothing about tokenisation, caching, context handling with video, or multi-turn retention.
So the card cannot support the claim, and it does not refute it either. Two other artefacts do.

### 1.1 The weights config: video *is* context tokens

`config.json` on the Hub is the strongest evidence available:

| field | value | what it implies |
|---|---|---|
| `model_type` | `minimax_m3_vl` | vision-language, single sequence |
| `vision_config` | CLIP ViT, `patch_size 14`, **`rope_mode: "3d"`** | frames are encoded into a 3-D positional space |
| `img_token_compression_config` | `patch_merge`, `spatial_merge_size: 2`, `temporal_patch_size: 2` | frames are **merged into sequence tokens**, like text |
| `vision_segment_max_frames` | `4` | a per-segment frame cap (interpretation **unverified**) |
| `image_token_index` / `video_token_index` | **`200025` / `200026`** | **video occupies reserved positions in the LM vocabulary** |
| `max_position_embeddings` | `1048576` | the 1M context those tokens live in |

`video_token_index 200026` is the whole argument in one line. Video is not a side object the model
consults; it is **placeholder tokens inside the text sequence**, participating in the same
attention and the same position budget as text. There is no separate store in which an
"understanding" could live, because there is no second representation.

### 1.2 Sampling is the caller's parameter

MiniMax's API docs (`text-openai-api`): videos accept **MP4, AVI, MOV, MKV**; **`fps` defaults to 1
and accepts 0.2–5**; URL or base64 up to **50 MB**; larger via the Files API as
`mm_file://{file_id}`, **up to 512 MB**.

So the model never decides what it saw. The caller decides how many frames exist, and every later
turn re-reads *those frames*. Change `fps` between turns and you get a different "understanding"
of the same file — which is only possible if the understanding is the sampled sequence itself.

### 1.3 The handles expire, which is the failure mode

`file-management-upload` documents `video_understanding` files as *"retained for up to **7 days**"*.
A conversation that refers to `mm_file://abc` on day 8 gets a dead handle. Nothing was ever
"kept" on the model's side.

---

## 2. Caching is a compute optimisation, never a retention mechanism

This is the most common conflation, so it is worth being exact.

| vendor | explicit caching of video | detail |
|---|---|---|
| **Gemini** | **yes, explicitly** | flagship example caches `contents=[video_file]`, `ttl="300s"` (default 1 h), retrieved by `cache.name`; "It's not possible to retrieve or view cached content" |
| **Anthropic** | images yes; **video does not exist** | cacheable list includes "Images & Documents"; the Vision guide contains **zero** occurrences of "video" |
| **OpenAI** | video not listed | covers "text, images, documents, and supported audio" |
| **MiniMax** | **not documented** | "What Can Be Cached" lists tools / system / text / tool use / tool results — **images and video are absent** |

Two consequences. First, no vendor documents caching as a way of "keeping understanding" —
every one of them documents it as making a repeated request cheaper. Second, and against our
intuition: **ZCode attaches `cache_control` to its video block** (verified in its
`@ai-sdk__anthropic@3.0.81.patch`), while MiniMax's own cache documentation does not list video as
cacheable. So that `cache_control` is, on this endpoint, **unverified** — worth knowing before
we copy it.

Also relevant, measured by us on 2026-10-14
([`thinking-measurements.md`](thinking-measurements.md)): M3's Anthropic route caches
**server-side by content**, with `cache_creation_input_tokens` always `0` — the cache exists
whether or not a marker asks for it, and the marker is inert on M3. Retention is automatic;
durability is not.

---

## 3. How multi-turn actually "remembers"

All three majors store **request content**, not a model-side representation:

- **OpenAI** — "each text generation request is independent and **stateless**"; even with
  `previous_response_id`, "all previous input tokens for responses in the chain are billed as input
  tokens". The media is re-fed and re-encoded.
- **Anthropic** — stateless Messages API plus `cache_control` prefix reuse, which is a KV-cache /
  prefill optimisation.
- **Gemini** — `previous_interaction_id`, or a `cached_content` handle pointing at cached
  **tokens**.

No vendor describes any representation other than the request context. That is the answer.

---

## 4. The three consequences, each with evidence

The corrected framing is only useful if it changes decisions. It does, in three places.

### 4.1 Context budget is the real limit

`image-context-cascade` (harness middleware) documents the shape of the failure: every image lives
in the session as base64 **and is resent in full on every subsequent request**; in one real
session screenshots ate **86.3% of the context window**; a single request body hit **8.34 MB** and
**413**'d the session. Its own note is the sharpest thing in this document:

> "**Compaction can't save you** … A 413 happens the moment the request leaves your process."

Video is far heavier than images — Gemini's 258 tokens/s at 1 fps is ~15 full screenshots for a
30-second clip — so the same arithmetic applies with a much larger multiplier.

### 4.2 The sampling policy decides what survives

Not just what is seen, but what remains *recallable* later: at 1 fps a 30-second clip is 30
frames, and the model's memory of the video is bounded by which frames those were. `fps` is
therefore a *retention* parameter, not only a fidelity one — and, as noted, the one dial no
harness in our survey sets.

**But retention is not comprehension, and the two must not be conflated.** MotionBlind
(arXiv:2609.09528) reports that *"neither more frames nor smarter frame selection closes the gap,
because these change which frames are seen, not whether motion is read."* A video held at higher
`fps` across every turn is still a video the model may mis-read temporally. Raising `fps` is a
cost-and-fidelity decision; it is not a fix for the temporal-reasoning gap and should never be
presented as one.

### 4.3 Loss is silent, and it is a known class of bug

| case | what happens |
|---|---|
| `langchain-ai/deepagents` PR #3990 (merged 2026-06-15) | "the offload step relied on LangChain's default message rendering … which only preserved text content. **Non-text content blocks were silently dropped**" from saved history. Surfaced with images. |
| `NousResearch/hermes-agent` PR #44708 (merged 2026-06-12) | compaction summaries re-emitted stale `MEDIA:<path>` directives; now replaced with `[media attachment]` — summarisation mangling media references mid-conversation |
| Gemini Files | automatically deleted after **48 hours** |
| Anthropic cache | **5 minutes** unless re-used |
| MiniMax Files handle | **7 days** |

**And in our own target harness**, verified at `d743267388`:

```ts
// packages/compaction/compaction-basic/src/summarizer.ts:220
return blocks.filter((block): block is Extract<ContentBlock, { type: 'text' }> => block.type === 'text')

// packages/compaction/compaction-image-offload/src/image-offload.ts:31
if (block.type === 'image') {
```

The compaction seam **keeps only text blocks** and the offload plugin handles **only**
`block.type === 'image'`. A `video` block has neither path. So on DSH today, a video would be
dropped at the first compaction with no offload, no placeholder and no warning — which is a
different and worse failure than the one this repo is about, because the transcript still looks
complete.

This is the concrete form the "does it persist?" question takes for the work in this repo, and it
is a stronger argument for a video block than "users want video" ever was.

---

## 5. Is there empirical evidence on multi-turn video retention?

**Yes, and it is thin.**

- **MT-Video-Bench** (arXiv:2510.17722, ACL Findings 2026) — 1,000 dialogues of 5–8 turns, 5,887
  QA pairs, six competencies including **Memory Recall**, **Answer Refusal** and **Topic
  Shifting**, videos up to 20 minutes. Best model **76.95%**. Its motivation is that "existing
  evaluation benchmarks remain limited to **single-turn question answering**".
- **SVBench** (ICLR 2025) — "temporal multi-turn question-answering chains"; existing benchmarks
  "fail to evaluate the capacity to sustain temporal reasoning throughout the entire duration of
  video streams".

Both keep the video in the API request and measure **degradation of context-resident video
tokens**, which is exactly the phenomenon this note describes. Neither evidences a stored
understanding, because there is none to evidence.

---

## 6. The third pattern: understanding as a *text artifact*

The two mechanisms above are both "the video is in the request". There is a third, and MiniMax
ships it — which is useful because it makes the trade-off visible in one API.

### 6.1 H3-Context-IR: video in, structured text out

`POST /v2/h3_context_ir`, documented at
`platform.minimax.io/docs/api-reference/video-generation-v2-h3-context-ir`. The endpoint's own
summary:

> "Deeply interpret multimodal context and generate a structured, semantically enriched video
> prompt."

Its input is `text` + optional `image_url` / **`video_url`** (`role=reference_video`) /
`audio_url`, with reference video capped at **3 clips, ≤50 MB each, 2–15 s per clip, ≤15 s total**.
Its output is an async task whose success payload carries `content.prompt` — and in the worked
example, that prompt is a long structured description:

```
integrated_multimodal_description: [Shot 1] Cinematic, wide shot with a slow push in …
[Shot 2] At 00:02.800, the camera cuts to a medium close-up … sudden, blinding flash …
overall_soundscape: Deep, resonant low-frequency thrumming of ship engines …
non_diegetic_music: Symphonic orchestral score, beginning with a slow, rising brass and string
crescendo that abruptly cuts off …
```

The doc's own warning is the whole point:

> "This endpoint **only returns an enhanced prompt**. It does not create a video generation task."

and:

> "H3-Context-IR is a complex system and its implementation is **not open sourced**."

**So the user's read is correct, with three refinements:**

1. **It is genuinely one-shot.** Video in, a text artifact out, nothing retained. There is no
   context to lose, no compaction to survive, no cache to expire — because the video never enters
   a conversation at all.
2. **It is not H3 doing the understanding.** The request carries `model: MiniMax-H3`, and the
   endpoint sits inside the *video generation v2* family, so it is coupled to H3 — but the doc is
   explicit that it does **not** create a generation task. H3-Context-IR is a separate, closed
   system whose output is *a prompt for* H3. The understanding and the generator are distinct.
3. **H3 itself is a video *generation* model**, not a perception model: `MiniMax-AI/MiniMax-H3`
   (★9,990, created 2026-07-30, **no repo licence declared**) ships "Prompt Writing Skill" plus
   eight style-specific *video generation* skills, and points at the video-generation API. Its
   HuggingFace badge links a separate licence. So "pure video models" at MiniMax are on the
   **generative** side — which is the world-model family, not the perception side.

### 6.2 The temporal grounding is the interesting part

Notice the timestamps: `At 00:02.800, the camera cuts to …`. The description is not a generic
caption — it is **time-anchored**. That is what makes a text artifact usable for anything temporal:
a prompt, an index entry, a chapter list, a searchable summary. It also shows what is lost when
understanding is flattened to text — the *model* no longer has the frames, only whoever reads the
text next.

### 6.3 The three mechanisms side by side

| | native tokens (M3 chat) | text artifact (H3-Context-IR) | frames → VLM (sampler) |
|---|---|---|---|
| what carries the understanding | the context window | a prompt string | a tool result |
| retained for later turns | **yes** — until compaction | **no** | only as text, in the log |
| cost per re-use | re-read (cached cheaply) | **re-pay per call** | re-sample |
| failure mode | dropped at compaction; exceeds context | silently lossy; re-derived each call | agent cannot reference what it never sampled |
| best for | conversation *about* a video | prompting, indexing, cataloguing | inspection with tools |

**The choice is driven by one question: does the video need to stay referenceable in a
conversation?** If yes, only context-resident tokens do it. If no — if you want a description, a
prompt, or a category — a one-shot text artifact is *cheaper, safer and stateless*, and it removes
every retention problem in §4 from the design space entirely.

That is the sharpest form of the answer to the original claim: "keeping the understanding" is
sometimes literally true (context-resident), sometimes an accident of the API's statelessness
(re-sending), and sometimes replaced by a text artifact that no longer holds the video at all.

## 7. Corrections this review forced on our own notes

1. **We mislabelled an arXiv reference.** The M3 model card links `arXiv:2606.13392` as "the
   technical report". That paper is **"MiniMax Sparse Attention"** (Lai et al., 11 Jun 2026) — a
   blockwise-sparse-attention operator paper on a 109B natively multimodal model, which does
   **not** describe M3's vision stack. **There is no M3 technical report covering video
   tokenisation or retention.** Corrected in [`vendor-minimax-report.md`](vendor-minimax-report.md).
2. **The `cache_control` on ZCode's video block is unverified against MiniMax**, whose cache docs
   omit video from the cacheable list. Do not copy it on the assumption that it works.
3. **"MiniMax Design" is not "MiniMax-Designer"**, is not open source, does not use M3, and is a
   Tier-B agent workbench. See [`video-product-harness-survey.md`](video-product-harness-survey.md).
