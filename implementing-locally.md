# The Ledger Pattern — Implementing on a Local Model

*For the **implementer** wiring a small / local model (e.g. `Dolphin3:8b` via
Ollama or llama.cpp) into the Ledger Pattern. This document is **not** read by the
model. The model-facing recipe is [ledger-agent.md](ledger-agent.md); the full
reasoning is [Ledger-pattern.md](Ledger-pattern.md) and
[The-Ledger-Pattern-Paper.md](The-Ledger-Pattern-Paper.md).*

## The one idea: the model does no bookkeeping — your code does

A frontier model can read [ledger-agent.md](ledger-agent.md) and drive the pattern
itself. A small local model **cannot**, and no amount of simplifying the
instructions changes that — it's not a context-length problem you can trim your
way out of. An 8b model structurally cannot:

- **measure its own token usage** — it has no tokenizer running on itself, so it
  can't tell that a slice is "≤ 1/6 of the window";
- **carry a self-computed budget into behavior** — even handed the right number,
  it won't reliably slice the input to it while also doing the real work.

So don't ask it to. **Move every bit of sizing, slicing, queueing, and assembly
into the harness.** The model's only job is: receive one already-correctly-sized
prompt, return one result. That's the whole reason the pattern works on a laptop
8b that would otherwise choke on the raw input.

If you find yourself writing instructions to the model about token budgets, stop
— that logic belongs in your code.

## What the harness owns

Everything except the single-slice inference call:

1. **Tokenizer-based slicing.** Count tokens with the model's *actual* tokenizer
   (llama.cpp `/tokenize`, the Ollama API, or the GGUF's tokenizer) — not a
   characters÷4 guess. Slice the input to the chunk size from the table below.
2. **The on-disk ledger.** A temp/cache file recording `(pass, slice) → result`.
   This is your resume point and your inspection surface.
3. **A sequential queue.** One slice at a time. **Never** fan slices out in
   parallel to a single local model (see gotchas).
4. **Progress reporting.** Emit a signal every time a slice lands in the ledger.
5. **Assembly.** Consensus (extraction) or ordered concatenation (generation),
   per the fork — done in code over the ledger contents, not by the model.

## Sizing table (pre-computed — set a config value, don't derive one)

A local model has **one fixed context size**. Look it up once, set the chunk size,
done. There's no "scales with the model" to reason about at runtime.

| Context | Prompt budget | Chunk (input slice) | Response reserve | Input fraction |
| --- | --- | --- | --- | --- |
| 4k | 0.5k | 0.5k | 3k | 1/8 |
| 8k | 1k | 1k | 6k | 1/8 |
| 16k | 2k | 2k | 12k | 1/8 |
| 32k | 4k | 4k | 24k | 1/8 |
| *8k, strict cap* | 1.3k | 1.3k | 5.4k | 1/6 (the ceiling) |

The rule is prompt + chunk ≤ ~1/3 of the window, input slice ≤ ~1/6. The round
rows above sit at **1/8 — deliberately under the 1/6 cap**. Rounding the prompt
and chunk *down* is the safe direction: a smaller slice means less to triage (more
recall headroom) and more room for output. Round **up** past 1/6 and you re-invite
the triage the slicing exists to prevent. When in doubt, slice smaller.

`Prompt budget` is the ceiling for *your* system prompt + schema + examples — if
yours is bigger, steal from the response reserve, not from keeping the chunk
small. The chunk size is the value you hand the slicer.

## The loop, in harness terms

```text
CHUNK   = table[context_size].chunk          # e.g. 1000 tokens for an 8k model
OVERLAP = round(CHUNK * 0.15)                # 10–20%
N       = 3 if task == "extraction" else 1   # generation: 1, or N best-of candidates

slices  = tokenizer_slice(input, size=CHUNK, overlap=OVERLAP)   # YOUR code, real tokenizer

for pass in 1..N:
    for i, slice in enumerate(slices):
        if ledger.has(pass, i): continue                 # resume after a crash/cancel
        result = ollama_generate(system_prompt, slice)   # the ONLY model call
        ledger.write(pass, i, result)
        report_progress(done / total)                    # update the user every slice

return assemble(ledger, task)                            # consensus OR concat, in code
```

`assemble()` is where the extraction/generation fork lives — and it is your code's
responsibility, never the model's:

- **extraction** → normalize to canonical identity, collapse within-pass
  span-overlap duplicates, then keep items meeting `floor(N/2)+1` consensus with
  consistent attribution. (Normalize *before* counting — see
  [ledger-agent.md](ledger-agent.md) invariant 6.)
- **generation** → read slices in order, concatenate, reconcile each seam
  (drop the restated overlap, check continuity). No consensus.

## You'll know it's working when — verify the slicer, don't trust it

You can't read the model's mind, but you don't need to: the harness is the part
that can be wrong, and it's fully observable. Watch it work.

- **Log the token count of every slice *before* you send it.** This is the load-
  bearing check. If any slice exceeds ~1/6 of the context window (or your
  `CHUNK + OVERLAP`), your **slicer is wrong, not the model** — fix the code, not
  the prompt. A clean run shows every slice at or under the chunk size, period.
- **The ledger file grows by one entry per slice, in order.** If it jumps,
  stalls, or rewrites entries, your queue or resume key is off.
- **Progress advances monotonically to 100%.** Stuck percentage = a slice hung
  or the queue isn't draining.
- **Kill it mid-run and restart — it resumes, doesn't restart.** If it
  reprocesses completed slices, your `ledger.has(pass, i)` check is broken.
- **Slice count ≈ `ceil(input_tokens / (CHUNK − OVERLAP))`.** Way more or fewer
  slices than that means the slicer is mis-sized.

If the output is still lossy *after* every slice logs under the cap, that's a
model/prompt issue to chase separately — but you've now ruled out the harness,
which is the half you control.

## Local-specific gotchas (amplified versions of the general rules)

- **Parallel is a trap, harder here.** Against one local model, firing slices
  concurrently doesn't speed anything up — it just multiplies the working set
  until you exhaust RAM/VRAM and the whole thing thrashes to a crawl. Sequential
  is not a nicety; it's what keeps the box alive. (Multiple *machines* is a
  separate, out-of-scope optimization.)
- **A long run looks like a stall.** An 8b grinding 30 slices ×3 passes can take
  minutes with no visible output. Without progress reporting the user concludes
  it hung and kills it. The percentage-per-slice update is the difference between
  "working" and "broken."
- **The ledger earns its keep here.** High slice counts (small window → many
  slices) mean long jobs, and long jobs get interrupted. On-disk + resume turns a
  crash at slice 14/30 into a 16-slice finish, not a restart.
- **Count tokens, don't estimate them.** characters÷4 drifts per language and
  tokenizer; on a tight 8k budget that drift is enough to push a "1k" slice over
  the cap. Use the real tokenizer.

## What this does *not* try to fix

This makes a local model usable *within* the pattern. It does not make the 8b
smarter at the extraction or generation itself — it removes the orchestration
burden so the model spends its limited capability on the one slice in front of it,
which is the most it can reliably do.
