# The Ledger Pattern

*A design pattern for reliably processing large text inputs through a
context-limited LLM. Named by TC; first formalized during fact-extraction work on
an LLM memory system, where a fact extracted cleanly from a focused window
vanished when the same model was handed a larger one.*

> **TL;DR** — Don't hand a model one giant prompt and hope. Window the input,
> process one slice at a time against a durable on-disk ledger, run multiple
> passes, and combine them by **consensus** (not union). Recall and accuracy go
> up; the job becomes resumable and inspectable.

## Intent

Process a large text input through an LLM **without** degrading the result by
overstuffing the context. Window the input, process one slice at a time against
a durable per-input ledger, and assemble the slices at the end.

## Motivation

A single large prompt makes a model *triage*: it returns the few most salient
results and silently drops the quiet ones, mis-attributes details, or truncates.
This is not fixed by a better prompt — it's a property of asking a model to
attend to too much at once.

> Live example: the fact "Mari's D&D class is a Pact of the Tome Warlock" was
> dropped from a 10-chunk extraction window and recovered, by the *same model*,
> from a focused window. The model could always do it — it just couldn't do it
> while triaging ten chunks of dense prose.

## Structure

1. **Ledger** — a durable, per-input record (a temp file on disk) of slice →
   result. Makes the run resumable (a crash keeps completed slices) and
   inspectable (you can read what each slice produced before assembling).
2. **Window** — slice the input into chunks sized by the model's context budget
   (see the allocation rule), not a fixed count, with a **10–20% overlap between
   adjacent slices** so a fact straddling a boundary isn't guillotined by a
   mechanical cut. The overlap is recall insurance at the seams; it is cheap
   relative to silently losing an edge fact.
3. **Process** — send one slice at a time. With async execution (see *Execution*
   below), each slice is a queued unit of work, not a blocking call.
4. **Assemble** — merge and de-duplicate the per-slice results at the end.
   **Span-overlap dedup runs first, before consensus counting** (see the
   sequencing caveat).

## The context allocation rule (prompt integrity)

Partition the model's context window in thirds-ish, and keep the input slice
small:

| Portion | Share | Purpose |
| --- | --- | --- |
| Prompt / instructions | ~1/6 | system prompt, schema, examples |
| Input chunk | ~1/6 | the slice being processed |
| Remainder | ~2/3 | the answer (see task-dependency below) |

The load-bearing constraint is the same for every task: **the input slice should
be ≤ ~1/6 of the context window, and prompt + chunk together ≤ ~1/3.** It
**scales with the model**: a 200k-context model takes far larger slices than an
8k local model — sizing by a fixed chunk count is the anti-pattern.

### What the 2/3 is *for* depends on the task

The reservation is real but its purpose flips by task type, and conflating them
causes the classic regression below:

- **Output-heavy tasks** (generation: summary rendering, prose synthesis) — the
  2/3 is literally the **output**. The hard token budget binds: a big answer
  needs room, which is *why* the input must stay small.
- **Extraction / classification** (fact capture, beat analysis, digest) — the
  output is tiny (a few JSON facts), so the 2/3 is **not** output; it is mostly
  free space. The input still stays ≤ ~1/6, but for a different reason:
  **recall**. A model handed a large input *triages* — it returns the salient
  results and silently drops the quiet ones — even when everything fits in
  context. That is an attention limit, not a token-budget limit.

> The trap: "extraction output is tiny, so I can make the input huge." No. The
> Warlock fact was lost in a 10-chunk window that fit comfortably in context —
> the failure was triage, not truncation. The ≤ ~1/6 input cap holds for
> extraction *because of recall*, independent of how small the output is.

So the cap is the invariant; the remainder is output-reserve when you generate
and recall-slack when you extract.

## Assembly discipline: aggregate by consensus, not union

The assemble step (4) is not free, and it has its own anti-pattern. When you
process the same input multiple times to beat single-pass variance, **naive
union — OR every pass's output together — is the wrong way to combine them.**
Union keeps near-duplicates (the same item reworded), overwhelms any downstream
filter, and, worst, preserves a one-off error from a single pass as if it were
signal.

