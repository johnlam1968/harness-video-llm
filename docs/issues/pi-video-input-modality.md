# Which labs have video-native LLMs, and what does it take to use one in Pi?

*A feature request. Drafted 2026-10-14, not yet posted.*

---

## Summary

Pi declares exactly two input modalities — `text` and `image`. Several mainstream models
accept **video** as input. At the moment, none of them can be used through Pi.

I'd like to propose adding `video` as a third input modality, and I think this is a
reasonable near-term ask given the table below.

## What already works

Video-capable models are widely available and cheap. Concretely, I ran this against
MiniMax's Anthropic-compatible endpoint this week:

```
POST https://api.minimaxi.com/anthropic/v1/messages
model: MiniMax-M3.1-Flash-Preview
content: [ { type: "video", source: { type: "base64", media_type: "video/mp4", data: … } },
           { type: "text",  text: "What colour is the background, and what text is drawn on it?" } ]
```

> **HTTP 200** — *"The background is bright blue, and the text reads **"FRAME 85"** in yellow."*

The test clip was a 4-second 320×240 mp4 with the frame number rendered into the picture, so
the answer could not have come from metadata. The model is genuinely reading the frames.

## Which models already accept video

| lab | model | declared input modalities |
|---|---|---|
| Google | Gemini 4-Argon | `text, image, **video**, audio, pdf` |
| Google | Gemini 3-Pro / 3.1-Pro / 3-Flash | `text, image, **video**, audio, pdf` |
| Xiaomi | MiMo-V2.6-Pro | `text, image, audio, **video**` |
| MiniMax | M3, M3.1-Flash-Preview | `text, image, **video**` |
| Zhipu | GLM-4.6V | `text, image, **video**` |
| Moonshot | Kimi K3 | `text, image, **video**` |
| ByteDance | Seed 2.1 Pro | `text, image, **video**` |
| Anthropic | Claude Opus 5.5, Sonnet 5.5 | `text, image, pdf` |
| OpenAI | GPT-4o, GPT-5.6, GPT-6.1 | `text, image, pdf` |

Source: the `sst/models.dev` canonical records (`models/<lab>/<model>.toml`,
`[modalities] input`), cross-checked against each vendor's own docs. 552 records across 69
labs were surveyed.

I'd highlight one thing: **video input is present at every Chinese frontier lab I checked and
absent at both Western ones.** Anthropic's vision documentation describes only `image` content
blocks, and OpenAI's API docs have an images-and-vision guide with no video counterpart. So
this isn't "Pi lagging the ecosystem" — it's a capability the Western frontier has not adopted
while the Chinese labs ship it as a headline feature. I'd expect it to spread, and it's worth
Pi being able to use it when it does.

## The three places this is blocked

**1. The modality vocabulary.** In `packages/ai/src/providers/model-schema.ts`:

```ts
export const MODEL_INPUT_MODALITIES = ["text", "image"] as const;
```

This is enforced at runtime, not just at the type level. `models.json` is validated by a
compiled TypeBox schema on load, and `input` is constrained to the same two values:

```json
"input": { "type": "array", "items": { "enum": ["text", "image"] } }
```

I confirmed this against the built CLI with an isolated config:

```
$ pi --list-models
Invalid models.json schema:
  - providers.volcengine.models.0.input.1: must be equal to one of the allowed values
```

(A deliberately invalid value like `"banana"` produces the same error, so the validator is
genuinely running.)

**2. The Anthropic driver.** In `packages/ai/src/api/anthropic-messages.ts`, content blocks
are serialised with `type` hardcoded and only `media_type` passed through:

```ts
media_type: block.mimeType as "image/jpeg" | "image/png" | "image/gif" | "image/webp",
```

So even if a video block were representable, it could not reach the wire as
`{"type":"video","source":{…}}`.

**3. `models.json` cannot create a new modality.** This is arguably fine — but worth stating
because it bounds what a workaround can do. My understanding of `docs/custom-provider.md` is
that it routes this correctly: *"Implement an unsupported wire protocol → a provider with
`stream` or `streamSimple`."* A provider extension can register its own `streamSimple` and
declare `input: ["text","image","video"]` as a literal in its own models list, which does work
today. That's my reason for filing this rather than just shipping that workaround: the
workaround needs a separate HTTP path per provider, and every user of every affected provider
has to write one.

## What I'm asking for

Open to whatever shape maintainers prefer. The smallest version that would close this for
users, in rough order of scope:

1. **Add `video` to `MODEL_INPUT_MODALITIES`** and relax the `models.json` schema. Small.
2. **Teach the content union and the Anthropic driver about video** — a `VideoContent`
   variant and a `type:"video"` case in the block serialiser, mirroring how `image` is
   handled today. This is the bulk of the work.
3. **Add an attachment path**, mirroring `inputLimits.images` for video (size, duration,
   frames). Optional if you prefer modalities to travel by URL first, which is cheaper.

## Prior art and non-goals

- **#3200** ("Support video/audio content in prompt command") looks closely related and is
  still open. If this belongs there instead, please close this one — I don't want to
  fragment it.
- **#7326** ("fix(ai): preserve MiniMax M3 video input capability") appears to have been a
  narrower version of this. It was closed; I'm resurfacing the capability question rather
  than the patch.
- **#8886** ("Add first-class video input support") — I see this was closed as `not-planned`.
  This issue is the same request with evidence and a vendor survey attached, in case the
  priority assessment was made without them. If the answer is still no, that's fine — I'd
  rather have a clear answer than an open one.

I'm not proposing anything about **audio**: MiniMax rejects a standalone audio content block
outright (`unsupported content type 'audio'`, error 2013), and audio-to-text belongs to a
separate transcription endpoint rather than to the chat route.

Happy to test against a real model if that's useful — I can provide the exact fixture and
the request/response transcript.

## Environment

`@earendil-works/pi-coding-agent` 1.2.0 / `@earendil-works/pi-ai` 1.2.0, Linux. Verified
against `main` at commit `d743267388` era for the pi repo source references above.