# The Ledger Pattern

A design pattern for reliably processing large text inputs through a
context-limited LLM. Instead of handing a model one giant prompt and hoping,
you **window** the input, process one slice at a time against a durable on-disk
**ledger**, run multiple passes, and combine them by **consensus** — not union.
Recall and accuracy go up; the job becomes resumable and inspectable.

> **TL;DR** — Window the input. Process one slice at a time against a durable
> ledger. Run multiple passes and aggregate by consensus. Do it asynchronously
> off a queue so the UI never blocks.

## The shape

```text
            ┌─────────────────────────────────────────────┐
  large     │  Window      slice the input (≤ ~1/6 of the  │
  input ───▶│              context window, 10–20% overlap) │
            ├─────────────────────────────────────────────┤
            │  Process     one slice at a time, off a      │
            │  ×N passes   background queue → on-disk ledger│  ◀── resumable
            ├─────────────────────────────────────────────┤
            │  Assemble    span-dedup, then keep by         │
            │              consensus (majority of passes)  │
            └─────────────────────────────────────────────┘
                                   │
                                   ▼
                              merged result
```

*Extraction is shown above. For **generation** (long output — prose, summaries),
the **Assemble** step forks: concatenate the slices in order and reconcile each
seam — best-of selection, **not** consensus. One pattern, two assembly modes.*

## Why

A single large prompt makes a model *triage*: it returns the most salient
results and silently drops the quiet ones, mis-attributes details, or truncates.
That isn't fixed by a better prompt — it's a property of asking a model to attend
to too much at once. Keeping each slice small (≤ ~1/6 of the context window) and
aggregating multiple passes by consensus recovers the quiet facts and discards
the one-off errors.

This holds even when the input *fits* — triage is a property of attention, not
just context — so the pattern also pays off as a pure recall amplifier whenever
detail matters more than speed. And the input needn't be text: tiling a large
image through the same loop surfaces details a whole-image pass misses.

That's the *extraction* path. The same windowing also drives *generation* (long
output): there the passes are best-of candidates and the slices are concatenated
in order, not voted on — one pattern, two assembly modes.

## The docs

| Document | What it is | Read it if you're… |
| --- | --- | --- |
| **[Ledger-pattern.md](Ledger-pattern.md)** | The formal spec — intent, structure, the context-allocation rule, the two assembly branches (consensus for extraction, ordered concatenation for generation), the overlap/consensus sequencing caveat, async execution. | …learning the pattern in full. |
| **[The-Ledger-Pattern-Paper.md](The-Ledger-Pattern-Paper.md)** | The original essay it was distilled from — the story, the motivation, the GoF framing. | …after the *why*, in prose. |
| **[ledger-agent.md](ledger-agent.md)** | The self-contained drop-in recipe — trigger, parameter defaults, algorithm, invariants — for an LLM/agent to apply mid-task. | …pointing an LLM at it (frontier models can self-orchestrate from this). |
| **[implementing-locally.md](implementing-locally.md)** | Implementer guide for wiring a small / local model into a harness: the model does no bookkeeping, the code owns sizing/slicing/queue/ledger/assembly. | …building the harness around a local 8b. |

There's also a Claude Code skill at `.claude/skills/ledger-pattern/` that
auto-invokes the recipe on matching work.

## License

Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE).
Use it, adapt it, build on it — just credit *The Ledger Pattern* and its author.
