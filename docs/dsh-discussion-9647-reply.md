# Reply draft — deepseek-harness Discussion #9647

**Provenance (not part of the comment).**

- The post came from session `session-659e3d91-fae3-41f9-87c9-c5f378f5f853`, titled
  *"Add MiniMax-M3.1-Flash-Preview model support to pi-ai"*, cwd `/home/john/CodingProjects`,
  `createDiscussion` at turn 33 step 18 — 2026-10-14T02:13:31Z, against the discussion's
  `createdAt` 02:13:32Z.
- `session-362b0f1f-3d5b-42e3-8989-39bf6087d358` (*"… (1)"*) is a **seeded fork** of it
  (`parentSession: session-659e3d91…`, created 2026-10-14T06:32:22Z). Its log carries a copy of
  the same events, which makes it look like a second poster; it is not one.
- Source citations re-checked at the revision the post names — `/home/john/deepseek-harness` @
  `d743267388`, `0.2.1-alpha.2` — with upstream `main` checked where noted.
- Replies being answered: `DC_kwDOT3T1g84BIJi8` (argszero, 2026-10-14T02:20:59Z) and
  `DC_kwDOT3T1g84BIQPi` (JannetGen, 2026-10-15T04:48:29Z).
- Thread references in the draft were read on 2026-10-15 via the discussion sweep recorded in
  [`dsh-video-discussion-landscape.md`](dsh-video-discussion-landscape.md).

---

Thanks — both replies are useful, and one of them changes the plan. Repro question first.

## This is the fourth ask, and the walls are already mapped

