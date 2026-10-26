---
title: "The second that decides everything: a story about perceived latency"
date: 2026-10-26
draft: false
author: "Sujit Yadawad"
tags: ["performance", "iOS", "mobile", "ux"]
description: "A large shopping app, a blank grey screen, and what I learned building the thing that made it feel fast."
toc: true
---

A few months ago I kept coming back to the same uncomfortable thought while using the shopping app I work on. On Android the numbers looked healthy, pages loaded in reasonable time and the dashboards said everything was fine. On iOS, the same app, the story was worse: the perceived-latency numbers were genuinely not good, and more to the point, in the first moment after I tapped, it did not *feel* fast. There was a beat where the screen went blank and I sat there, for just a second, wondering whether anything was happening.

That second is the whole story. It is where a lot of products quietly lose people, and it is where I ended up spending a good chunk of my work.

## Two kinds of fast

There are two different numbers hiding inside the word "fast."

One is the number engineers measure: the request came back in so many milliseconds, the page rendered in so much time. The other is the one the customer feels: did the app react when I touched it, or did I stare at an empty screen and start to doubt it?

These are not the same number, and the gap between them is bigger than most teams think. A customer never sees your percentiles. They see a screen. If that screen is blank while data loads, the app feels slow even when the real load time is perfectly fine. If the screen immediately shows the *shape* of what is coming, the exact same load time feels quick.

{{< figure src="/images/blog/perceived-vs-actual.svg" alt="Two timelines of equal length; the blank-screen wait feels slow throughout, the skeleton wait reads as progress" >}}

The backend call did not change. What changed is what the person looked at while they waited.

## Why iOS felt worse than Android

There is a specific, almost pedantic reason the blank beat was more painful on iOS, and it is worth knowing because it shaped everything that followed.

The web view that iOS apps use does not paint the page as soon as the first bytes arrive. It waits until it has received *enough* renderable content, roughly a couple of hundred bytes, before it draws anything at all. Other browsers, like Chrome and Firefox, start painting as soon as the first byte is available. Apple's web view holds back. This is [documented behaviour that other developers have measured](https://developer.apple.com/forums/thread/736218): the web view "waits for ~250 renderable bytes before it starts to render HTML," unlike Chrome and Firefox which "start to render the content as soon as the first byte is available."

In practice that means the exact same page, served over the exact same network, shows a blank screen for longer on iOS than it does on Android, purely because of when the web view decides to make its first mark. The customer is doing nothing wrong and the network is doing nothing wrong. The platform is simply choosing to wait before it paints, and that choice lands directly on perceived latency. So on iOS the blank second was not a rounding error I was chasing for sport; it was a real, platform-level gap that our own numbers confirmed.

## Why the blank second costs so much

People decide whether something is working almost immediately, often before the real content has arrived. On a shopping app that opens to a blank beat, customers open it a little less, trust it a little less, and leave a little earlier. None of that shows up as a latency chart. It shows up as behaviour, which is much harder to notice and much more expensive.

Once I saw it that way, the problem reframed itself. This was not only a "make the server faster" problem. A big part of it was a design and client problem: what do we put on the screen during the unavoidable wait?

## But does anyone still care?

There is a fair objection here, and I heard it more than once. In a world of fast phones, quick networks, and customers who have supposedly never been more impatient, does half a second of blank screen really change anything? Is perceived latency a real lever, or a nice-sounding idea that engineers like to talk about? It is a reasonable doubt. Felt speed sounds soft, and soft things are easy to wave away when you are deciding where to spend effort.

I did not want to answer it with opinion. I wanted to answer it with evidence, on both the axis engineers argue about and the axis the business actually cares about. So the real question became: if we remove that blank second, does a latency metric move, and does customer behaviour move with it? Hold that question. The rest of this is how I answered it.

## The idea: show the shape, not the void

The answer we landed on is an old one, done well: a skeleton. Instead of a blank screen or a lonely spinner, the app paints a lightweight preview of the page's structure the instant you arrive. Grey blocks where the image and the text and the cards will be, roughly where they belong. The real content quietly swaps in as it arrives.

{{< figure src="/images/blog/blank-vs-skeleton.svg" alt="Side by side: a blank loading screen with a spinner versus a skeleton screen showing placeholder blocks" >}}

Nothing about the real load time changes. But the wait stops being an empty, anxious void and becomes an occupied, predictable one. The customer sees the page forming, so the second that used to breed doubt now reads as progress.

