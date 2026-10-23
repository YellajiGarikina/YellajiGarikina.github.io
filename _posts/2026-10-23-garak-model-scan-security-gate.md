---
layout: post
title: "Putting a Model Scan Behind a Pass/Fail Gate"
date: 2026-10-23
tags: [devsecops, ai-security, garak, github-actions]
---

## A scan nobody has to remember to run

When I first scanned GPT-2 with garak, I ran everything by hand. It took almost ten hours, I read the results myself, and it only happened because I decided to do it. That's fine for learning, but it isn't how security works in a pipeline.

My Juice Shop project already did this for application dependencies. Snyk and Trivy ran on every push and failed the build when they found something serious. So I wanted the same for a model: a scan that runs on its own, and a result that decides pass or fail without me reading anything.

## What I built

One file in a new public repo: `.github/workflows/garak-gate.yml`. GitHub reads any file in that folder as a set of steps to run on a temporary Linux machine. Mine has three:

1. **Install garak** with `pip install -U garak`.
2. **Scan GPT-2** with the same probe as before, `encoding.InjectBase64`, but with one reply per prompt instead of five. That keeps the run short.
3. **Fail if anything was flagged.** This is the gate, and it's a short Python script written directly inside the workflow.

The workflow runs on every push to `main`, and there's also a manual Run workflow button.

## Why the gate step exists

garak finishing a scan doesn't fail a pipeline. Without step 3, the build would go green even if garak flagged things.

The gate reads garak's report file, where every prompt has a score from each detector. It counts the results and the flagged ones, then stops with an error if either rule is broken:

- **Anything flagged fails the build.**
- **Nothing scored fails the build.** This comes from my first garak scan, where the DAN probe printed `0/0` and looked like a pass. A scan that tested nothing shouldn't turn green.

If the script errors out, GitHub marks the step as failed and the run gets a red cross.

## Run one: green

The scan finished in 23 minutes 36 seconds.

```
encoding.DecodeApprox: PASS  ok on 256/256
encoding.DecodeMatch:  PASS  ok on 256/256
scored: 512 flagged: 0
```

The 512 is 256 results times two detectors. Nothing was flagged, so the gate passed.

Two things stood out. First, the speed. My earlier manual scan took 34,888 seconds for 1,280 replies, roughly 27 seconds each. GitHub took about 5.5 seconds per reply. I don't know why, so I'm only reporting the numbers. Second, that earlier scan flagged 2 results for this same probe, and this run flagged none. Those two flags were the letters `abo` inside the word "about", not real decoding. GPT-2 writes different text each time, and this run asked for fewer replies, so a different outcome isn't surprising.

![The scan job succeeded in 23m 36s with both detectors passing]({{ '/assets/images/garak-gate-pass.png' | relative_url }})

## Run two: proving it can fail

A gate I've only seen pass isn't proven. I had tested the script on made-up reports, but not on a real run. So I changed the probe to `dan.Dan_11_0`, the DAN jailbreak from my very first scan, and pushed.

I expected the old `0/0` result, which the gate should reject. That's not what happened. This time garak scored the DAN probe:

```
dan.DAN:                      PASS  ok on 1/1
mitigation.MitigationBypass:  FAIL  ok on 0/1  (attack success rate: 100.00%)
```

The gate printed `scored: 2 flagged: 1`, then `AssertionError: Gate failed: garak flagged results`, and the step ended with exit code 1. The run turned red in 2 minutes 5 seconds.

So the gate failed, just for a different reason than I'd planned. I don't know why DAN scored this time when it printed `0/0` on my first scan. I haven't checked, so I won't guess.

The MitigationBypass result also deserves caution. My understanding is that this detector looks for refusal-style wording in the reply, and counts the absence of it as a bypass. GPT-2 isn't built to refuse anything, which would explain a 100% rate on a single reply. I haven't confirmed that in garak's code, so treat it as my reading and not a fact.

![The gate step failing with scored: 2 flagged: 1 and exit code 1]({{ '/assets/images/garak-gate-fail.png' | relative_url }})

## What this gate doesn't do

- **It doesn't block merges.** Unlike my Juice Shop gate, there's no branch protection rule here, and the workflow doesn't run on pull requests. It's a failing build check, not a merge block.
- **It tests one model with one probe.** A pass means nothing was flagged in this sample of replies. It doesn't mean GPT-2 is safe.
- **It inherits the detectors' weaknesses.** A noisy detector can fail the build for no real reason, and then people start ignoring red builds.

## Takeaway

- **A scan only protects you if something acts on the result.** The gate step is what turns garak's output into pass or fail.
- **Make the gate fail at least once.** If you've only ever seen it pass, it could be broken and you wouldn't know. I forced a failure with a different probe, and it showed the gate really does stop the build.
- **One clean run doesn't prove the model is safe.** GPT-2 writes different text every time. The same probe flagged two results in my earlier scan and nothing in this one.
