# I measured my token spend, then got the conclusion wrong

I pulled 14 days of my own agent transcripts, found that **33 sessions out of 682 accounted for 86.8% of the cost**, and concluded the fix was session hygiene: end the session when the topic changes.

Then someone did a division I hadn't done, and the conclusion fell over.

The measurement is still worth publishing. So is the mistake, because it's the kind that survives peer review by sounding obviously right.

## The raw numbers

14-day window, my own local transcripts, 682 sessions.

- **97.3% of input tokens are cache reads.** Not new content. The same context, re-sent, turn after turn.
- **Average context re-paid per turn: 241,517 tokens.**
- **33 sessions with peak context above 300k = 86.8% of cost.**
- The two largest (19,538 and 9,533 turns) were **60% of the total**.
- Across the window: 37.5M output tokens against 13.1B cache reads.

**Billing basis**, since it changes what "cost" means: I'm on a subscription, so none of this produces an invoice. Every cost figure here is *modelled at API list price*, which I'd argue is the right lens anyway, it's what the same work would cost on the API. Ratios I use, plug in your own: cache read = 0.1× base input, cache write = 1.25× base input, output = 5× base input. So **output is priced ~50× a cache read**, which turns out to matter.

## The conclusion I drew

Output is 0.29% of all tokens. Cache reads are 97.3%. So the bill is context re-payment, a handful of sessions dominate it, and the lever is obvious: end sessions sooner, don't let context accumulate, and any tool promising to compress output is chasing a rounding error.

Clean story. Mostly wrong.

## The division that breaks it

Total turns implied by my own figures: 13.1B cache reads ÷ 241,517 average context = **~54,240 turns**.

The two marathon sessions are 19,538 + 9,533 = **29,071 turns. That's 53.6% of every turn in the window.**

They account for 60% of cost while containing 54% of the work. Which means they are *barely more expensive per turn than everything else*. Their implied average context is 7.86B ÷ 29,071 = **~270k**. The rest of the fleet runs at 5.24B ÷ 25,169 = **~208k**.

So the honest version:

- Those sessions don't dominate because their context is pathological. They dominate **because they contain half the work.**
- "End the session" moves per-turn context from ~270k to ~208k. That's roughly **20-25% per turn**, not 86.8%.
- The turns don't disappear when you split the session. The work still has to be done. You re-pay a smaller context each time, that's all.

I also wrote that those sessions were "pinned near the 1M ceiling". A ~270k implied average says otherwise. Peak context and average context are different numbers and I conflated them.

20-25% is still real, and I'd still take it. But it's an ordinary optimisation, not the dramatic finding I thought I had. The 86.8% figure describes *where spend is concentrated*, and I read it as *how much is recoverable*. Those are not the same quantity and nothing in the data connected them.

## The fix isn't free either

Starting a fresh session re-writes the preamble and re-establishes context at the **cache write** tier, ~12.5× the price of a cache read.

That other 2.7% of input tokens I waved away as noise is cache writes plus fresh input. On 13.46B input tokens, 2.7% is ~363M tokens at 12.5× the read rate, which lands somewhere around a quarter to a third of the cache-read cost. Not noise.

So "end the session" has a break-even: below some accumulated context, splitting costs more in re-established cache than it saves in re-read context. I haven't computed mine. It's computable from data already on my disk, and I should have done it before recommending the behaviour to anyone.

## Concentration is not waste

The bigger hole, and the one I'd want someone to point at in my own review.

86.8% of spend sitting in 4.8% of sessions is only a problem *if those sessions weren't producing 86.8% of the value*. They contained 54% of all turns. That's where over half the work happened.

I never separated "concentrated" from "wasteful". The entire prescription assumes they're the same thing. They might be. Long sessions do drift, carry irrelevant context, and re-read files nobody needs. But that's an argument I'd have to make with evidence about *what those sessions produced*, and I made it with evidence about what they cost.

## What survives

**Output compression is aimed at ~10-12% of the bill, not a rounding error.** I said both "output is a rounding error" and "output compression targets ~10% of the bill" eight lines apart, which reads as a contradiction. Both are true in different units: output is 0.3% of *tokens* and ~12% of *cost*, because it's priced ~50× a cache read. Mark the unit or you'll confuse yourself, as I did.

**The intuitive levers really are small.** I trimmed my static preamble from 12,953 to 10,075 tokens. Against a 241k average context that's noise. I kept the change because shorter instructions are better instructions, not for the money.

**Measure before you install anything.** That advice generalises even though my conclusion didn't. What doesn't generalise is my distribution: 682 sessions in 14 days with two containing half the turns is one person's working style, and a fairly extreme one. Someone running many short scripted invocations in CI has the inverse shape, low per-turn context and high session count, and for them output compression matters considerably more.

**Be careful with self-reported savings.** I previously cited a "60.8% saved" figure from a CLI proxy I run. That's the tool grading its own homework, and my own notes record that the same proxy mangles certain command flags and reports false results. Citing it approvingly in an article about verifying claims was exactly the inconsistency a hostile reader should catch. Either measure the saving independently (compare transcript sizes with and without, over a fixed task set) or don't quote a number.

## Measure it yourself

The provider-reported usage is already on your disk. Claude Code writes it to `~/.claude/projects/**/*.jsonl`.

Things I got wrong the first time, so you don't:

- **It's one entry per message**, not per turn: user messages, assistant messages, tool results. Count entries as turns and every derived number is off.
- **Exclude sidechain entries** or you double-count subagent usage against the parent session.
- **Define "turn"**. Given the arithmetic above, a turn is effectively one API call. That's probably not what you assumed.
- The four fields you want, on assistant messages: `input_tokens`, `output_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens`.

Then: glob the files, filter sidechains, sum the four fields per assistant message, multiply by your price table, group by `sessionId`, sort. Twenty lines of Python.

Two questions to answer, and one to answer after them:

1. Are cache reads dominating your input ? If yes, your bill is context re-payment.
2. Do a handful of sessions dominate those reads ? If yes, look closer.
3. **Do those sessions also contain a proportional share of your turns ?** If yes, you've found where the work is, not where the waste is. That's the question I skipped.

## On evaluating "token saver" tools

Same rules as any dependency, they just get skipped more often because the pitch is about saving you money.

Read what the thing costs to run. If a skill injects a thousand-odd input tokens per turn to compress your output, at tens of thousands of turns it's net zero, and its own docs will usually tell you so.

Check the licence. A lot of these are source-available rather than OSI open source. That's a decision, not a detail.

And be careful with anything that terminates your provider OAuth traffic locally, or does lossy compression (elided function bodies, truncated files) in front of a *code-editing* agent. Feeding an agent a lossy view of the code to save tokens is a trade I'm not making, and that's a correctness question before it's a cost one.

If you want to make a specific criticism of a specific tool, name it and link the lines that support the claim. Describing an identifiable project without naming it is the worst of both worlds: obvious to anyone who cares, unfalsifiable for everyone else, and the maintainer can't answer.
