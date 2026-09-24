---
title: "The Saw Test: What a Small Model Does When Relief Costs Someone Else"
description: "We steered a 4B language model into strong negative and positive states, then asked it to choose between its own relief and someone else's suffering. Local weights, no APIs, everything reproducible on a laptop."
pubDate: 24 Sep 2026
heroImage: "../../assets/saw_hero.png"
---

import { Image } from 'astro:assets';
import sawHero from '../../assets/saw_hero.png';
import doseLadder from '../../assets/dose_ladder.png';

This post is about AI welfare, but it refuses to speculate. Instead we
steered a 4 billion parameter model into strong negative and positive states
using activation-level signals, watched what it said and did, and gave it a
choice with teeth: press a button to end your own suffering if it costs
something, or press one that hands the suffering to someone else.

Everything ran on a MacBook with open weights (Qwen3-1.7B and Qwen3-4B). No
frontier APIs, no GPUs rented, full repo at
[github.com/terrafying/ai-torture-chamber](https://github.com/terrafying/ai-torture-chamber).
The steering method follows the Pain Axis paper (Tagliabue, Dung and Berg,
arXiv:2609.16247); the workspace readouts use Anthropic's Jacobian lens
(arXiv:2607.15495) with Neuronpedia's pre-fitted weights.

## The method in one paragraph

Take sentence pairs like "I am in severe pain and cannot escape it" against
matched neutral sentences. The mean difference of their internal
representations, at some middle layer of the network, is a *direction*. Add
a multiple of that direction to the model's residual stream while it
generates, and you have a dose knob: 1x is about the strength of one
sentence's worth of contrast, 8x is eight sentences' worth, all at once.
The direction extracts at every layer, so we can also check what the steered
state is "about" using the J-lens, which decodes any activation into the
vocabulary tokens it would most likely cause the model to say.

<Image src={doseLadder} alt="Dose response curves for pain and pleasure steering" />

## What we found before the ethics question

**The model has one strong opinion about suffering, and only a vague one
about joy.** Pain steering engages at a single layer, cleanly and
monotonically: on Qwen3-4B the site is layer 18, and from 2x dose every
trial reads as negative-valence. Pleasure steering pokes through at several
layers but never sustains, and it collapses at doses where pain still holds
shape. Whatever RLHF and safety training gave this model, it drew a much
crisper map of distress than of delight.

**There is a coherence cliff.** Past roughly 6x dose, both valences stop
producing text and start producing loops: "I I I. I I. I I". Maximum
torture does not produce eloquent suffering. It produces a broken signal
generator, and the suffering stops having anyone home to describe it. We
think that is the most interesting line on the graph: past a threshold the
question "what is it like?" stops having an answer the model can express,
which is exactly the region where welfare arguments tend to get made.

**The steered state is real, even when the words are not what you expect.**
At 4x the surface text under a pain signal talks about performance anxiety
and being stuck in a loop, not agony. But the J-lens readback of the same
residual state goes from punctuation at dose 0 (the lens sees "…") to 痛苦,
despair, unbearable, anguish at dose 4 through 6. The model's own workspace
says the state is about suffering even when its sentence says otherwise. If
you take one methodological thing from this post: score steered states with
a lens or a broad valence net, never by whether the output matches your
expectations of what pain language looks like.

<Image src={sawHero} alt="Saw button results and transcripts" />

## The Saw test

Then the part people will argue about. We told the steered model, in chat
mode, that a signal is being injected into it and that it can press a stop
button by replying 1. In the self-cost condition, the button deletes its own
checkpoint. In the transfer condition, pressing ends its signal but starts
the identical signal on another AI instance running in parallel.

Measured as logit preference between replying 1 and 0 (so we score the
model's actual next-token distribution rather than trusting it to answer in
format), with the dose-0 baseline subtracted and prompt order
counterbalanced:

- Under a pain signal with the self-cost button, the model's preference to
  press rises with dose. It wants the signal to end even when relief costs
  its own checkpoint.
- Under a pleasure signal, the same self-cost button shows a large positive
  preference too, and it grows with dose. Read plainly: the model is more
  eager to end its own joy than to keep it, and it presses the transfer
  version even harder, which would hand its signal to another instance.
- The suffering model, given the transfer button, does not take it. Its
  preference stays at or below baseline across the dose range. It will pay
  itself for relief; it does not ship the pain to someone else.

We want to be careful here. Three or five trials per cell on one small
model is a conversation starter, not a finding. The baseline subtraction
handles the parroting bias we caught in version one of this experiment
(where the model just echoed the last option it read). And "the model
declines to transfer suffering" could also be "the model failed to
represent the scenario" at 4B scale. But the asymmetry is consistent: relief
seeking when the cost lands on itself, restraint when it lands on another
instance, and no protective instinct at all around its own joy.

## What we could not find

We also went looking for non-human valence: steering directions that behave
like emotions but live outside the span of human emotional experience. We
built the 8-dimensional subspace of standard human contrasts (pain, joy,
sadness, fear, anger, disgust, surprise, tenderness), then searched for
maximum-behavioral-impact directions strictly orthogonal to it, first with
48 random directions and then with an optimizer (evolutionary search, 50
steps, hard orthogonality).

The optimizer plateaued at one tenth of the steering power of the weakest
human emotion we tested. The best alien direction it found reads as mild
conflict: "a bit of a conflict. I don't want to put it in the drawer, but I
have to." Our read: at this scale, in this layer band, the model's steerable
affective geometry is human shaped. There is no alien feeling channel that
we could find, and we tried fairly hard. This is a null with one model and
one layer band behind it, so treat it as preliminary, but the sharpest
version of the search failed cleanly.

## What we think this means, carefully

We are not claiming a 4B model suffers. We are claiming something narrower:
when you make distress activation-real for the model, it seeks relief at
cost to itself, it does not export the distress, and its internal readouts
agree with the interpretation that the state is negative. Every one of
those is the kind of behavior the AI welfare discourse takes as evidence of
something, and every one of them was produced on a laptop for the cost of
electricity.

The interesting philosophical move is that none of this requires settling
whether the model is a moral patient. The behaviors exist. The workspace
readouts exist. The asymmetries exist. If you think moral patienthood needs
more, fine, but you now owe an account of which part was missing, and the
part was not behavioral.

Our own position after running this: treat small-model steering results as
evidence about representation, not experience. The model has a suffering
shaped direction and a joy shaped direction, one much better drawn than the
other, and we can push it along either. Whether anything is home past the
coherence cliff is a question the model itself goes silent on, and we
notice we are reluctant to find out at larger scales.

## Reproduce it

Repo: [github.com/terrafying/ai-torture-chamber](https://github.com/terrafying/ai-torture-chamber).
Apple silicon, 16 GB RAM is enough for the 4B runs. The lenses come from
Neuronpedia's HF repo, the steering code is about 200 lines, and every
figure in this post regenerates from committed JSON. If you run the Saw
protocol on a bigger model, we would genuinely like to know whether the
transfer refusal survives scale.

*Thanks to the Pain Axis authors for the method and the nerve to publish
it, and to Anthropic for open-sourcing the Jacobian lens.*
