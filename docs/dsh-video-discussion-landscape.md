# The DSH repo's own video and modality discussions

**Date:** 2026-10-15 · Method: read-only sweep of **all 9,627 discussions** in
`deepseek-ai/deepseek-harness` via the REST list endpoint, filtering title + body for `video`,
then reading the relevant threads in full over GraphQL.

**Method note worth keeping:** the repo has **zero issues** — `search/issues` returns `total_count: 0`
even for the bare repo query — because everything goes through Discussions. Anyone searching issues
for a DSH topic finds nothing and concludes wrongly that no one asked. Sweep the discussions.

**Headline:** video input has been asked for **four separate times** in this repo, and the
declaration gate beneath it is a **known, thrice-confirmed, still-open issue** with a patch-ready
proposal attached. **Not one thread has a maintainer reply.** Every response we found is from
another community member.

---

## 1. The four video asks

| # | Title | Author | Date | Comments |
| :-: | --- | --- | --- | :-: |
| [#4725](https://github.com/deepseek-ai/deepseek-harness/discussions/4725) | support pdf/video passthrough | denial123789 | 2026-08-27 | 1 (community) |
| [#6050](https://github.com/deepseek-ai/deepseek-harness/discussions/6050) | 希望 DeepSeek 支持视频阅读：上传视频并理解内容 | Start-Ten | 2026-09-09 | **0** |
| [#9647](https://github.com/deepseek-ai/deepseek-harness/discussions/9647) | Video input: the modality seam is extensible, but the current failure mode is a silent drop | **johnlam1968 (us)** | 2026-10-14 | 2 |
| [#9729](https://github.com/deepseek-ai/deepseek-harness/discussions/9729) | Feature request: video content blocks (`video/*`) alongside ImageBlock | Nonexistent01 | **2026-10-15** | **0** |

**#9729 arrived the day after ours** and asks for the same structural change — a
`VideoBlock { type: 'video' }` alongside `ImageBlock`, plumbed end to end (attachment acceptance,
typed content + compaction rules, pass-through where supported and graceful degradation otherwise).
Its evidence is a search across 289 `@deepseek-ai/*` packages finding no real `video/` handling
anywhere. It carries **no measurement and no fail-open argument**; ours carries both. The two
should be read together.

**#6050 is the user-demand thread** and is worth crediting, because our post argues from model
capability while that one argues from a person: lecture recordings, meeting minutes, "did the
footage say what it meant to", and accessibility for deaf and blind users. Zero comments.

---

## 2. The real wall above the vocabulary: the declaration surface

This is the chain nobody has connected to video yet, and it is the most actionable finding in the
whole sweep. Even if a `video` block existed, **a route could not declare that it accepts one**.

### 2.1 The chain, as reported in [#5837](https://github.com/deepseek-ai/deepseek-harness/discussions/5837)

derekkage, 2026-09-07, for `llm-pi-ai.providers.*.models[]`. A model's resolved `inputModalities`
fall through four steps, ending at text-only:

1. the per-model `input` field on the entry
2. the catalog `base?.input`
3. the provider's `defaultInput`
4. `["text"]` — the default

Two of the five routes they audited were **MiniMax M3** itself (`minimaxai/minimax-m3` on NVIDIA,
`minimax/minimax-m3:free` on OpenRouter): multimodal per the vendor, silently text-only to the
harness. The rest were `z-ai/glm-5.3-flash`, `qwen/qwen3.7-flash`, two Gemma 4 variants and
`nvidia/nemotron-3-nano-omni`.

argszero replied within 16 minutes confirming the chain against source, and noted this was the
**third independent confirmation** of one and the same gap, after
[#5367](https://github.com/deepseek-ai/deepseek-harness/discussions/5367) (Dxboy266, 2026-09-02) —
the `llm-deepseek` twin, where the catalog schema defaults an image-capable model to text-only —
and the `llm-pi-ai` resolution order itself.

A third commenter (`lebedevigor174-dot`, 2026-10-02) re-verified on **v0.2.0-rc.2** that the chain
and the admission gate are **unchanged**, and reported that v0.2.0-rc.2 did add a **visible
"Input types" field** — so the supported workaround is to declare it explicitly:

```yaml
models:
  - id: qwen/qwen3.8-27b
    input: [text, image]
```

### 2.2 Verified independently here, at `d743267388`

```ts
// packages/llm/llm-pi-ai/src/config.ts:80
export const DEFAULT_INPUT: readonly PiAiModality[] = ['text']

// packages/llm/llm-pi-ai/src/catalog.ts:65
function declaredInput(configured) { return configured === undefined || configured.length === 0
  ? undefined : [...configured] }
```

and the gate that consumes the resolved value:

```ts
// packages/api/session-controller/src/commands.ts:339-347
if (model.inputModalities !== undefined && !model.inputModalities.includes('image')) {
  throw new RemoteError('session/attachment-invalid',
    `Model "${current.model}" does not support image input.`, { reason: 'MODEL_DOES_NOT_SUPPORT_IMAGES' })
}
```

**One nuance the reports miss, and it matters:** the guard fires only when
`inputModalities` is **defined and lacks** the modality. An **undefined** declaration passes. So
the failure is not "undeclared = refused" — it is "the chain resolves undeclared to `['text']`,
which is defined, and then refuses". That is why it looks like a model limitation and is not.

### 2.3 The patch that already exists

[#8870](https://github.com/deepseek-ai/deepseek-harness/discussions/8870) — and the identical text
posted three times as **#8870 / #8873 / #8876** by `m0nk111`, 2026-10-04, **zero comments each**.
It makes `discoverModels` read `input_modalities` from a gateway listing, including OpenRouter's
nested `architecture.input_modalities`.

Its own test note is the most useful line in the sweep, because it names the wall in one clause:

> "The recorded OpenRouter fixture expectation gained the modalities its recorded reply declares,
> after dropping values outside the pi-ai vocabulary (`file`/`audio`/**`video`** dropped,
> `text`/`image` kept)."

So a gateway that *correctly publishes* `video` gets the value **stripped by the consumer**. That
is the same structural wall as `MODALITY_GATE` (`llm-pi-ai/src/catalog.ts:48`) and pi-ai's own
`Model.input: ("text" | "image")[]`.

**Read together, the two layers are:** a block type that does not exist (#9729), and a
declaration vocabulary that would discard it if it did (#8870, #5837). Any video work has to
address both, and no thread currently does.

---

## 3. The client-side half, in a separate thread

[#6060](https://github.com/deepseek-ai/deepseek-harness/discussions/6060) (bhzhangsun, 2026-09-09)
locates a **different** wall: DSH's assistant block model is closed.

```ts
// packages/client/ui-conversation/src/client/contract/records.ts:33-38
type AssistantBlock =
  | { kind: 'text'; text: string }
  | { kind: 'reasoning'; text: string }
  | { kind: 'image'; attachment: ImageAttachmentRef }
  | { kind: 'tool-call'; callId; name; argsRaw }
  | { kind: 'other'; block: unknown }   // catch-all, renders as JSON
```

Three gaps, per the author: block kinds cannot be registered (no entry point; `other` is a
deliberate catch-all), only `image` routes to a pluggable slot (`conversation.message.images` is a
**single** slot serving the image family only), and there is no agent-side channel for emitting a
rich block. argszero confirmed all of it against source and drew the boundary between what a plugin
can already reach and what core must extend.

This is the mirror image of the host-side rule in `docs/subsystems/llm-streaming.md:34` — *"A new
modality belongs in the merge-extensible map only when its adapter, UI, compaction, and durable
replay paths honor it."* #6060 is that sentence, discovered independently from the UI side. Related:
[#2995](https://github.com/deepseek-ai/deepseek-harness/discussions/2995) (agent-emitted images
inline, 2 community replies) and [#1969](https://github.com/deepseek-ai/deepseek-harness/discussions/1969)
(generic non-image file upload in the composer, 0 comments).

---

## 4. Two more threads worth crediting

- **[#6049](https://github.com/deepseek-ai/deepseek-harness/discussions/6049)** (ffournierm,
  2026-09-09): `read_image` refuses on a genuinely vision-capable model
  (`zai-org/GLM-5.3-Flash`) because its input is not **declared** — *"the refusal was about the
  declaration, not the model."* Same wall, reached through the tool surface.
- **#2995's reply** from `weijiafu14` (Tsinghua, harness product) shows a **working pi2dsh
  integration**: a Pi tool's image block saved through DSH's native attachment service and rendered
  inline in the conversation, with permission checks intact. It does not close the assistant-side
  gap, but it is a live demonstration that the pi2dsh delivery path our plan depends on does carry
  native media blocks today.

---

## 5. What the sweep changes for us

1. **Our reply is not a cold report** — it is the fourth ask, and it should say so and link the
   others instead of implying novelty.
2. **The strongest addition is the two-layer argument.** Anyone can ask for a `video` block; very
   few will notice that the declaration vocabulary would strip `video` even after the block exists.
   That is our marginal contribution, and #8870 supplies the receipt.
3. **The repo's silence is itself reportable.** Nine threads, six weeks, no maintainer reply. That
   is a legitimate thing to point out once, politely — and it is an argument for the plugin route
   being the only one that depends on nobody's roadmap.

## 6. Sweep method, for reuse

```sh
# Discussions are the only surface: the repo has zero issues.
gh api -X GET search/issues -f q='repo:deepseek-ai/deepseek-harness' --jq .total_count   # => 0

# 9,627 discussions; page the REST list endpoint and filter locally.
gh api --paginate "repos/deepseek-ai/deepseek-harness/discussions?per_page=100" \
  --jq '.[] | select((.title + " " + (.body // "")) | test("video"; "i")) | "#\(.number)\t\(.title)"'
```

Source: the discussion sweep, filtered to `video`. The declaration chain (#5837, #5367, #6049),
the block model (#6060, #2995, #1969) and the patch (#8870) were read in full over GraphQL. No
thread was posted to or modified.
