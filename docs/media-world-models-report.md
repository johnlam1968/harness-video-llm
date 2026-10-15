# Media understanding & world models — state of play, 2026-10-15

## Q1 — World models

The 2026 taxonomy is settled and the survey literature says so in its own words. Three independent 2025–26 surveys split the field explicitly:
- **arXiv 2510.16732** (Oct 2025, v3 Jun 2026) files JEPA under *Temporal Modeling → Global Difference Prediction* ("Global Prediction / Masked / JEPA"), while rendering is a *Spatial Representation* choice ("Decomposed Rendering Representation" vs "Global Latent Vector"). **VERIFIED**
- **arXiv 2606.00133** (28 May 2026) splits *Representation* §3.1.1 "Observation-space (pixel-level)" vs §3.1.4 "Joint-embedding (representation-space) predictions", and *Dynamics* §3.2.3 "Implicit Generative" vs §3.2.4 "Representation-space Predictive". **VERIFIED**
- **arXiv 2607.06401** (Physical Intelligence + Shanghai AI Lab, Jul 2026), §2.3 **"Understanding vs. Predicting: Two Views of the World Model"**: "The JEPA family is positioned in the latent-space simulator region because its defining prediction target is an abstract future representation rather than reconstructed pixel." **VERIFIED**
- **arXiv 2601.07823** (Jan 2026) names the two halves literally: "Diffusion/Flow-Matching Video Models" vs "Video Joint-Embedding Predictive Architecture Models". **VERIFIED**

**The generative/predictive split is real and load-bearing for agents.** LeCun's own test is action-conditioned rollout: "a world model is a predictive system that, given the current state of the world … and given an action that you imagine taking, [you can] predict the next state" (Lemley Lecture, Brown, 1 Apr 2026). **VERIFIED** Meta's CausalVQA (arXiv 2506.09943) supplies the empirical half: VLMs answer "what happened" but fail "what could have happened" — Gemini 2.5 Flash scored 61.66% vs 85–95% human. **VERIFIED** MotionBlind (arXiv 2609.09528, Sep 2026) is the sharpest for a harness: "Neither more frames nor smarter frame selection closes the gap, because these change which frames are seen, not whether motion is read." **VERIFIED** LGWM (arXiv 2610.11942) shows where a JEPA-style predictor beats *every* VLM: for GUI agents, "safety is a property of the transition rather than of the screen," and prompted VLMs are near chance on RSWT-BENCH where the JEPA predictor hits 0.987 AUC. **VERIFIED**

| System | Type | Open weights | 2026 status | Usable as a video-understanding component for an agent today? |
|---|---|---|---|---|
| **V-JEPA 2 / 2.1** (Meta) | Predictive-representation (JEPA); no pixel decoding | **Yes — MIT** (Apache-2.0 for 3 dataset utils). v2.1 = 80M→2B, 384px | v2.1 released 2026-03-16 (arXiv 2603.14482) | **Yes for latency/privacy**, but niche: no generative UI; needs an alignment head for QA |
| **Genie 1–3** (DeepMind) | Generative / interactive simulator | No | Research preview. Project Genie is Ultra-only. 720p @24fps, "consistency for a few minutes," visual memory ~1 min back | **No** — a generator, not an analyser |
| **Cosmos 3 / Reason** (NVIDIA) | Generative (Predict/Transfer) **+** a VLM (Reason) | Yes, NVIDIA Open Model License; `cosmos-predict2` repo archived, moved to `nvidia/cosmos` | Cosmos 3 (1 Jun 2026) unifies reason+predict+transfer+action | Only via **Cosmos-Reason** (a VLM); Predict/Transfer are generation-only |
| **World Labs Marble** | Simulator (Gaussian splats + collision meshes) | No | **Public API** (`docs.worldlabs.ai`); definitive agreement to join AMD | No — it's a scene *producer* |
| **Decart Oasis 3** | Generative interactive | No | **Live public API**, 768×512 gRPC, billed per second | No |
| **Odyssey-3** | Generative autoregressive diffusion | **Not verified** | Public API (JS/Python SDKs) | No |
| **Runway GWM Worlds 2** (3 Sep 2026) | Generative AR diffusion video+audio, 720p/24fps/48kHz, indefinite length | No | **Research preview, contact form, no API**; "Long-term memory is also imperfect," sliding KV window evicts older frames; no paper exists (the publications page links to a *different* paper) | No |

**Is video understanding different from world modeling?** Yes, and the consensus is *complementary, not subsumed*. Three arguments, all verified: LeCun's action-conditioning test; Meta's understand→predict→plan framing; and Video-MME-v2 (arXiv 2604.05015) finding native-video SOTA is "increasingly **saturated**, exposing a critical discrepancy between inflated leaderboard scores and real-world model capabilities," with "a substantial gap between current best model Gemini-3-Pro and human experts." No primary source argues subsumption; the closest is CUA-Suite (arXiv 2603.24440), which argues only that 30fps screen recordings are "a superset of information" over screenshot abstractions — necessary, not sufficient (it still reports ~60% task failure on professional desktop apps).

