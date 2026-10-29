# Small "decision models": do they help Claude?

*A plain-language test report. October 2026.*

## The short answer

We tested small AI models that only give short answers, like "yes", "no" or "category B".
We wanted to know if they make Claude Code faster or cheaper.

**They did not.** For everyday work with Claude, they gave no gain.

- Claude alone was faster. It was also as accurate, or more accurate.
- Claude never chose the helper by itself. It ignored it in 9 of 9 runs.
- When we forced Claude to use the helper, one job took 8 times longer (141 seconds instead of 16).

The models are not useless. They can make sense inside your own app. That happens when you must run thousands of small checks, or when your data may not leave your own machine. We explain this below.

If you only use Claude Desktop or Claude Code to work, you need to do nothing. Do not install anything for this.

## What is a decision model?

Claude is like a thinking expert. You give it a task, it reads, thinks, and writes a long answer.

A decision model is like a fast sorting clerk. It does not write text. It gives one short, typed answer:

- yes or no ("Is this line a real error?")
- one choice from a list ("Is this change a bug fix, a new feature, or a doc update?")
- a score ("How sure are you?")

The clerk is small. It runs on a normal laptop graphics card (GPU). It needs no internet and no cloud. It costs nothing per answer.

## Why we tested it

The idea sounds good:

1. Some tasks have a very large input. For example, a log file with 900 lines, or 400 files.
2. The small model reads everything first and keeps only the important parts.
3. Claude then reads less. That should save time and money.

We wanted to know if this idea works in daily practice.

## How we tested

**Computer:** a laptop with an NVIDIA RTX 2070 Super Max-Q graphics card (8 GB memory).

**Three small models**, all free to download and run:

| Model | Made by | Size | Licence |
| --- | --- | --- | --- |
| tev1:4b | Together AI | 4.6 GB | not stated |
| d1:3b | Liquid AI | 3.7 GB | LFM Open |
| d3:4b | vLLM Semantic Router | 5.8 GB | Apache-2.0 |

**One connector:** a free add-on ("skill") called quicksilver. It lets Claude Code ask the small model for help.

**Claude:** Claude Code 2.1.310 with Opus 5.5.

**Three jobs.** We wrote the correct answers *before* we ran any test.

| Job | What the input was | What was correct |
| --- | --- | --- |
| 1. Find the errors | A log file with 920 lines. About 70 lines say "error" but are harmless. | 20 real failures |
| 2. Find the files | 395 code files | 32–33 files that do one specific kind of database work |
| 3. Sort the notes | 42 real change notes, with the label removed | The original label: new feature, fix, docs, research, test, other |

## Results, round 1: small model against small model

First we compared the three small models with each other. We also compared them to `grep`. Grep is a very simple free search tool. It looks for the word "error".

| | Plain search | d3:4b | tev1:4b | d1:3b |
| --- | --- | --- | --- | --- |
| Job 1: real failures found (of 20) | 19 | **20** | 18 | 19 |
| Job 1: flagged lines that were really errors | 23% | **74%** | 53% | 15% |
| Job 1: time | 0 s | 107 s | 114 s | 43 s |
| Job 2: files found (of 32) | – | 30 | 30 | 30 |
| Job 2: flagged files that were correct | – | **88%** | 86% | 81% |
| Job 2: time | – | 259 s | 275 s | 172 s |
| Job 3: notes sorted correctly | – | **76%** | 60% | 67% |

**How to read this:**

- d3:4b was the best small model on every measure that counts.
- d1:3b was the fastest, but it flagged many harmless lines. Only 15% of its flags were real errors.
- Plain search was fast and free, but it missed one real failure and most of its flags were wrong.

## Results, round 2: Claude alone against Claude with the helper

Now the real question. Does Claude do better with d3:4b as a helper?

We ran each setup 3 times (2 times for "forced"). The table shows the middle value.

| Job | Setup | Time | Tokens read | Times helper was used | Result |
| --- | --- | --- | --- | --- | --- |
| Errors | Claude alone | **16 s** | 184k | – | 20 of 20, none wrong, every run |
| Errors | Helper available | 16 s | 185k | 0 | 20 of 20, none wrong |
| Errors | Helper forced | 141 s | 287k | 2 per run | 20 of 20; one run had 2 wrong |
| Files | Claude alone | 28 s | 138k | – | 30–33 of 33, none wrong |
| Files | Helper available | 28 s | 183k | 0 | 30–32 of 33, none wrong |
| Files | Helper forced | 20 s | 189k | 0 | 31 of 33; Claude used grep anyway |
| Sorting | Claude alone | 22 s | 96k | – | 88–93% correct |
| Sorting | Helper available | 22 s | 96k | 0 | 88–93% correct |