Before my own answer: video has been requested in this repo four times —
[#4725](https://github.com/deepseek-ai/deepseek-harness/discussions/4725) (2026-08-27),
[#6050](https://github.com/deepseek-ai/deepseek-harness/discussions/6050) from a user whose use
cases are lecture recordings, meeting minutes and accessibility (2026-09-09), this one, and
[#9729](https://github.com/deepseek-ai/deepseek-harness/discussions/9729) yesterday, asking for
essentially the same `VideoBlock` this post proposed. Credit where it is due — @Start-Ten's thread
argues from a person rather than from a capability table, and it deserves the same weight.

More useful is that the layer *underneath* this one is already diagnosed, repeatedly and in
detail, and it changes what a fix has to touch:

- [#5837](https://github.com/deepseek-ai/deepseek-harness/discussions/5837) — `llm-pi-ai` custom
  provider entries silently resolve to text-only when the model is genuinely multimodal. It names
  `minimaxai/minimax-m3` and `minimax/minimax-m3:free` among the affected routes.
- [#5367](https://github.com/deepseek-ai/deepseek-harness/discussions/5367) — the same gap on the
  `llm-deepseek` side, where the catalog schema defaults an image-capable model to text-only.
- [#6049](https://github.com/deepseek-ai/deepseek-harness/discussions/6049) — `read_image`
  refusing a vision-capable model for the same reason.
- [#8870](https://github.com/deepseek-ai/deepseek-harness/discussions/8870) — a patch-ready
  proposal to read `input_modalities` from gateway listings (posted three times as #8870/#8873/
  #8876).
- [#6060](https://github.com/deepseek-ai/deepseek-harness/discussions/6060) — the client-side
  twin: assistant block kinds cannot be registered, and `other` is a deliberate JSON catch-all.

So there are two walls, not one, and this post only covered the second: **a `video` block does not
exist** (#9729), and **the declaration vocabulary would discard it if it did**. #8870's own test
note says so in one clause — after mapping an OpenRouter listing, values outside the pi-ai
vocabulary (`file`/`audio`/**`video`**) are dropped. I verified that chain here too: `DEFAULT_INPUT
= ['text']` at `packages/llm/llm-pi-ai/src/config.ts:80`, and the admission gate at
`packages/api/session-controller/src/commands.ts:341` rejecting when the resolved
`inputModalities` is defined and lacks `image`.

One nuance those reports leave out, because it explains why this is so confusing in the field: the
guard fires only when `inputModalities` is **defined and missing** the modality. `undefined`
passes. The failure is not "undeclared models are refused" — it is that the resolution chain turns
an undeclared model into `['text']`, which *is* defined, and then refuses on that. The user sees
"this model does not support video" and believes the model.

## On the repro

There is no error to reproduce, and that is the finding: it is not a configuration problem and
not a crash — the request succeeds. Two halves.

**1. The drop, minimally.** `packages/llm/llm-pi-ai/src/context.ts:67`, `userContent()`,
switches on `block.type` with cases for `text` and `image` only and ends:

```ts
default:
  // Other merge-extensible blocks are not user-input vocabulary for pi-ai.
  break
```

Anything else in a user message never reaches the wire and nothing is logged. So the repro is:
put a `video` block (or any merge-extensible block the adapter does not know) into a user
message, send it, and read the request body — it succeeds, and the block is gone. There are no
logs to attach precisely because nothing recorded the omission; the artifact is the absence. A
fix looks like the visible-omission rule below: substitute explicit placeholder text, and count
it.

**2. Why the omission costs something, provider-side.** Measured 2026-10-13 against
`https://api.minimaxi.com/anthropic/v1/messages` (credential resolved from a running DSH
process; value never printed). The evidence is a direct HTTP probe, not a harness — worth saying
plainly, since "the model reads video" and "a harness can deliver a video" are different claims
and only the first is established by this test.

```bash
ffmpeg -hide_banner -loglevel error -y \
  -f lavfi -i "color=c=blue:s=320x240:d=4,drawtext=text='FRAME %{n}':fontcolor=yellow:fontsize=28:x=20:y=100" \
  -f lavfi -i "sine=frequency=880:duration=4" \
  -c:v libx264 -pix_fmt yuv420p -c:a aac -shortest probe.mp4
```

| content block sent | result |
|---|---|
| `{"type":"video","source":{"type":"base64","media_type":"video/mp4","data":…}}` | **200** — *"…incrementing yellow-and-white frame numbers from 01 to 85."* |
| `{"type":"image","source":{"type":"base64","media_type":"video/mp4","data":…}}` | **400** — `image data url media type "video/mp4" not supported (2013)` |
| `{"type":"audio",…}` | **400** — `unsupported content type 'audio' (2013)` |

The frame number is burned into the picture with `drawtext`, so a correct answer cannot come from
metadata. The second row is the one that matters for a harness: the *same bytes* sent as an image
block are refused, which is why any video path has to rewrite the block type rather than smuggle a
MIME through an image slot.

## A correction to my own post

The comparison table above attributes this placeholder to OpenClaw:

```
[Media omitted from provider request because the selected model does not support video input.]
```

That string is **ZCode's**, not OpenClaw's — 0 code-search hits in `openclaw/openclaw`. It comes
from `zai-org/zcode apps/zcode-cli/packages/contracts/src/model/media-policy.ts`, templated over a
typed reason, and ZCode's claim to `omittedVideoCount` counters belongs there too. OpenClaw's own
visible string is different —
`packages/ai/src/provider-transcript-transform.ts:12`:

```ts
const VIDEO_OMISSION = "(video omitted: provider does not support video input)"
```

Both omit visibly, so the argument in my post stands unchanged; the citation was mine and it was
wrong. Correcting it here rather than quietly, since the whole point of the report is that a
silent error is worse than a loud one.

## On the structural correction (argszero)

**Accepted — it kills the shape I sketched.** `userContent()` is module-private, and
`packages/llm/llm-pi-ai/src/index.ts` exports `PiAiAdapter`, `PiAiAdapterOptions`, `Config`,
`recordKeyFor`, `supportedProtocols`, `name`, `inject`, `apply` — not the converter.
`registerAdapter` refuses a route another adapter holds: `prepareRoutes` throws
`DUPLICATE_ADAPTER` (`packages/llm/llm/src/index.ts:431`). Both verified at `d743267388`. So a
plugin cannot widen an existing pi-ai route's content vocabulary, and the open route is a new
provider route with its own adapter. That is the plan now.

One further constraint pushing the same way, which I had not checked: **DSH has no `onPayload`
equivalent.** `GenerateOptions` has no provider-options passthrough, and `PiAiAdapter` hands
`models.streamSimple` only the profile options plus
`temperature`/`maxTokens`/`sessionId`/`signal`/`headers` (`adapter.ts:402`). The only
request-side extension seam in the LLM layer is `@deepseek-ai/dsh-deepseek-llm-api-extensions`,
which merges **top-level fields** into the DeepSeek Messages body — a base-field collision is a
`REQUEST_EXTENSION` error — on the DeepSeek route only, and cannot rewrite `messages[].content`.
The one request-adjacent seam that does take structured input is the **provider** registration on
`dsh-session-llm-title` (`ctx.sessionTitle.register({…})`), and it selects the model that produces a
title — not the request body. So on the DSH side the omission decision has to be made **before** the
request is built, not at the wire — which happens to be what MiniMax-Code does, and is why its
design transfers.

**Where I have to disagree — §2, the file paths.** At the revision this post cites
(`d743267388`, `0.2.1-alpha.2`), `@deepseek-ai/dsh-llm-deepseek` has one serializer,
`packages/llm/llm-deepseek/src/serialize.ts` (168 lines), not `protocols/messages/serialize.ts`;
`packages/` contains no `protocols` directory at all, `grep -rn chat-completions
packages/llm/llm-deepseek/src` returns nothing, and upstream `main` today lists the same layout.
Both DeepSeek paths **throw**: assistant content at line 36, user/tool-result content at line 67,
both through `unsupported()` → `UNSUPPORTED_CONTENT` (line 11). And `chat-completions` is not a
selectable DeepSeek protocol — `config.ts:207` rejects the field outright:

```
llm-deepseek: protocol is not configurable; remove it and use a Messages-compatible baseURL
```

So "same durable log, opposite outcomes, decided by which protocol the route defaults to" does not
hold as written. What does hold is sharper, and it is why I asked: the asymmetry is **per adapter
family, not per protocol**. `llm-deepseek` throws; `llm-pi-ai` drops — and it drops on *every*
protocol pi-ai serves, because `userContent()` assembles the pi-ai `Context` before pi-ai chooses
an api (`anthropic-messages`, `openai-completions`, the Responses protocols). Same block, same
route, one api choice later: still dropped. If you are reading a newer revision with a `protocols/`
layout, say which commit and I will re-read it — I cannot substantiate the one I cited.

**§3 accepted, with two sharpenings.** (a) `next` takes no arguments:
`'llm/stream'(this, options, next: () => AsyncIterable<StreamChunk>)` (`llm/src/index.ts:75`), and
`streamWithRegistration` closes over `options` in the arrow it passes to `ctx.waterfall`.
Re-dispatch is therefore not a workaround but the only expressible route — which is what your
prior art does: `@argszero/cordis-plugin-history-sanitizer@0.1.1` (npm, MIT) re-dispatches as a new
object with `prepend: true`. (b) On not marking the copy: agreed, and the sharper reason is that
the marker has a **live consumer**, not only a test invariant — `isAgentLoopRequest` gates the
title service (`packages/session/session-title/src/index.ts:547`), so a marked re-dispatched copy
enters `onMainRequest` and can schedule a second title call for the same pending revision.

`ReplayEnvelope.blocks` confirmed verbatim: "one entry per emitted block in first-seen stream
order. When assembly drops a block it drops the entry at the same position; entries whose length
does not match the emitted block count discard the whole envelope" (`llm/src/types.ts:430`).

## One thing neither reply raised, which decides whether this ships

The durable half. `ContentBlockMap` is merge-extensible, but the harness states the admission
rule for a new modality at `docs/subsystems/llm-streaming.md:34`:

> A new modality belongs in the merge-extensible map only when its adapter, UI, compaction, and
> durable replay paths honor it.

Concretely: the v4 session format namespaces an unknown content tag as `plugin:<type>`
(`packages/session/session-format-v3-to-v4/src/content.ts:15,45`), so a plugin's video block is a
plugin-namespaced tag that also needs a session projection for the Web UI and a compaction path
that preserves it. Otherwise the request half works and the log cannot reconstruct the attachment
after a reload — model-visible must equal logged. There is a working precedent for the shape:
`fileHandleText` already projects every durable file to visible handle text on every route
(`packages/llm/llm/src/content.ts:151`), and a video a route cannot carry should degrade to
exactly that rather than vanish.

## Plan, updated

- **A new route with its own adapter**, not "add video to MiniMax" — the registry forbids taking
  over an existing one — emitting
  `{"type":"video","source":{"type":"base64","media_type":"video/mp4","data":…}}`, with the
  decision made at ingress where it is reachable.
- **Visible omission with a typed reason**, in the style of ZCode's
  `UnsupportedModelInputMediaKind`. On `llm/stream` for routes that cannot carry the block, and at
  ingress for the rest.
- **Three reference implementations, now read in source rather than cited from a blog:**
  `zai-org/zcode` dispatches on `mediaType.startsWith("video/")` alone with no capability gate —
  two patches, 44 + 48 lines, covering both the Anthropic-style `{type:"video", source:{…}}` and the
  OpenAI-compatible `{type:"video_url", video_url:{url:"data:…"}}` encodings;
  `MiniMax-AI/minimax-code` (MIT, open source, pi-based) rides video through pi-ai's existing
  `ImageContent` with a `video/*` MIME and rewrites the serialized block at the payload boundary,
  recursing into nested tool-result content, deciding omission at ingress with typed reasons, and
  never setting `fps`; `openclaw` qualifies per route and substitutes visible text otherwise.
  Between them these answer every mechanism question the plan had.
- **Cost control the others skip:** MiniMax's OpenAPI exposes `MediaSource.fps` in [0.2, 5]
  (default 1) and `max_long_side_pixel`. Sampling is server-side, so that dial is the only lever
  on the token/latency trade-off, and no harness surveyed sets it.
- On the Pi side: the ecosystem says no quickly and without discussion — #8886 closed
  `not_planned` **9 seconds** after opening, #7326 **7 seconds**, and no release since v0.80.7 adds
  video. But `StreamOptions.onPayload` exists and runs as the last stop before the wire in all
  nine pi drivers, which is exactly the seam the extension needs. @weijiafu14's example in #2995
  is also worth reading: it is a working `pi2dsh` integration carrying a native image block through
  DSH's attachment service, which is evidence the delivery path is real today.

Question 1 (roadmap) still stands for the maintainers — as do the threads above, several of which
have sat unanswered for six weeks. I will post the plugin with the `dsh-plugin` topic either way,
and I would rather be told it is already planned. If the two walls are better addressed separately,
I am happy to split this: the declaration chain in #5837 is not specific to video and does not need
me in it.
