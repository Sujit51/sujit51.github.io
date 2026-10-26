---
title: "Why perceived latency matters more than you think"
date: 2026-10-26
draft: false
author: "Sujit Yadawad"
tags: ["performance", "iOS", "mobile", "ux"]
description: "Real speed and felt speed are different numbers, and the one your customers react to is the one you probably aren't measuring."
toc: true
---

There are two kinds of fast. One is the number your dashboards show: the request took 400 milliseconds, the page rendered in 1.2 seconds. The other is the one your customer feels: did the app react when I touched it, or did I sit looking at a blank screen wondering if it was broken? These two numbers are not the same, and the gap between them is where a lot of products quietly lose people.

I have spent most of the last few years on high-traffic mobile navigation, and perceived latency is the metric I keep coming back to. Here is why it deserves more attention than it usually gets.

## Felt speed is the one customers act on

A customer cannot see your p50. They see a screen. If that screen is blank while data loads, the app feels slow even when the actual load time is perfectly reasonable. If the screen shows the shape of the content immediately, the same load time feels quick. The underlying network call did not change. What changed is what the person stared at while they waited.

This matters because people decide whether something is working in the first second, often before the real content has arrived. On a shopping app, a homepage that feels sluggish on open is a homepage people open less. The cost of slow is not a latency chart, it is behaviour: fewer sessions, earlier exits, less trust that tapping will do anything.

## Where the gap comes from

The gap between real and felt speed opens up at a few predictable places:

- **The blank wait.** The classic one: a grey screen or a spinner while a page fetches. Nothing tells the customer that progress is happening, so the wait feels longer than it is. Research on waiting has shown this for decades: an unoccupied wait feels longer than an occupied one, and uncertainty makes it worse.
- **Janky interactivity.** A screen that looks loaded but drops frames when you scroll, or ignores a tap for a beat, reads as slow even if the data arrived fast.
- **Layout shift on arrival.** Content that pops in and reflows under the customer's finger feels unstable, which registers as the app not being ready.

The lesson is that latency is not only a backend problem. A big part of it is a front-end and design problem, about what you show during the wait.

## A concrete fix: skeleton loading

The technique I have gotten the most mileage from is skeleton loading. Instead of a blank screen while a page loads, you render a lightweight preview of the page's structure immediately: grey blocks where the image and text will be, in roughly the right places. The real content swaps in when it arrives.

Nothing about the actual load time changes. What changes is that the customer sees the shape of the page within a frame instead of staring at grey, so the wait feels occupied and predictable rather than empty and uncertain. When I built this for a large shopping app, the perceived-speed metric improved by about a quarter while the time to actual interactivity stayed flat. The app felt dramatically faster at no real cost to interactivity.

## How to know it actually worked

This is the part I care about most, because "feels faster" is the easiest claim in engineering to fool yourself on. If you ship a perceived-performance change and eyeball it, you will always conclude it worked, because you wanted it to.

So treat felt speed as a hypothesis and prove it against a control:

1. **Pick a perceived-speed metric, not just a backend one.** Something that approximates when the customer sees meaningful content, not only when the server responded.
2. **Run it as a real A/B experiment.** Compare customers who got the change against those who did not, on enough traffic that the result is not noise, and confirm it holds across the whole window rather than a lucky slice.
3. **Keep a guardrail on real interactivity.** The failure mode of a perceived-speed trick is making the page look ready before it actually is, so time-to-interactive has to stay flat. If felt speed improves while interactivity regresses, you traded a real cost for a cosmetic win, and that is not a win.

The change only counts when the perceived metric moves, the guardrail holds, and the result survives a controlled comparison.

## The takeaway

Real latency and perceived latency are different numbers, and the one your customers react to is usually the one you are not measuring. Shaving milliseconds off a backend call is worth doing, but so is thinking carefully about what the screen shows while the customer waits, because that is the speed they actually experience. Measure felt speed directly, improve it deliberately, and prove it against a control. The payoff is an app that feels fast, which is the only kind of fast a customer ever notices.
