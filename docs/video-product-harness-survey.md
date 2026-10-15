# Video harnesses & products — tier survey (data fetched 2026-10-15)

## 1. MiniMax-Designer — what it actually is

**It exists, but not under that name.** The official product is **MiniMax Design**
([design.minimax.io](https://design.minimax.io/), [design.minimax.cn](https://design.minimax.cn/)),
announced 2026-08-20 as a multimodal creative **agent workbench**. Press coverage calls it
"多模态创作 Harness": the agent parses the brief, decomposes it, picks text/image/video/audio
models + Skills, adds storyboard planning before generation and editing/assembly after, ships a
"3D director stage", and lets ComfyUI users attach local workflows and have the agent edit
workflow nodes. It does **not** use M3 (M3 is MiniMax's text/coding model; Design orchestrates
H3 + other vendors). Not open source. The only official OSS artifact is
[minimax-desgin-plugin](https://github.com/MiniMax-AI/minimax-desgin-plugin) — name misspelled in
the org — a **ComfyUI plugin, GPL-3.0, 10★**, sync-snapshot of frontend/backend only.

| Project | URL | ★ | License | Tier | Underlying harness | Understands video? |
|---|---|---|---|---|---|---|
| MiniMax Design (hosted) | design.minimax.io | n/a | closed | **B** | MiniMax's own agent workbench + Skill生态 | no public evidence |
| minimax-desgin-plugin | github.com/MiniMax-AI/minimax-desgin-plugin | 10 | GPL-3.0 | **A** | ComfyUI | no |
| MiniMax H3 (model) | github.com/MiniMax-AI/MiniMax-H3 | 9,990 | none declared — *not verified* | **A** | ComfyUI / SGLang | no |
| H3-Context-IR API | platform.minimax.io/docs/api-reference/video-generation-v2-h3-context-ir.md | n/a | closed (*"not open sourced"*) | **C** (capability) | HTTP service, no agent loop | **yes** — interprets text/image/audio/**video** → enriched prompt |
| Hailuo 2.3 + **Media Agent** | minimax.io/news/minimax-hailuo-23 · hailuoai.video/agent | n/a | closed | **B** | Hailuo app agent routing to models | not evidenced |
| MiniMax **Video Agent** API | platform.minimax.io/docs/guides/video-agent.md | n/a | closed | **B/A** | template fill (`/v1/video_template_generation`) | no |
| `videos_understand` MCP (`/matrix/api/v1/mcp/videos_understand`) | — | — | — | **unverified** | absent from official docs index + MCP guide | — |
| MiniMax Code / mmx-cli | github.com/MiniMax-AI/minimax-code (2,029) · /cli (2,191) | 2,029 / 2,191 | MIT / n/a | **D** | MiniMax's own coding-agent harness | no |
| OpenMontage | github.com/calesthio/OpenMontage | 68,504 | AGPL-3.0 | **D** | *your* coding agent (Claude Code/Codex/Cursor…) + skills | **yes** (reference-video start, self-eval) |
| HyperFrames | github.com/heygen-com/hyperframes | 64,283 | Apache-2.0 | **D**/A | 21 skills for Claude Code/Codex/Gemini CLI; deterministic HTML→MP4 | no |
| video-use | github.com/browser-use/video-use | 29,003 | MIT | **D** | Claude Code / Codex / Hermes skill + ffmpeg | **yes** (cuts filler, self-evaluates cuts) |
| Hypit | github.com/hypit-ai/hypit | 22,433 | NOASSERTION — *not verified* | **D** | Claude Code/Codex via `npx skills add` | **yes** (clones a reference video, word-anchored) |
| MoneyPrinterTurbo | github.com/harry0703/MoneyPrinterTurbo | 131,401 | MIT | **B** | plain Python pipeline + LLM calls | no |
| MoneyPrinterV2 | github.com/FujiwaraChoki/MoneyPrinterV2 | 32,115 | AGPL-3.0 | **B** | plain pipeline | no |
| ShortGPT | github.com/levismithxyz/ShortGPT | 0 (dormant, forks alive) | NOASSERTION | **B** | plain scripts | no |
| diffusionstudio/editor | github.com/diffusionstudio/editor | 3,317 | MPL-2.0 | **D** | Codex, Claude Code, OpenCode, **Pi** | **yes** (`probe`/`grab`/`filmstrip`/`transcribe`/`listen` over MCP+CLI) |
| OpenCreator (ex-KrillinAI) | github.com/krillinai/OpenCreator | 13,184 | Apache-2.0 | **D** | **Codex CLI as the execution engine** | **yes** (Whisper + LLM subtitle/segmentation) |
| Crayotter | github.com/idwts/Crayotter | 228 | NOASSERTION | **C** | **LangGraph `StateGraph`** (`script/graph.py`) | **yes** (per-video multimodal analysis) |
| FireRed-OpenStoryline | github.com/FireRedTeam/FireRed-OpenStoryline | 3,489 | Apache-2.0 | **C**(+D) | own 3-stage pipeline; Claude Code / OpenClaw skills | **yes** (clip segmentation, content + emotion recognition) |
| HKUDS/VideoAgent (EMNLP2026) | github.com/HKUDS/VideoAgent | 1,949 | MIT | **C** | bespoke tool-using agent | **yes** (understanding + editing + remaking) |
| HKUDS/ViMax | github.com/HKUDS/ViMax | 12,664 | MIT | **B/C** | bespoke multi-role pipeline | partial (frame-level consistency QA) |
| Microsoft DeepVideoDiscovery | github.com/microsoft/DeepVideoDiscovery | 423 | MIT | **C** | deep-research QA agent | **yes** |
| Toonflow / ArcReel / seedance-2.0 / moyin-creator | HBAI-Ltd/Toonflow-app · ArcReel/ArcReel · Emily2040/seedance-2.0 · MemeCalculate/moyin-creator | 17,851 / 5,519 / 7,701 / 4,722 | MIT / AGPL-3.0 / MIT / AGPL-3.0 | **B** | GUI + MCP + agent assist | no |
| waoowaoo | github.com/waooAI/waoowaoo | 14,559 | NOASSERTION | **D** | ships a Codex runtime image | partly |
| Model repos (Wan2.2 · Open-Sora · HunyuanVideo · LTX-Video) | Wan-Video/Wan2.2 · hpcaitech/Open-Sora · Tencent-Hunyuan/HunyuanVideo · Lightricks/LTX-Video | 17,946 / 29,873 / 12,622 / 11,106 | Apache-2.0 / Apache-2.0 / Tencent / Apache-2.0 | **A** | training+inference code | no |

**Not found / not verified:** `Agent-V`, `VPG` (video podcast generator), `VideoAgent2` (arXiv
2504.04471 exists, **no official repo located**), `AgentDirector`, `AutoCut` (only toy repos,
≤3★), `Revid` (closed product), `video-understand`-style OSS wrappers.

## 2. Synthesis

**The agent-harness layer is real — and in 2026 its default shape is Tier D, not a bespoke
framework.** diffusionstudio/editor states it outright: "Edit videos with Codex, Claude Code,
OpenCode, or Pi." OpenMontage's pitch is "turn your AI coding assistant into a full video
production studio." HyperFrames, video-use, Hypit and FireRed-OpenStoryline all ship as *skills*
consumed by a general coding agent; OpenCreator goes furthest and adopts a harness as its
execution engine (Codex CLI). Against that, only **one** surveyed project uses a mainstream agent
framework: Crayotter on **LangGraph**. CrewAI, AutoGen, OpenAI Agents SDK, OpenHands and DSH
appear nowhere in production video code. So: yes — several video products genuinely run
underneath a general-purpose coding-agent harness, and the harness is now the integration seam
(MCP servers + `diffusion`-style CLIs + ffmpeg), not the orchestration logic.

**Video *understanding* (Tier C) exists, but it is narrow and mostly CLI/MCP-shaped.** The loop
"look at frames → decide → re-cut" is real in: diffusionstudio's `probe`/`grab`/`filmstrip`/
`transcribe`/`listen` (the agent literally cannot cut footage it has not inspected), video-use's
self-evaluation at every cut boundary, Hypit's word-anchored clone of a reference video,
OpenMontage's start-from-a-reference-video mode, Crayotter's per-video multimodal analysis, and
HKUDS/VideoAgent. Note what they watch: extracted frames and word-level transcripts handed to a
VLM — not an end-to-end video model with an agent loop around it. Vendors keep understanding
*behind* the API: MiniMax's H3-Context-IR is the only confirmed closed Tier-C surface (video +
audio + image in, enriched prompt out, explicitly "not open sourced") — it informs a generation
prompt, it does not decide an edit.

**Tier B is where the volume and the stars are, and it is script-generation, not reasoning.**
MoneyPrinterTurbo (131k★), MoneyPrinterV2, ShortGPT, Toonflow, ArcReel, seedance-2.0, moyin:
LLM writes the script/prompts, stock libraries + TTS + ffmpeg do the rest, and no output ever
feeds back. Tier A model repos (Wan2.2, Open-Sora, HunyuanVideo, LTX-Video) have no agent layer
whatsoever — including MiniMax's own open-sourced H3. And the naming gap is stark: half the
projects the brief named (Agent-V, VPG, AgentDirector, AutoCut) do not exist as real repos,
while vendor "Agents" (Hailuo Media Agent, MiniMax Video Agent) are template filling and model
routing.

**Verdict for MiniMax specifically.** Design = Tier B agent workbench (planning + model/skill
orchestration, ComfyUI escape hatch, no public footage-understanding evidence); H3 = Tier A
engine with a closed Tier-C prompt-IR sidecar; the API "Video Agent" = template assembly; the
`videos_understand` MCP path `/matrix/api/v1/mcp/videos_understand` is **not publicly documented**
— it appears in neither `platform.minimax.io/docs/llms.txt` nor the official MCP tool guide, whose
video tools are `generate_video` / `image_to_video` / `query_video_generation` only. Treat it as
**not verified** unless you hold a vendor doc.

## 3. URLs fetched

- https://design.minimax.io/ (linked from minimax.io nav)
- https://github.com/MiniMax-AI/minimax-desgin-plugin · raw README
- https://api.github.com/orgs/MiniMax-AI/repos (37 repos, stars/licenses)
- https://www.minimax.io/news/minimax-hailuo-23
- https://www.dtinsight.com.cn/sys-nd/4088.html
- https://platform.minimax.io/docs/llms.txt · /docs/guides/mcp-guide.md · /docs/guides/video-agent.md · /docs/api-reference/video-generation-v2-h3-context-ir.md
- https://raw.githubusercontent.com/MiniMax-AI/MiniMax-MCP/main/minimax_mcp/{server,const}.py
- GitHub API `repos` + `search/repositories` for every project in the table
- READMEs: calesthio/OpenMontage, browser-use/video-use, krillinai/OpenCreator, idwts/Crayotter,
  hypit-ai/hypit, HKUDS/ViMax, FireRedTeam/FireRed-OpenStoryline, diffusionstudio/editor,
  HBAI-Ltd/Toonflow-app, waooAI/waoowaoo, heygen-com/hyperframes
- Search results only (not opened): arXiv 2504.04471 (VideoAgent2), arXiv 2511.08521 (UniVA)