*A token is a small piece of text, about three-quarters of a word. It is how AI models count how much they read.*

**What this shows:**

- When the helper was available, Claude **never used it**. Claude saw that it did not need it.
- When we forced it, the work got slower. Tokens went up. Accuracy did not improve.
- On the sorting job, Claude (88–93%) was clearly better than the best small model (76%).

## Why it does not pay off

1. **The small model reads one item at a time.** Each item takes about 0.1 seconds. 920 lines take about 2 minutes. Claude reads the whole 63 KB log in one step.
2. **Every Claude task has a fixed start-up cost.** About 40,000 tokens go to the system prompt and the project instructions. Saving a few thousand tokens on the input does not matter next to that.
3. **The helper adds steps.** Claude needed 6–7 turns instead of 3. Each turn re-reads the stored context. So the total went up, not down.
4. **Files in a well-organised project are small.** Claude finds the right file with a simple search and reads a few lines. It does not need a filter.
5. **Judgment is hard for small models.** Claude is better at "what kind of change is this?" than any small model we tested.

## What this means if you use Claude Desktop or Claude Code

- You do not need decision models. Claude already does this work well, alone.
- Do not add extra helpers hoping for speed. In our test, they made things slower.
- Cost of this whole test: **$5.16** in Claude usage.

## What this means if you build apps

Our test was about *developer tools*. Building a product is a different question. Small decision models can make sense when:

- **You have a very large number of small checks.** Examples: sort 100,000 support tickets, or scan every incoming message for a risk.
- **Data must stay on your own machine.** The small model runs locally. No text goes to a cloud.
- **The input is too big for Claude to read.** For example, a 50,000-line log. We did not test this size.
- **The answer is a fixed list of choices.** Yes/no or one of 5 categories fits well. Open-ended judgment does not.
- **You want a cheap first filter.** The small model removes the obvious cases. Claude handles the hard ones.

Be careful with accuracy. Our best small model sorted 76% of the notes correctly. It also flagged a few harmless lines, and it missed a few real items. If a missed item is costly, keep a human or Claude as a second check.

### Practical tips from our test

- **Make the whole model run on the graphics card.** The default tool (Ollama) left part of the model on the slower main processor. We made a small settings file with 3 lines: `FROM d3:4b`, `PARAMETER num_gpu 99`, `PARAMETER num_ctx 4096`. There was no new download. A warm answer then took 0.09 seconds instead of 0.58 seconds.
- **Watch the heat.** Under full load, the laptop GPU reached 78–80 °C and slowed itself down. Nothing reached a danger level. We waited for the GPU to cool before each timed test.
- **Check the licence.** Each model has its own licence. One of ours does not state one. Read it before you build a product.
- **Server tweaks helped little.** Flash attention and cache settings made almost no difference on their own.

## Safety note on add-ons (skills)

A Claude skill is a small package of instructions and code. It can send data and run commands. Treat it like any software you install.

- **Use the original source only.** Pin it to one exact version. We found copy-cat versions of the same skill with the same description. They were later deleted from GitHub. That is a common sign of a malware campaign.
- **Read the code before you trust it.** We read all of quicksilver. It had no dependencies, made one kind of network call, and wrote only to its own folder.
- **Check where your data goes.** The default setting of this skill sends your content to a cloud service. We changed it to work only with the local computer (127.0.0.1). It now refuses any other address.

## Limits of our test

- We tested mid-size inputs only. Inputs too big for Claude to read were not tested.
- The error log was generated by us. It had real formats and planted failures.
- We ran each setup 3 times (2 times when forced), on one machine.
- A faster computer, or newer models, may give different results.

## When to look again

Test again if:

1. A real job has input that is too big for Claude to read.
2. You get a faster graphics card, or a tool that judges many items in one pass.
3. A small model gets close to Claude on judgment tasks.

## Bottom line

For daily work, Claude alone is faster and as good. Small decision models are a tool for apps that need many cheap, private, simple checks. They are not a shortcut for Claude.
