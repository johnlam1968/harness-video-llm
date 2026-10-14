# MiniMax-M3.1-Flash-Preview is missing from the generated catalog

*A bug report / small feature request. Drafted 2026-10-14, not yet posted.*

---

## Summary

`MiniMax-M3.1-Flash-Preview` works against MiniMax's Anthropic-compatible endpoint but does
not exist in Pi's generated model catalog, so it cannot be selected or used. I confirmed this
is the only thing standing between it and working — the endpoint, the protocol and the
reasoning controls all behave correctly.

## Why

MiniMax documents the model on its Anthropic SDK page and lists it as available through M Plan
and MiniMax Code. This looks like it should just be a catalog entry.

## Reproduction

`MiniMax-M3.1-Flash-Preview` does not appear in `pi --list-models`, and neither `minimax` nor
`minimax-cn` carries it. `MiniMax-M2.7`, `MiniMax-M2.7-highspeed` and `MiniMax-M3` are all
present.

It also cannot be added by hand. Per `docs/models.md`, `modelOverrides` changes metadata for
an **existing** built-in or extension-provided model, and *"Unknown override IDs are
ignored."* So this entry:

```json
{
  "modelOverrides": {
    "minimax-cn/MiniMax-M3.1-Flash-Preview": {
      "id": "MiniMax-M3.1-Flash-Preview",
      "reasoning": true,
      "input": ["text", "image"],
      "contextWindow": 1048576,
      "maxTokens": 512000
    }
  }
}
```

is silently discarded, and the model still does not appear. That also means a user cannot
self-serve this today.

## Expected

`MiniMax-M3.1-Flash-Preview` selectable under `minimax-cn`, with the metadata below.

## Verified specs

Taken from MiniMax's Anthropic SDK docs and cross-checked against the canonical
`sst/models.dev` record (`models/minimax/MiniMax-M3.1-Flash-Preview.toml`):

| field | value |
|---|---|
| context window | `1_000_000` |
| max output | `512_000` |
| input modalities | `text, image, video` |
| reasoning | **forced on** — `thinking: {type:"disabled"}` returns 400 |
| depth control | `output_config.effort` ∈ `low \| medium \| high \| xhigh \| max`, default `max`; `none` returns 400 |
| audio | **not supported** — `type:"audio"` returns `unsupported content type 'audio'` (error 2013) |

## Suggested change

In `packages/ai/scripts/generate-models.ts`, the MiniMax variant loop currently collapses
modality to `text`/`image` only. Widening it to preserve a model's declared input modalities
would fix this:

```ts
// current behaviour: binary text-or-image
input: m.modalities?.input?.includes("image") ? ["text", "image"] : ["text"],
```

becomes something that passes through `video` when the source declares it, plus the matching
change to `model-data.ts`'s modality validation.

## Prior art

- **#7326** — *"fix(ai): preserve MiniMax M3 video input capability"* — is essentially the same
  change for `MiniMax-M3`, closed 2026-07-30. Its approach looks right to me; I'm not aware of
  a reason it wasn't merged, so this may just need re-landing.
- `sst/models.dev` already publishes
  `providers/minimax-cn-coding-plan/models/MiniMax-M3.1-Flash-Preview.toml` with
  `reasoning_options` `low|medium|high|xhigh|max` and zero cost, so the source data exists.

## Note on availability

MiniMax's docs state the model is *"available only through M Plan and MiniMax Code for
now."* So the entry should carry whatever zero-cost pricing the token plan implies, matching
what `sst/models.dev` publishes, rather than pay-as-you-go rates.

## Working request

For anyone verifying by hand, this returns 200 and the model reads the frames correctly:

```
POST https://api.minimaxi.com/anthropic/v1/messages
Content-Type: application/json
Authorization: Bearer $MINIMAX_CN_API_KEY

{
  "model": "MiniMax-M3.1-Flash-Preview",
  "max_tokens": 300,
  "output_config": { "effort": "low" },
  "messages": [{
    "role": "user",
    "content": [
      { "type": "video",
        "source": { "type": "base64", "media_type": "video/mp4", "data": "<base64>" } },
      { "type": "text", "text": "What colour is the background, and what text is drawn on it?" }
    ]
  }]
}
```

> The background is bright blue, and the text reads "FRAME 85" in yellow.

## Environment

`@earendil-works/pi-coding-agent` 1.2.0 / `@earendil-works/pi-ai` 1.2.0, Linux.