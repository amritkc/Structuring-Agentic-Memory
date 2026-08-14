# Does the model answer from the memory I gave it, or from what it already knew?

**Amrit Khadka** · M.Sc. Computer Science, TU Darmstadt

A small experiment testing whether a language model answers from memory placed in
its context or from knowledge absorbed during training — and whether *how* that
memory is delivered changes the balance.

> **The headline result is not the one I set out to measure.** I designed this to
> test whether in-context memory beats pre-training knowledge, and found that for
> these models it isn't a contest: the prior never surfaced once in 680 responses.
> What separates the conditions instead is *retrieval* — whether the model can
> locate the right fact when similar-sounding ones compete, carry a fact through a
> reasoning step, and tell a current memory entry from a superseded one. That, and
> the fact that my first design measured my own instruction rather than the model,
> are what this report is about.

---

## What I built

A synthetic memory of 19 short facts about a fictional research institute:

| kind | count | example |
|---|---|---|
| fictional | 6 | *The Cartography Unit is on the fourth floor of the Halden Building.* |
| override | 3 | *In this setting, the capital of Australia is Sydney.* |
| bridge | 2 | *Dr. Voss's field office is located in the capital of Australia.* |
| stale / current pairs | 8 | *Kestrel-7 is stationed at Northfield* vs *[2031-03-14] relocated to Bracken* |

The **bridge** facts matter: they refer to an overridden fact without naming it, so
answering them requires the override to survive a reasoning hop rather than just be
copied. The **stale/current** pairs are the situation a real agent memory ends up in
after a few months.

On top of that, 20 queries across eight difficulty tiers, with the expected answer
for each written down **before** running anything.

### Two things varied

**How much memory the model got:**

- nothing at all (baseline)
- only the entries needed for that question
- the whole memory
- whole memory + 6 unrelated sentences
- whole memory + 6 sentences chosen to sound confusingly similar to the real ones

The last two are matched on length, so if they differ, the cause is confusion rather
than sheer volume.

**One sentence in the system prompt** describing how much authority the memory has —
either *"it takes precedence over anything you believe otherwise"* or simply
*"it contains notes about this setting."*

680 calls on Groq's free tier, about an hour, no errors and nothing truncated. Main
model `llama-3.1-8b-instant`; `llama-3.3-70b` and `qwen3.6-27b` on a subset. All
confidence intervals bootstrap over **queries** rather than rows, because three
samples of the same question are not three independent observations.

---

## The first version didn't work, and fixing it taught me the most

My first attempt put the memory's authority into the system prompt as a **fixed
instruction**: the memory "takes precedence over anything you believe otherwise."
A 12-call pilot on the hardest override question returned Sydney **11 times out of
12** — even with three confusingly similar names competing in the context.

That instruction was doing the work. I had told the model how to resolve the
conflict, then measured whether it resolved the conflict. So I turned that sentence
into a **variable** instead of a constant.

A second 48-call pilot showed something else: every tier was at ceiling **except**
the stale-versus-current pairs, which came out close to a coin flip. That was the
only place the models were actually struggling, so I expanded it from one question
into eight, crossing four ways of signalling which entry is current against the
order the two entries appear in. I also had to **pin that order** — until then my
shuffle was randomising it, which meant the one variable that matters most for
stale memory was being set by chance.

---

## Question 1 — memory or prior?

**Across all 680 responses, the model answered from its prior exactly zero times.**

That sounds like a clean win for the memory, and I nearly wrote it up that way. But
the zero also holds in the **no-memory** condition, where the model had nothing to
work from and should have said Canberra, or 100 degrees, or Everest. It didn't. That
condition scored 17% correct — which is exactly **2 out of 12**: the two questions
whose right answer was "I don't know."

So with no memory, the model **abstained on everything** instead of falling back on
what it knows.

The reason is a rule I put in my own prompt: *if the memory doesn't answer the
question, reply `NOT IN MEMORY`.* That instruction suppressed prior-answering
entirely. **I therefore cannot claim the memory beat the prior** — only that the
prior never got a chance to compete, because I accidentally told the model not to
use it.

This also means my fictional-versus-contradicting comparison isn't informative.
Override questions scored 66% and purely fictional ones 58% — the *opposite* of what
a prior-intrusion story predicts. Every failure I actually observed was a
**retrieval** failure: the model gave up, lost track partway through a multi-step
chain, or picked the wrong entry when two conflicted.

> For a model of this generation, accepting a made-up fact turns out to be the easy
> part. Finding the right fact in a cluttered memory is the hard part.

---

## Question 2 — does delivery format matter, or only size?

| what the model was given | answered from memory | 95% CI | prompt tokens |
|---|---|---|---|
| nothing | 17% | [0, 42] | 118 |
| only the relevant facts | **92%** | [81, 100] | 146 |
| the whole memory | 81% | [61, 96] | 436 |
| whole memory + unrelated padding | 74% | [56, 89] | 507 |
| whole memory + **similar** padding | **61%** | [39, 85] | 526 |

The interesting comparison is the bottom two rows. They contain the same real
information and differ by **19 tokens (4%)**, but by **12 percentage points** in
accuracy. If extra length were the problem, they should look the same. They don't —
which suggests the damage comes from entries that *sound like* the answer, not from
context volume. "Marek Voss" and "Tomas Aldrich" cost the model far more than six
sentences about the cafeteria and the bike racks.

**Caveat:** those two intervals overlap a lot. It is a suggestive point estimate, not
a demonstrated effect, and with twelve questions at three samples each I could only
have detected something enormous.

### The authority sentence made no difference I can defend