## Q2 — Media as a harness capability seam

**The seam exists. It is universally image-only, and one gate is genuinely per-model.**

**1. DSH `ctx.attachments` — `AttachmentStore` (abstract seam).** `validateImage` / `saveImage` / `readImage`. The interface is *literally named for images*; there is no `validateVideo`, no audio. **VERIFIED**

**2. DSH per-model modality gate — `PiAiModelProfile.input?: PiAiModality[]`.** This is the exact interface the parent asked about. Docs: "Declaring images is what makes a hand-declared vision model usable … This is a claim about the endpoint, not a check of it: nothing interrogates a gateway for what it accepts." A route-level `defaultInput` covers a whole gateway. **VERIFIED**

**3. DSH `read_image` tool — the gate closes at execution.** "Requires the current model to accept image input"; its "schema is route-independent, and execution refuses unless the exact routed model declares image input." It is not even registered without `ctx.attachments`. The replay adapter exposes `ReplayModelConfig.inputModalities` explicitly "so a scenario can exercise capability gates (e.g. image-capable `read_image`)." **VERIFIED** — this is the cleanest production example anywhere of *tool schema* + *per-model execution gate*.

**4. MCP — no native video type.** Current schema revision `2026-07-28`: `ContentBlock = TextContent | ImageContent | AudioContent | ResourceLink | EmbeddedResource`. Audio was added; **video never was**. Media is therefore *always a tool call*. Real servers: **video-vision-mcp** (260k/mo, ffmpeg + whisper.cpp by default, Sigstore-attested — a media primitive, not a VLM proxy), **video-research-mcp** (581k/mo, 34 tools), **video-url-analyzer-mcp** (144k/mo, the only one using MCP's native image type), **aigroup-video-mcp** (41k/mo, stale ~12mo). **VERIFIED**

**5. OpenRouter — the best public `image yes / video no` gate.** `ModelArchitecture.inputModalities: InputModality[]` where `InputModality = "text" | "image" | "file" | "audio" | "video"`, plus a `modality` string like `"text->text"`. **VERIFIED**

**6. LiteLLM — same idea, weaker contract.** Booleans `supports_vision`, `supports_audio_input`, `supports_video_input`, plus parallel arrays `supported_modalities: text|image|audio|video`. Of 4,159 catalog entries only **99 declare video input**, and 9 of those declare no vision flag at all. Gaps: `supported_modalities` is absent from `litellm/types/utils.py` (untyped in Python) and `supports_video_input` appears **zero times** in the entire docs corpus. **VERIFIED**

**7. OpenAI Agents SDK — media as input, never as a tool.** No `VisionTool`; `input_image` (URL or base64 data URI) is a first-class input item. Tools *return* images via `ToolOutputImage`. Caveat: on the Chat Completions backend image-bearing tool output is silently dropped unless you set `preserve_tool_output_all_content=True`. Issue #159 ("How to utilize vision of LLM?") was closed by its own reporter with **zero maintainer replies**. **VERIFIED**

**Production division of labour.** *Native:* Gemini-3-Pro on Video-MME-v2; CUA-Suite feeding 30fps screen recordings straight in; the Agents SDK's `input_image`. *Separate service:* DSH's `read_image` tool; every MCP video server; LGWM's JEPA predictor as a transition verifier. Both patterns coexist; the split is drawn at *when* you need a prediction, not at *whether* you can see.

## Synthesis

- **Media understanding is converging on a harness capability — but a `text|image` one.** Every harness that has formalised it (DSH, OpenRouter, LiteLLM, MCP, the Agents SDK) has landed on a *per-model input-modality declaration*, and DSH is the only one that also closes the gate at tool-execution time. That is the pattern to copy.
- **The gap is video and audio, not images.** MCP's schema has no video type; DSH's `AttachmentStore` is named `validateImage`; LiteLLM covers 99 of 4,159 models. The `image yes / video no` distinction is *expressible* (OpenRouter's enum proves it) but *almost never populated*.
- **World models do not change the picture for agents watching things.** Generative world models (Genie 3, Marble, Oasis, GWM Worlds 2) are scene *producers* with no analysis path; Cosmos contributes only via its Reason VLM. An agent that watches a screen recording should stay on native video input + a specialist service.
- **World models change the picture only where a prediction is required.** The verified cases are all action-conditioned or transition-level: LeCun's test, CausalVQA's counterfactual/anticipation collapse, MotionBlind's frame-count invariance, LGWM's 0.987 AUC against near-chance VLMs. The correct seam is therefore *not* "add a world model to the harness" but "declare a `predict` modality alongside `input` modalities, and gate it per model — exactly the way `read_image` already gates `input`."
