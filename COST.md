# Cost estimate

Two artifacts, priced separately, for a go/no-go decision. Figures are engineering hours for
one developer already familiar with both codebases. Nothing here has been built yet.

Last updated 2026-10-14.

---

## 1. What gets built

**A — `pi-video-modality`**, a pi extension. Registers a sibling provider
`minimax-cn-video` that delegates to pi's built-in `anthropic-messages` stream and rewrites
video blocks in an `onPayload` hook. Ships a `read_video` tool as the ingress path. Nothing in
pi's shipped catalog is shadowed; `pi.unregisterProvider()` reverts.

**B — a DSH plugin**, a `LlmAdapter` subclass that adds video end-to-end in DSH's own
vocabulary. Independent of A.

---

## 2. Cost breakdown

### A — pi extension

| # | item | h | confidence |
|---|---|---|---|
| A1 | Provider registration + model metadata (`minimax-cn-video`, M3/M3.1, `input` cast) | 3 | high — shape copied from pi's own example |
| A2 | `streamSimple` delegate + `onPayload` video block rewrite + size guard | 5 | med-high |
| A3 | `read_video` tool (fs read, base64, mime sniff, path + size limits) | 4 | high |
| A4 | Visible omission when the selected model cannot take video | 5 | **low** |
| A5 | Test matrix — pi mandates 11 categories | 12 | med |
| A6 | Packaging, README, npm publish (`pi-package` keyword) | 4 | high |
| | **A total** | **33 h ≈ 4 days** | |

**A4 is the one open question.** `downgradeUnsupportedImages` runs inside `pi-ai` *before* a
provider is chosen, so an extension may have no seam to influence it. If there is none, the
fallback is to accept pi's behaviour: a video sent to a non-video model goes out as
`type:"video"` and returns a **400** from the provider. That is loud rather than silent, which
is acceptable — hence low confidence but bounded impact.

### B — DSH plugin

| # | item | h | confidence |
|---|---|---|---|
| B1 | Extend `ModelModalityMap` via declaration merging + verify it survives the plugin loader | 4 | med |
| B2 | Video block → `ImageContent{mimeType:"video/*"}` translation in a new `LlmAdapter` subclass | 10 | med |
| B3 | `onPayload` rewrite on DSH's side | 6 | med |
| B4 | Video attachment plumbing (`RequestVideoAttachment`, access resolver, limits) | 8 | **low-med** |
| B5 | Plugin contract tests + one live DSH session | 8 | med |
| B6 | Packaging, dsh.so record, `dsh-plugin` topic | 4 | high |
| | **B total** | **~40 h ≈ 5 days** | |

**The good news on B3**, found after the first draft of this estimate: DSH's adapter *calls*
pi's `streamSimple` (`packages/llm/llm-pi-ai/src/adapter.ts:402`), and `SimpleStreamOptions.onPayload`
is a **caller-supplied** hook. DSH is therefore the caller and the same one-hook rewrite works
without touching pi at all. The earlier assumption that B needed a patch of `llm-pi-ai` is
wrong.

`PiAiAdapter extends LlmAdapter` (`adapter.ts:230`) is a documented base class, so B2 is a
subclass rather than a fork. That is the difference between a plugin and a permanent
maintenance burden.

---

## 3. Totals

| scope | hours | days |
|---|---|---|
| A only | 33 | 4 |
| B only | 40 | 5 |
| **A + B** | **~73** | **9–10** |
| Lean cut (A-min + B-min, see §5) | ~35 | 4–5 |

For comparison, the plan this replaces specified a custom `streamSimple` owning its own HTTP.
Delegating to pi's built-in implementation instead is what took A from an estimated ~55 h to
33 h — `custom-provider.md` would otherwise have required reimplementing six stream contracts,
and we would have frozen a copy of a driver that upstream keeps fixing.

---

## 4. Effectiveness

### What native modality buys over what already exists

The DSH registry already has working video plugins, all verified at L4–L5:
`dsh-video-lens` (★114), `dsh-video-understand`, `dsh-mmx-bridge`, `dsh-mimo-agent-tools`,
`dsh-guide-dog`. **All are tool-based** — the model calls `video_understand(...)` and reads a
text summary.

Native modality differs in four ways:

1. **One hop instead of two.** The model sees the video rather than a description of it.
2. **Fidelity.** A summarizing tool decides what survives; the modality passes the frames.
3. **Transcript persistence.** The video stays in the session, so *"what was on that button
   in the clip?"* re-reads the actual video instead of a summary written once, early.
4. **Addressability.** "Frame 40", "the second scene" become referents.

**Point 3 decides it.** For *"watch this once and tell me what's in it"* the existing tools
are already sufficient and this work is worth close to nothing. For iterative work on moving
images — game and animation development, UI motion, CI repro clips, reviewing a bug as a clip —
the summary hop destroys exactly the detail the second and third question need.

If the driving use case is the first one, stop here: install `dsh-video-lens` and skip the
entire estimate.

### Probability the work is obsoleted

| | odds within ~6 months | consequence |
|---|---|---|
| **A (pi)** | roughly even | pi #3200 is open with maintainer engagement (*"this will take a while i'm afraid"*), and three other implementations are in flight — turisanapo's PR, Jeomon/Tau, toolittlecakes' `pi-gemini-audio-read`, which already uses the identical payload-rewrite technique |
| **B (DSH)** | low | would require DSH to extend `ModelModalityMap` and `userContent` in core; no external party will do that for us |

A being obsoleted is not wasted, but it is not ours either. The *measurement* — that the block
`type` must change, not just the mime type — and the *vendor survey* are already banked in
pi discussion #10915 and carry value independent of either artifact.

---

## 5. Cheap cuts, if the estimate is too high

| cut | saves | cost |
|---|---|---|
| Drop A4 (visible omission), accept the provider 400 | 5 h | degraded error message instead of a good one |
| Trim A5 to the essentials — text, video, one tool call, usage, abort | 7 h | not upstreamable-grade, but usable |
| Drop B4 by accepting inline base64 with no durable attachment record | 8 h | video not replayable across compaction |
| **Lean total** | **~35 h** | 4–5 days |

The lean build is the version I would actually ship for personal use.

---

## 6. Zero-cost items already banked

- pi discussion [#10915](https://github.com/earendil-works/pi/discussions/10915) — the
  measurement, the vendor survey, the ZCode reference
- DSH discussion [#9647](https://github.com/deepseek-ai/deepseek-harness/discussions/9647) —
  the modality seam, the silent-drop finding
- pi issue [#10916](https://github.com/earendil-works/pi/issues/10916) — the M3.1 catalog gap.
  **Auto-closed** by the new-contributor bot within seconds of posting, as all issues from this
  account will be. Maintainers are said to review auto-closed issues daily; if reopened, the
  model appears in **both** Pi and DSH for nothing.

---

## 7. The decision, stated plainly

A's audience is pi's 114,906-star user base. **That is not John.** If the goal is video in the
harness John actually works in, A is ~33 h spent on other people's behalf, in a repository
where three other people are already building it.

B is the durable half, and the one whose gap only DSH can close.

So: **skip A, do B** — unless the point is to contribute upstream, in which case A is the
contribution and should be scoped as a PR behind an `lgtm`, not as a published package.