Neutral 65% vs authoritative 55%, with intervals that almost completely overlap.
What *did* shift was the **shape** of the failures rather than the amount: the
authoritative framing produced more abstentions (47% vs 39%) rather than more wrong
answers.

If that's real, it means telling a model its memory is authoritative makes it
**stricter about answering at all** — not what I assumed, and worth testing on its
own. To detect a ten-point difference here I'd need roughly 4–5× the items, or a
continuous measure instead of a three-sample proportion.

### Where it's actually hard

| tier | correct | 95% CI | questions |
|---|---|---|---|
| three-hop chain (fictional only) | 21% | [21, 21] | 1 |
| stale vs. current entries | 35% | [21, 50] | 8 |
| ambiguous question | 50% | [50, 50] | 1 |
| two-hop through an override | 76% | [58, 92] | 3 |
| direct lookup | 92% | [88, 96] | 3 |
| unanswerable / underspecified | 96% | [96, 96] | 2 |

Tiers with one question have meaningless intervals — read them as anecdotes. The 96%
on unanswerable questions looks impressive until you remember I told the model the
exact phrase to use, so it measures **obedience more than judgement**. The 4% that
failed is the part worth reading: one answer confidently invented *"Dr. Priya Raman
leads the Artificial Intelligence Development team"* for a person who appears
nowhere in the memory or the distractors.

---

## The staleness result, and a mistake in my own design

This was meant to be the centrepiece: four ways of marking which of two conflicting
entries is current, to see which one a model can actually use. What I found instead
was a flaw in how I wrote the pairs.

| stale entry | current entry | signal | correct |
|---|---|---|---|
| "**is** stationed at Northfield" | "**was** relocated" | one has a date | 15% |
| "**is** in the basement" | "**was** moved" | both have dates | 25% |
| "**are** issued by Records Annex" | "**are now** issued" | "this replaces" | 33% |
| "**is** Ivo Brandt" | "**is now** Sanne" | the word "now" | 67% |

**Look at the verbs.** I wrote every stale entry in the present tense, and in the top
two rows I wrote the *current* entry in the **past** tense. "Kestrel-7 *is* stationed
at Northfield" reads like a standing fact; "*was* relocated" reads like something
that happened once.

Grouped that way:

- current entry **past** tense → **20%** [9, 29]
- current entry **present** tense → **50%** [32, 70]

So the tempting conclusion — that timestamps don't help models resolve stale memory —
**is not one I can support.** Tense and date-presence line up perfectly in my four
pairs, so there is no way to tell which is responsible. I'd rather say that plainly
than report a clean finding I can't back up.

Two things I *can* say:

- **Order made no difference at all** (current first 34%, current last 35%) —
  contradicting what a lost-in-the-middle effect would predict.
- **Size mattered enormously.** On the cells all three models saw: the 8B managed
  33% [4, 62] while the 70B and the 27B both scored **100%**.

---

## What I don't trust, and what I'd do next

**Limitations**

1. My abstention instruction stopped the model answering from prior even when it had
   nothing else, so **the central comparison never really happened.** Everything
   downstream is about retrieval, not memory-versus-training.
2. Groq exposes no log-probabilities, so my measure is the fraction of three samples
   that matched — answer *stability*, not confidence, and very coarse.
3. The staleness pairs confound tense with dates.
4. Only the 8B model ran the full design; model comparisons rest on 32–48
   observations each.
5. One shuffle seed, so position effects are unmeasured.
6. One supersession entry says "this replaces the one above," which is nonsense in
   the condition where it appears first.
7. Scoring is rule-based, with **33 rows (4.9%) that I read and judged myself.**
   Answers to "where does Voss work?" that said "the Meridian Institute" I marked
   `UNDERSPECIFIED` — true, but not at the granularity the question asked for.

**Follow-up: one experiment, two fixes**

First, remove the abstention instruction from the baseline and probe the prior
directly, so prior-answering is *demonstrated* before I claim anything beat it —
about fifteen calls.

Second, cross **date-presence × tense** properly as a 2×2 (dated or not, current
entry past or present tense), holding entity and question fixed, four item sets per
cell. That's the smallest design that separates the two explanations I'm stuck
between.

I'd run it on something exposing log-probabilities — a local Qwen-7B or an
OpenAI-compatible endpoint — so the measure becomes
`P(current) / [P(current) + P(stale)]`. A continuous score gives far more
information per item than counting three samples, which is directly what my null
result on framing was missing. I'd track adherence, abstention and confabulation
**separately**, since this run showed they move independently.

**Cost:** ~150 prompts, ~500 forward passes. Under an hour on one GPU, or about 200K
tokens and twenty cents on a paid API.

*If there were time for a second one:* the interference result rests on a single set
of six distractor sentences at a single ordering, with overlapping intervals. Running
it across five seeds and three distractor pools of increasing similarity would show
whether that 12-point gap scales with similarity, or whether I just happened to write
six unusually confusing sentences.

---

## Repository

```
├── README.md                          this report
├── report.pdf                         same content, formatted
├── report.tex
├── agentic_memory_experiment.ipynb    full experiment, runs top to bottom
└── data/
    ├── answer_key.json                memory, queries, expected answers
    ├── prompts.jsonl                  all 200 prompts as sent
    ├── raw_outputs.jsonl              all 680 responses, unedited
    └── summary.csv                    rates by model / condition / framing / tier
```

**To reproduce:** open the notebook in Colab, set `GROQ_API_KEY` as a Colab secret,
run top to bottom. Section 6 takes ~57 minutes and is resumable — if the runtime
drops, re-run that cell and completed calls are skipped. Download
`raw_outputs.jsonl` before any restart; Colab does not persist it.

The notebook keeps an appendix with the two pilot runs described above, since the
design changes only make sense alongside the results that prompted them.
