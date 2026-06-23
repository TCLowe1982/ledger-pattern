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

## Why

A single large prompt makes a model *triage*: it returns the most salient
results and silently drops the quiet ones, mis-attributes details, or truncates.
That isn't fixed by a better prompt — it's a property of asking a model to attend
to too much at once. Keeping each slice small (≤ ~1/6 of the context window) and
aggregating multiple passes by consensus recovers the quiet facts and discards
the one-off errors.

## Read the full pattern

The complete write-up — intent, structure, the context-allocation rule,
consensus-vs-union discipline, the overlap/consensus sequencing caveat, async
execution, and the reference implementation — lives in:

**➡️ [Ledger-pattern.md](Ledger-pattern.md)**

**Implementing it (or pointing an LLM/agent at it)?** The self-contained recipe —
trigger, parameter defaults, algorithm, and invariants — is in
**➡️ [ledger-agent.md](ledger-agent.md)**.

## License

Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE).
Use it, adapt it, build on it — just credit *The Ledger Pattern* and its author.
