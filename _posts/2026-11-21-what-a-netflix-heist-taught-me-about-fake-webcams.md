---
layout: post
title: "What a Netflix Heist Taught Me About Fake Webcams"
date: 2026-11-21
tags: [deepfakes, liveness-detection, ai-security]
---

### A Gadget From a Heist Show

I was watching Lupin Part 4 recently, and there's a scene where a gadget swaps someone's face on camera in real time. Whoever the camera records just shows up as someone else. It's a flashy plot device, but as a security tester my first reaction wasn't "cool gadget." It was "how close is this to something real, and how would you actually catch it?"

That question turned into a weekend project, then a longer one.

### Why This, Specifically

Video verification is everywhere now: bank onboarding, online exams, remote interviews. All of it rests on one assumption, that the person on camera is a real, live human being right now.

> Liveness detection is a way of checking that a video feed shows a real, present person right now, not a photo, a recording, or a pre-made video.

I wanted to test that assumption myself rather than just read about it, so I built something to attack.

### Building a Target Worth Attacking

Before attacking anything, I needed something to attack. So I put together a small browser page that tries to confirm a person is really there. It starts simple: check the camera's name, its resolution, its frame rate. Then it gets more active: it throws a random instruction at you, like "blink twice" or "turn your head left," and watches your face through MediaPipe's face tracking library, running right in the browser to see if you actually do it within a few seconds.

It's the same idea a bank clerk uses on a video call. Looking at your face isn't enough, so they ask you to do something unpredictable, like touch your ear and say a number. A photo or a recording can't know what it'll be asked before it's asked.

### Round One: Just Replaying Video

My first attack was the simplest one.

> A replay attack means feeding a system a video that was recorded earlier, instead of a live camera feed, and hoping it can't tell the difference.

Using OBS Studio's virtual camera feature, I made the browser believe a completely different video feed was a live webcam. I tried the passive checks first, camera name, resolution, frame rate, and all three turned out to be easy to fake. A careful attacker can rename a virtual camera and match its resolution to a real one in a couple of clicks.

Then I turned on the random challenge. A genuinely live feed passed every time, which is the result I wanted, since a real system shouldn't punish someone for using different camera software. A looped recording of myself, on the other hand, only passed about one time in four. It got lucky when the random instruction happened to match something already happening in the loop, which is its own useful finding. A random challenge is a real improvement over passive checks, but alone it isn't bulletproof.

### Round Two: An Actual Live Face Swap

The harder question was what happens when the attacker isn't replaying old footage, but is live, with their face swapped for someone else's in real time. For this part I only ever used my own face, and nothing here was run against a real company's live system, strictly a personal lab test.

I set up Deep-Live-Cam, an open source real time face swap tool, and ran it without any GPU acceleration. The limitation showed up fast:

```
Processing: 37%|▎| 665/1818 [02:29<03:20, 5.76frame/s,
```

Under 6 frames per second, against a normal webcam's 25 to 30. In plain terms, genuinely live, real time face swapping at that quality needs serious GPU power. Without one, it's nowhere near usable.

Even on recorded video, where speed wasn't the issue, the swap was far from flawless. Two things gave it away immediately. Putting a finger over part of my face broke the illusion completely, with my real face peeking through underneath. And the mouth movement never quite matched what was actually being said, since the tool swaps face shape but has no idea what the audio sounds like.

### What Actually Catches This Stuff

Putting both rounds together, a few defenses stood out.

A single check is never enough. Camera name and resolution are trivial to fake, so they should only ever be one weak signal among several, never a decision on their own.

Random, unpredictable challenges genuinely help against simple replay attacks, but they aren't foolproof alone and work best stacked with something else.

Occlusion is a great trap for a face swap. Asking someone to briefly cover part of their face mid-challenge forces the algorithm into a scenario it tends to handle badly.

Audio and mouth movement should agree. Checking whether lip motion actually lines up with speech closes a gap that pure face detection doesn't even try to cover.

### Takeaway

A deepfake device like the one in the show is closer to fiction than practice, real time, convincing face swapping at that quality still needs hardware most people don't have lying around. But the underlying attack, replaying or faking a camera feed, is very real and very cheap to attempt. The defenses that actually hold up are the ones that stack several weak signals together rather than trusting any single check, and the ones that force a fake feed into situations it wasn't built to handle.
