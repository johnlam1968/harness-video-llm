# Media understanding is bigger than a harness — and that is the harness question

**Date:** 2026-10-15 · Companion to
[`multimodal-use-cases-and-coding-value.md`](multimodal-use-cases-and-coding-value.md) (which asks
what video input is *for*) and [`video-retention-mechanism.md`](video-retention-mechanism.md)
(which corrects what "the model keeps the understanding" means).

The instinct behind this note is right and worth taking seriously: video understanding is not a
Pi problem or a DSH problem. It is a platform problem — YouTube, surveillance indexing, broadcast
archives, the video-production tools — and coding harnesses are downstream of it. But that does
not make it irrelevant to a harness. It makes **the seam** the relevant design question, and the
seam is a harness decision. Below: what already exists, the ladder every agent sits on, and where
a flexible harness can honestly draw its boundary.

*(Sections marked ⏳ await the two external surveys in flight; the structural sections are complete.)*

---

## 1. The claim, sharpened

> "Video understanding is bigger than a particular harness… how a flexible harness handles media
> understanding is important for its scope, use cases, and growth. Of course, developers might have
> boundaries for harness capabilities."

Both halves are true, and the tension between them is the whole design question. Harness scope is
set by *what the harness owns*: the loop, the durable record, the tool surface, the model
connection. It is **not** set by what a model can perceive. A harness that tries to own perception
re-implements a VLM pipeline; a harness that ignores perception ships a transcript where the user
attached a film.

---

## 2. What a harness already owns, and what it must not

Verified in this repo and in the DSH source at `d743267388`:

| concern | who owns it | consequence if the harness gets it wrong |
|---|---|---|
| The durable record of what was attached | harness | an unloggable attachment is un-reconstructable and un-diagnosable |
| Whether the model could see it | harness | a silent drop is the worst failure available (this repo's whole thesis) |
| Honest statement when it could not | harness | "model does not support video" stated as a fact about the *model* when it is a fact about the *declaration* |
| Replaying it next turn | harness | the media vanishes mid-conversation with a clean transcript |
| Frame extraction, OCR, ASR, embeddings | a tool or service | re-implementing these inside the loop is waste |
| The perception itself | the model | not the harness's to own |

**The boundary that follows is not arbitrary**: the harness owns *transparency about perception*, and
delegates the perception. That is a much smaller, cheaper and more defensible scope than owning media
understanding — and it is exactly what the four native-video implementations we surveyed do (each
delegates to the model, each makes the omission visible, none samples frames itself).

---

## 3. The ladder: five tiers of media understanding in an agent

Not all media understanding is equal, and most confusion in this area is tier confusion. Every real
system sits on one of these, and they differ in cost, state and failure mode more than in accuracy.

| tier | what happens | who does it | cost | where the state lives | failure mode |
|:-:|---|---|---|---|---|
| **T0 — metadata** | filename, mime, duration, size, EXIF | harness / filesystem | ~free | request | wrong metadata misleads triage |
| **T1 — text extraction** | ASR transcript, OCR text, subtitle track | **external service** (`dsh-speech-funasr-http`, SenseVoice) | seconds, local, cheap | becomes text in the context | silent partial transcript reads as complete |
| **T2 — sampled frames → VLM call** | ffmpeg scene sampling → VLM, agent calls it as a tool | **tool/plugin** (`dsh-video-lens`, `video-use`) | ~15 screenshots-equivalent per clip | tool result in the log | agent cannot reference what it never sampled |
| **T3 — native media tokens** | video enters the chat model's context | **model** (`{type:'video'}`, fps-sampled) | high, in-context | the context window — and nowhere else | dropped at compaction (DSH today) |
| **T4 — persistent representation** | learned state of the scene/world | **world model** | research-stage | a learned space | not shipping; see ⏳ |

**The single most useful thing in this table**: T1 and T2 are *already built* on this host —
`dsh-speech-funasr-http` and `dsh-video-lens` respectively. T3 is what this repo is working on. T4 is
the research horizon. The ladder is not a roadmap; it is the landscape most agents already occupy,
usually without knowing which tier they are on.

### 3.1 The division of labour that follows

Most "video understanding" products are **T1 + T2**, and they work. The honest reading of the
product survey in [`video-product-harness-survey.md`](video-product-harness-survey.md) is that the
practical video agent is a **triage-and-tool system**, not a perceptual one: a cheap stage
(transcript, scene detection, metadata) decides whether and how to spend the expensive stage
(frames into a VLM, native video into the chat model). Nothing in the surveyed literature requires
skipping the cheap stage, and for cataloguing — the "quickly scan and categorize" case — the cheap
stage is usually sufficient.

**For cataloguing specifically, T0+T1 is very often enough.** A library indexer needs: is this worth
a human, what is it about, does it duplicate something. Transcript plus metadata plus cheap
embeddings answers all three. Native video input is the *last* tier to reach for, not the first.

---

## 4. The decision-model tier: System One cannot see, but it can triage

The instinct to use a decision model (Jev / System One) for fast categorisation is sound in shape
and **blocked in fact**, and the block is documented in our own research
(`system1-runtime-repo/docs/DECISION_MODEL_RESEARCH.md:99`):

> | Modality | Text only (string / JSON object / array of strings). **No image, audio, video** |

Weights are closed; there is no paper. So a decision model can never look at a frame. What it *can*
do — and do extremely well, being a probability-distribution model rather than a generator — is
triage **over the outputs of a cheaper tier**:

> T0/T1/T2 outputs (transcript, scene captions, metadata, embeddings)
>   → decision model: *is this worth T3? is this a duplicate? does this match the user's intent?*
>   → gate, route, or spend the expensive tier

That is a two-stage architecture, and it is precisely what the runtime in
`system1-runtime-repo` already implements for *text* inputs — the engine, the carrier, the probe,
the judge. Extending it to media is therefore not a new idea; it is the same pattern applied one tier
down the ladder. And it is the honest answer to "cheaply scan and categorize a lot of video":
**decision-model triage over T1 text, with T2 or T3 spent only where the gate says it matters.**

The failure mode to design against is the one our own [`docs/dsh-video-discussion-landscape.md`](dsh-video-discussion-landscape.md)
catalogues: a tier that returns "unparsed" silently and a decision model that reads it as "fine".
Every gate needs a visible disposition, which is the same rule as the visible omission.

---

## 5. Why a harness's handling of media is a scope and growth question

The strategic case, stated plainly:

1. **Media is where users already are.** A coding agent that cannot open a screenshot of a bug or a
   10-second screen recording is visibly behind its competition; several surveyed harnesses ship
   exactly those ingress paths as their differentiator.
2. **Transparency is a growth surface, not a tax.** Making the *failure* legible — "I could not see
   this video, here is why" — is a small amount of code that converts a correctness bug into a
   product decision the user can act on. It is also the cheapest possible answer to "video in,
   hallucinated answer out".
3. **The seam is where ecosystems attach.** Every capability seam that is designed invites plugins;
   every one that is hard-coded gets a wrapper instead. The DSH plugin ecosystem's video work is
   *entirely* T2 — because that is the only tier a plugin can reach.
4. **The boundary keeps the harness honest.** Declaring "I do not own perception; I own honesty
   about it" is a scope that a small team can hold for years. Declaring "I will understand your
   video" is a scope that requires a VLM team.

The risk in that argument is real too, and should be stated: a harness that says "attach a video"
and then cannot is worse than one that never offers it. So the boundary must come with the
**enforced** visible disposition, or the scope claim is just marketing.

---

## 6. Where DSH stands today, tier by tier

Verified in this repo and the DSH source at `d743267388`:

| media | native in the harness? | what exists | gap |
|---|---|---|---|
| **text** | yes | `ContentBlockMap.text`, typed, durable, logged | — |
| **image** | yes | `ImageBlock`, durable attachment store, per-model modality gate, compaction offload, client projection | — (the model of what a first-class media path looks like) |
| **audio** | **no** | delegated: `dsh-speech-funasr-http`, SenseVoice server → transcript text | no native block; T1 only |
| **video** | **no** | delegated: `dsh-video-lens` (scene sampling → VLM), Pi-side in progress | no native block, no modality declaration, and **compaction keeps only text blocks**, so a video placed natively today would be dropped mid-conversation |
| **any media** | — | a plugin surface that plugins use heavily (all the above are plugins) | the seam exists in practice, undocumented as a seam |

Two conclusions, one uncomfortable:

- **The uncomfortable one**: DSH's compaction seam keeps only text blocks and its offload path is
  image-only. A `video` block that reached the context today would be dropped at the first
  compaction with no placeholder. The tier-3 story is not "video works" until compaction,
  projection and replay all honour the block — which is exactly the rule
  `docs/subsystems/llm-streaming.md:34` states for a new modality.
- **The designable one**: the four-tier ladder gives a harness a *progression* rather than a
  yes/no. T1 and T2 are shipped as plugins. T3 is a vocabulary + adapter + compaction change.
  T4 is a research question. A harness that documents these four tiers and their contracts has a
  media story that survives its own roadmap.

---

## 7. Platform scale: what YouTube actually shows, and the real indexing stack

*(Surveyed 2026-10-15. Marked VERIFIED where a primary source was fetched.)*

### 7.1 YouTube's video understanding is real, and its architecture is private

The instinct that "YouTube must have an underlying video model" is **correct**, and YouTube's own
shipped features prove it — each one is impossible without per-video understanding of frames and
audio:

| shipped feature | date | what it proves | model credited |
|---|---|---|---|
| [Auto dubbing](https://blog.youtube/news-and-events/auto-dubbing-on-youtube/) | 2024-12-10 | per-video audio → text → translated speech | Google DeepMind + Google Translate (stated) |
| [Expressive Speech](https://blog.youtube/news-and-events/youtube-auto-dubbing-expressive-speech/) | 2026-02-04 | prosody/emotion modelling; 27 languages; **6M+ daily viewers** watching ≥10 min of dubbed content | not stated |
| Automatic Smart Filtering | 2026-02 | **video-level** classification deciding when *not* to dub (music, silent vlogs) | not stated |
| [Ask YouTube](https://blog.youtube/news-and-events/youtube-news-google-io-2026/) | 2026-05-19 | conversational retrieval "across **all of YouTube's catalogue**", long-form and Shorts, with follow-up turns | not stated |
| Gemini Omni (Shorts Remix) | 2026-05 | video **and** audio manipulation | Gemini (stated) |

**But the architecture is not public.** The canonical Google Research publication is *Deep Neural
Networks for YouTube Recommendations* (2016) — a recommendation model, not a video-understanding
one. No Google Research page describes the current video stack, its scale, or its latency budget.
So the honest reading is: **YouTube's advantage is the corpus, the human labels and the serving
budget, not a publicly better video model.** It publishes outcomes and never architectures.

**Why this matters for a harness.** Ask YouTube is the interesting one: conversational retrieval
over an entire video corpus, with follow-up turns. That is the *platform* version of the capability
this repo is arguing a coding harness should have — and it is shipped, to Premium users, which sets
the bar rather than settling the question.

### 7.2 The realistic "scan and categorize a library" stack

Assembled from what is actually usable today, with prices where a primary source gave one:

```
ffprobe  →  ffmpeg scene-cut detection  →  faster-whisper (ASR)
        →  CLIP/SigLIP2 frame sampling  →  embeddings  →  vector DB
        →  one VLM pass per video (Qwen3-VL-8B) for a summary + category labels
```

- **faster-whisper** (MIT) — **13 minutes of audio transcribed in 16 s on a single RTX 3070 Ti**
  (int8, batch 8), ≈48× realtime. This is the tier that makes everything downstream affordable.
- **Qwen3-VL** (**Apache-2.0, verified**) — 2B→235B, hours-long video, 256K→1M context,
  timestamp-grounded localisation. The strongest permissively-licensed candidate, and the one any
  self-hosted stack should start from.

**Managed alternatives.** AWS Rekognition does labels, faces, people-pathing, celebrities,
moderation and segments — **no embeddings** — at **$0.10/min** for archived video and **$0.05/min**
for shot detection (official AWS pricing). Azure Video Indexer runs 30+ models and returns an
insights document, also without embeddings. **Twelve Labs is the only managed service verified to
return video embeddings**, billed in tokens plus storage rather than per-minute, so library-scale
cost is harder to forecast. (Google Video Intelligence and Azure pricing could not be retrieved —
404s — so no claim is made about either.)

**Conclusion that lands on §3:** this stack contains **no native video tokens anywhere**. Cataloguing
at scale is T0+T1+T2 with one VLM pass per video, and T3 — handing the video to the chat model —
is absent from it. That is strong independent evidence for the ladder: the honest production
answer to "scan and categorize video" never needs native video input.

### 7.3 The trap worth warning people about: model licences ≠ repo licences

| project | repo licence | model licence |
|---|---|---|
| **VideoLLaMA3** | Apache-2.0 badge | **"research preview intended for non-commercial use ONLY"** (verified in its own README, sitting directly beneath the Apache badge) |
| **TimeChat** | permissive-looking | **non-commercial research preview**; 8×V100 to train |
| VideoChat2 / VideoChat3 | research | QA/caption output only — **no vector output, so not an indexer** |

Verified directly: `DAMO-NLP-SG/VideoLLaMA3` carries an Apache 2.0 badge and, three lines below,
"subject to the model Licenses of Qwen, Terms of Use of the data generated by OpenAI and Gemini".
**Always check the model licence, never the repository's.** For anything we might publish or build
on, that single check is the difference between a usable dependency and a legal problem.

### 7.4 The tier boundary, restated with the evidence

§3 argued that T3 is the last tier to reach for. The survey confirms it from the opposite
direction: labels, shots, transcript and moderation are **solved commodity** at $0.05–0.10/min,
while open-ended video *semantics* ("what is this video actually about") is **still frontier work**
— a 4B video model in July 2026 is only just overtaking a general VL model of similar size.

So the division is not cost alone. It is that **closed-set labelling and retrieval are commodities,
and open-ended understanding is not.** A harness that needs to *know what is in* a video is in the
commodity tier and should use T1/T2. A harness that needs to *reason about* a video with the user,
turn by turn, is in the research tier — which is the honest reason video input is worth having
*and* worth being careful about.

---

## 8. ⏳ Awaiting: world models and media capability seams

One survey is still in flight, covering whether "video understanding" is subsumed by, or
complementary to, world models (Genie / Cosmos / JEPA), and whether any harness or framework
already exposes media understanding as an explicit, per-model-gated capability seam.

Its conclusions matter here: if a harness-level media seam is converging on a standard, this repo
should align to it; if world models change what "understanding a video" means, the T3/T4 boundary in
§3 is where it shifts.

## 9. References in this repo

- [`docs/multimodal-use-cases-and-coding-value.md`](multimodal-use-cases-and-coding-value.md) — what video-capable models are for; the coding-value evidence review
- [`docs/video-retention-mechanism.md`](video-retention-mechanism.md) — what "the model keeps the understanding" means (it does not; the frames are context tokens)
- [`docs/dsh-video-discussion-landscape.md`](dsh-video-discussion-landscape.md) — the four video asks in the DSH repo and the declaration-surface gap
- [`docs/video-product-harness-survey.md`](video-product-harness-survey.md) — video products tiered by whether an LLM reasons over footage
- [`docs/evidence.md`](evidence.md) — the reproducible measurements the rest rests on
- `system1-runtime-repo/docs/DECISION_MODEL_RESEARCH.md:99` — the decision model is text-only
