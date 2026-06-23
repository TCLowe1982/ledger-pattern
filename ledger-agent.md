# The Ledger Pattern — Agent Instructions

*Self-contained recipe for an LLM/agent applying [The Ledger Pattern](Ledger-pattern.md)
within its own work. This is the actionable core; the full write-up explains
**why** each step is shaped the way it is — read it when a choice is unclear.*

## Reach for the pattern when

Either is true:

- a large text input produced a lossy / triaged / mis-attributed result, OR
- a single call would put more than ~1/3 of the model's context in the prompt.

## First, name the branch: extraction or generation?

The pattern has **two task shapes**, and they diverge at assembly. Decide which
you are doing *before* you window anything — the choice changes what `overlap`
is for, what `passes` mean, and how you assemble. Misapplying the extraction
assembly (consensus) to a generation task — or vice versa — is the failure this
fork exists to prevent.

| | **Extraction / classification** | **Generation** |
| --- | --- | --- |
| Examples | fact capture, beat analysis, scene recap, digest, tagging | arc rendering, scene prose, long-form summary *text* |
| Output size | tiny (a few items) | large (the 2/3 reserve is literally the output) |
| Why slice small | **recall** — a big input makes the model triage and drop quiet items | **token budget** — a big output needs the room |
| What `overlap` buys | edge-fact recall: a fact straddling the cut isn't guillotined | narrative continuity: the seam between slices reads smoothly |
| What `passes` buy | variance reduction, voted by **consensus** | optional **best-of**: render N variants, *select* the best — never vote |
| How you **assemble** | normalize → collapse within-pass dupes → keep by **consensus** | **concatenate in slice order**, reconcile each seam — no consensus, no dedup-by-recurrence |

If you're extracting, follow the **extraction** algorithm and invariants below.
If you're generating, follow the **generation** ones. WINDOW and PROCESS are
shared; only ASSEMBLE forks.

## Parameters (defaults)

| Parameter | Default | Notes |
| --- | --- | --- |
| `slice_size` | ≤ ~1/6 of the context window | Both branches. Token budget, **not** a fixed chunk count — scales with the model. Prompt + slice together ≤ ~1/3. |
| `overlap` | 10–20% of `slice_size` | Both branches, **different purpose**: extraction → edge-fact recall at the cut; generation → narrative continuity across the seam. |
| `passes` (N) | extraction: 3 · generation: 1 | Extraction: independent runs combined by consensus. Generation: a single pass by default; >1 means best-of candidate *variants you select among*, not vote. |
| `consensus_threshold` | `floor(N/2)+1` | **Extraction only.** N/A to generation — generation has no consensus step. |
| `execution` | async, off a queue | Both branches. Return a job id immediately; process slices on a background worker; alert on done. |

## Algorithm

### Shared: WINDOW + PROCESS

```text
1. WINDOW
   slices = split(input, size=slice_size, overlap=overlap)   # by tokens, not count

2. PROCESS  (repeat for pass in 1..N)
   for slice in slices:
       if ledger.has(pass, slice): continue                  # resume: skip done work
       result = llm(prompt + slice)                           # one slice at a time
       ledger.write(pass, slice, result)                     # durable, on disk
       emit_progress()
   # generation: thread the tail of the previous slice's OUTPUT into the prompt
   # so the next slice continues the thread — that is what the overlap is for here.
```

### ASSEMBLE — extraction / classification

```text
3a. ASSEMBLE (extraction)
   items = ledger.read_all()

   items = normalize(items)                       # canonical identity FIRST — embedding cluster
                                                  # or cheap LLM pass. This is REQUIRED before any
                                                  # dedup: a reworded overlap won't match on raw
                                                  # text, so you cannot span-dedup byte-wise.

   for pass in 1..N:                              # collapse within-pass span-overlap duplicates:
       collapse(items, where=same_identity & same_pass, to=one_occurrence)
                                                  # one fact from two adjacent slices of ONE pass
                                                  # is one occurrence, not two

   keep = [x for x in distinct(items)             # THEN count across passes
           if occurrences_across_passes(x) >= consensus_threshold
           and attribution_is_consistent(x)]      # majority subject on conflict, else drop
   return keep
```

### ASSEMBLE — generation

```text
3b. ASSEMBLE (generation)
   parts = ledger.read_all_in_slice_order()       # NO consensus, NO keep-what-recurs

   output = ""
   for part in parts:                             # stitch in sequence
       output = splice(output, part, at_seam=reconcile)
                                                  # at each seam, reconcile the overlap region:
                                                  # drop the restated span, smooth the transition,
                                                  # check continuity (names, tense, open threads)
   return output

   # multi-pass (best-of): render N independent candidates, SELECT the best with a
   # judge — do not merge or vote them token-by-token.
```

## Invariants

Each is tagged with the branch it governs.

1. **[both] Name the branch first.** Everything downstream — assembly, and the
   meaning of `overlap` and `passes` — forks on extraction vs. generation.
   Defaulting silently to extraction is the unstated assumption the pattern
   exists to catch.
2. **[both]** Size slices by **token budget**, never by a fixed chunk count.
3. **[both]** The ledger is **on disk** and execution is **async** so a
   backgrounded job that dies mid-run resumes from the ledger instead of
   restarting.
4. **[extraction]** The slice cap holds **even when the output is tiny**: the cap
   is for *recall*, because a model handed too much input triages and drops quiet
   facts — not just to leave room for output.
5. **[extraction]** Aggregate by **consensus in a normalized space**, never by
   raw union. Union preserves one-off errors and near-duplicates as if they were
   signal; more passes then produce *more* noise.
6. **[extraction] Normalize first, then collapse within-pass duplicates, then
   count across passes.** You cannot span-dedup on raw text — the overlap region
   is not byte-identical once a pass rewords it, so normalization is what
   *recognizes* the duplicate. Counting before collapsing lets one overlapped
   fact read as two hits and falsely clear the consensus bar.
7. **[generation]** Assemble by **ordered concatenation**, reconciling each seam
   for continuity and dropping the overlap restatement. Overlap here is a
   continuity buffer, not recall insurance. Never consensus-filter prose — you
   stitch chapters in sequence, you don't keep the ones that recur.