> Worked example (this is how the corollary was found — by violating it): a
> 3-pass fact extraction produced 74 candidates, the durability judge kept ~71
> (its filter overwhelmed), and a mis-attribution present in only **1 of 3**
> passes survived the union — "Priya's D&D class is a Pact of the Tome Warlock"
> (it is Mari's). More passes produced *more* noise, the opposite of the intent.

The discipline is **consensus in a normalized space**:

1. **Normalize before counting.** "Mari's class is Pact of the Tome Warlock" and
   "Mari is a Pact of the Tome Warlock" are the same fact; counted raw they are
   two singletons and neither reaches consensus. Collapse to a canonical identity
   first — embedding-similarity clustering, or a cheap LLM normalization pass.
2. **Count occurrences in normalized space.**
3. **Keep on majority.** Keep an item only if it recurs in `consensus_count ≥
   floor(N/2)+1` passes **with consistent attribution**. A one-off (the 1/3
   mis-attribution) drops out; a real item (Warlock-as-Mari, 3/3) stays. On
   conflicting attribution across passes, take the majority subject or drop.

Done this way, **more passes = higher confidence**. Union does the reverse — so
the multi-pass *count* and the multi-pass *aggregation* are two separate
disciplines, and skipping the second negates the first.

## Overlap ↔ consensus interaction (the sequencing caveat)

Overlap and the consensus discipline talk to each other, and order matters. An
overlapped fact now appears in two adjacent slices *within a single pass*. If you
count occurrences toward `floor(N/2)+1` consensus **raw**, that one fact from one
region of text reads as two hits — it can falsely clear the consensus bar off a
single pass.

So **normalize-and-collapse must run before you count passes, and it must dedup
within-pass span-overlap first.** Collapse adjacent-slice duplicates of the same
fact to one occurrence per pass, *then* count across passes. The normalization
machinery is already required for consensus (above); overlap just makes running
it non-optional.

## Execution: process asynchronously, off a queue

Multi-pass over dozens of slices is a lot of LLM calls. Running them
synchronously and blocking the UI is the cardinal sin — a novice pointing a local
model at a long transcript will freeze the app for minutes and conclude the
pattern is broken.

Model it as a restaurant order ticket, not a customer standing at the counter:

1. **Fire the job, hand back a receipt.** The user gets a job id immediately and
   keeps chatting / working.
2. **Process slices one at a time off a background queue.** Emit progress as
   slices complete.
3. **Alert on done.** Notify when assembly is finished; surface the result then.

This stacks with the ledger by design: the on-disk ledger is the resumability
layer, so a backgrounded job that dies at slice 14 of 30 resumes from the ledger
instead of restarting. The async queue and the on-disk ledger were built for each
other — the ledger is on disk *because* the work is long-running and backgrounded.

> Particularly load-bearing for local models. An 8k-context model takes many
> small slices to cover a long input — exactly the case where call count is
> highest and a synchronous loop hangs longest. The async-with-ledger shape is
> what keeps the pattern usable on a laptop, not just against a 200k remote model.

## Applicability

Use it whenever:

- a large text input gave a lossy/triaged/mis-attributed result, OR
- a single call would put more than ~1/3 of the model's context in the prompt.

Especially: extraction/classification over long transcripts (fact capture, beat
analysis, scene recap, digest, summary rendering).

## Consequences

- **+** Higher recall and accuracy on large inputs (no triage).
- **+** Resumable and inspectable (the ledger).
- **+** Async + ledger keeps the UI responsive and the job resumable on long /
  local-model runs (job id, progress, done alert).
- **−** More LLM calls (bounded, and cheap relative to losing the data); overlap
  adds redundant slice coverage (also bounded — the span-dedup step pays it back).
- **−** Assembly must de-duplicate across slices (a fact restated in two windows),
  and span-overlap dedup must run *before* consensus counting or it inflates the
  count.

## Reference implementation

The pattern was extracted from [Marinara Extender](https://github.com/TCLowe1982/Marinara-Extender),
a persistent-memory system for LLM characters, where parts of it run in
production:

- **Incremental, resumable processing** — an import pipeline persists each unit
  of work as it completes, so a cancel or crash keeps everything done so far
  (the ledger, as resumability).
- **Windowed backfill / repair** — batch scripts window the input, offer a
  dry-run preview, then assemble.
- **Context-sized fact extraction** — a fact pass windows long scenes; an early
  fixed-chunk-count version was a concrete instance of the allocation rule — and
  of its failure mode (the dropped Warlock fact above).

## License

Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE).
Use it, adapt it, build on it — just credit *The Ledger Pattern* and its author.

## Acknowledgments

*The pattern was named and first formalized by TC out of the Marinara Extender
work. Both refinements — sliding-window overlap and async-with-ledger — surfaced
from a user-experience review by [Ken](https://github.com/Mr-Ken-1102)
(editor-in-chief) with Gemini Pro,
stress-testing the pattern from the novice / local-model angle the original
draft, written from inside the implementation, was blind to. This writeup — the
standalone adaptation and the prose throughout — was drafted by Claude (Opus 4.8,
via Claude Code).*