## It is not just us: Google treats the wait as a canvas

I am not the only one who takes this seriously, and the company I point to most is Google. They have spent real design effort on the loading moment, to the point where their guidelines treat a progress indicator as a first-class piece of the interface rather than an afterthought you drop in when something is slow.

The clearest example for me is the Google Home app. When it is connecting to a device, which is genuinely slow because it is doing real work over the network, it never leaves you staring at a dead spinner. The animation is lively and a little playful, it moves in a way that feels responsive, and the whole time it reads as "I am working on it" rather than "I am stuck." I have used it many times and not once did it make me feel like I was waiting around, even though, by the clock, I was.

That is not an accident. Google's design system now ships what they call an expressive loading indicator: instead of a flat spinning circle, it is a shape that moves and cycles through colour as it runs, deliberately designed to feel alive and fresh to the eye. The device-connection animation in particular has a "reaching out" quality, ripples pulsing outward while the app does its real work. Here is that idea, rough and in my own rendering:

{{< figure src="/images/blog/expressive-loader.gif" alt="An animated loader: coloured rings pulsing outward from a breathing core dot, cycling through blue, red, yellow, and green" >}}

Their own guidance makes the philosophy explicit: be honest about progress when you can show it, keep motion purposeful rather than decorative, and give the customer a sense of life during the wait. You can read it in [Material 3's progress-indicator guidelines](https://m3.material.io/components/progress-indicators/overview) and the newer [Material 3 expressive loading indicator for Android](https://developer.android.com/develop/ui/compose/components/loading-indicator), and if you want to see how much craft goes into these moments, this [Google Home animation study on Behance](https://www.behance.net/gallery/60610079/Google-Home-Animations) is worth a look. The thread that connects their work and mine is the same belief: the time a customer spends waiting is not dead space, it is a surface you get to design, and what you put there decides how the wait feels.

## Why this was the hard one on iOS

Here is the part people are often surprised by: the same idea was meaningfully harder to deliver on iOS than on Android, and the reason is instructive.

On Android, the team never had to touch the native app for this. They delivered skeletons through a caching layer in the web stack, the kind that can intercept a page request and serve a lightweight placeholder from local storage before the real page arrives. The app itself did not need to know skeletons existed.

On iOS that door was closed. The web engine Apple gives apps does not support that same background-caching mechanism in a way that works for an app spanning dozens of retail domains, and the other interception tricks we evaluated all fell over for one reason or another. So iOS could not borrow Android's approach at all. We had to do it natively: paint the skeleton ourselves as a layer over the loading page, and manage its whole life by hand.

And the hard part was never drawing grey boxes. It was timing, specifically knowing when to take the skeleton *down*. There is no clean, trustworthy signal that says "the real page is ready now." The best the app can do is watch what is actually being painted on screen and infer that real content has started to appear. Lean too eager and the skeleton vanishes while the page is still blank underneath, so the customer sees skeleton, then a blank flash, then content, which is worse than either alone. Wait too long and you have thrown away the very feeling of speed you were trying to create. Getting that moment right, reliably, was the whole game.

On top of that, a shopping app this size is a patchwork of many screens built by many teams, some rendered natively and some as web content. A skeleton is only worth building if it behaves correctly across all of them without every team having to re-solve the timing puzzle for their screen.

## The design decision I am proudest of

The tempting move was to build a separate skeleton system that ran alongside the existing loading machinery. I deliberately did not. Two systems watching the same page and both trying to show and hide something at the same moment is how you get flicker, things stuck on screen, and bugs no one can reproduce.

Instead I folded the skeleton into the loading path the app already had. The existing machinery still decides when loading starts and ends; the skeleton simply rides along as what gets shown during that window, and if anything about it is unavailable the app falls back to exactly the old behaviour with no special handling. That gave me one code path instead of two, a safe fallback by default, and a system where any new screen can adopt a skeleton with just a small piece of configuration and an asset, no changes to the core. The win could spread across the app instead of living in one place.

## A peek under the hood

For the engineers reading this, here is the shape of it in plain iOS terms, without the internal details.

The skeleton itself is deliberately boring: a `UIImageView` showing a lightweight image of the page layout, bundled with the app and laid over the web view (`WKWebView`). Because it is a local image and not a web render, it appears in well under a frame with zero network calls. I considered vector and native-layout skeletons too, but a pre-rendered image wins on the one thing that matters here, time-to-first-pixel, and the view stays abstract enough that I can swap the internals later without touching anything around it.

The timing hangs off the web view's own navigation lifecycle. `WKWebView` hands its delegate a sequence of callbacks as a page loads, and I use them as the show and hide triggers: show the skeleton the moment a page starts loading (the earliest point the app knows a new page is coming), and hide it on the first reliable "real content is painting now" signal. I fall back to the later finished and stopped callbacks, and keep a timeout as a last-resort safety net so the skeleton can never get stuck on screen. When it does hide, it fades out with a short Core Animation transition rather than cutting, which bridges the moment real content takes over so the swap never looks abrupt.

Memory is handled by letting the platform do the work. The skeleton is a child of the loading container, so it inherits that container's lifecycle, when loading ends the view is pulled out of the hierarchy and released by ARC, which means it costs nothing when it is not on screen. A teardown hook clears it immediately if the web view is disposed, and because the whole thing is an overlay that sits outside the page-load path, it adds no measurable cost to the real load it is decorating.

## How I knew it actually worked

This is the part I care about most, because "it feels faster" is the easiest claim in engineering to fool yourself with. If you ship a perceived-performance change and just look at it, you will always conclude it worked, because you wanted it to.

So I treated "feels faster" as a claim to be proven, not a vibe to be admired:

- I measured a perceived-speed signal, something that approximates when the customer sees meaningful content, not only when the server answered.
- I ran it as a real experiment, comparing customers who got the skeleton against customers who did not, on enough traffic that the result was not noise, and I checked that it held over time rather than in one lucky window.
- I kept a guardrail on genuine interactivity. The failure mode of any perceived-speed trick is making the page merely *look* ready before it actually is, so the time to real interactivity had to stay flat. If felt speed improved while real responsiveness got worse, that is not a win, it is a costume.

The change only counted when the felt-speed signal moved, the interactivity guardrail held, and the result survived that controlled comparison. It did, and the app felt dramatically quicker to open while staying exactly as responsive as before.

## The doubt, answered

Which brings me back to the skeptic's question: does anyone actually care about perceived latency, or is it a soft idea that does not survive contact with real data? The experiment answered it on both axes, and that is the part I find most convincing.

On the latency axis, the perceived-speed metric improved clearly and held up across a large, controlled comparison, while real interactivity stayed flat. The win was not a measurement artifact; it was the customer genuinely seeing meaningful content sooner.

On the business axis, and this is the one that ends the argument, customer behaviour moved too. Removing that blank second did not just make the app feel better in a survey sense; it showed up in the numbers the business watches, enough that the result cleared the bar to launch rather than being written off as a nice-to-have. That is the evidence that converts "felt speed sounds soft" into "felt speed pays for itself." A perceived-latency change is not decoration. When you remove the blank second at scale, people notice, and they behave differently, and both of those are measurable.

This matters beyond one feature. It is easy for a team to deprioritize perceived performance because it sounds cosmetic next to "hard" work on throughput or backend latency. The evidence says that is a mistake. The blank second is not cosmetic. It is one of the cheapest, highest-leverage things you can fix, and it moves the metrics on both sides of the table.

## What I took away

A few things I am keeping from this:

- **Real latency and perceived latency are different numbers, and customers react to the second one.** They never see your percentiles; they see a screen. The time after a tap is often decided entirely by what you chose to put there while they waited.
- **The blank second is not cosmetic, and the doubt about it is now settled for me.** The question I started with was whether anyone still cares about perceived latency in a world of fast phones and short attention spans. The experiment answered it on both axes: a perceived-speed metric moved and held under a controlled comparison, and customer behaviour moved with it, enough to clear the bar to launch. Customers still care. The evidence says so, not my opinion.
- **Prove felt speed, do not admire it.** It is the easiest thing in engineering to fool yourself about, so measure it directly, run it against a control, and keep a guardrail on real interactivity so you are not shipping a costume.
- **Treat the wait as a surface you design.** A skeleton that shows the shape, or a loader that feels alive the way Google's does, both come from the same belief: the loading moment is not dead space.

So yes, perceived latency is worth your time. It is one of the cheapest, highest-leverage things you can improve, and the return shows up on both sides of the table, the latency metric and the business one.

One honest caveat to close on. Making an app *feel* fast is not a licence to stop making it *actually* fast. Perceived-latency work buys you the first second; real page-loading work is what earns the rest, and the two are complementary, not substitutes. How I think about genuinely reducing page load, not just the feel of it, is a longer story. I will write that one up another day.
