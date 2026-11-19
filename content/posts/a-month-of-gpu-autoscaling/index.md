---
title: "My autoscaler rented a machine every hour, forever"
date: 2026-11-19
description: "A month of renting GPUs automatically on Vast.ai and RunPod: hosts that never boot, offers that no longer exist, alerts that ping-pong, and queue priorities that did nothing."
tags: ["ai-infrastructure", "runqy", "gpu", "autoscaling", "task-queue"]
draft: true
---

For a long time I didn't need autoscaling. There wasn't enough volume to justify it.

Then the volume grew, and at certain hours some queues got very long. A fixed set of machines is either too small at the peak or too big the rest of the day, and I was tired of choosing.

So on October 16 I shipped GPU autoscaling in [runqy](https://github.com/Publikey/runqy), our open-source task queue. Work arrives on a queue, the server rents a machine on Vast.ai or RunPod, and gives it back when it goes idle. I thought the hard part would be the policy: when to scale up, when to scale down.

It wasn't. The hard part is that "renting a GPU" on a marketplace means borrowing a stranger's computer, and strangers' computers are weird. Here is what a month of that taught me.

## My tests were green and the API said 404

Before spending a cent, I checked my Vast.ai code against the live API. The offer search called an endpoint that answers 404. Every single rental would have failed.

The unit tests passed, of course. I had written a fake Vast.ai server for them, and my fake server made exactly the same mistake as my code. I had tested my own misunderstanding against itself, and it agreed with me.

The GPU filter was the same story in miniature. Vast's docs write `RTX_3060`. That matches nothing. Their CLI quietly turns the underscore into a space before calling the API, and the API wants `RTX 3060`.

RunPod had a quieter trap. I didn't send a volume size, so RunPod was going to create its default 20 GB persistent volume for every pod, and bill it. I only caught it because I read the API reference line by line before the first rental.

## Four rentals, four dead hosts

My first real Vast.ai rental landed on a machine in China. It never started the worker image: pulling several GB from ghcr.io is slow or blocked there. Fair enough. I added country filters and a minimum download speed (there are offers at 7 Mb/s, if you enjoy waiting).

The second landed in Japan and died on a Docker daemon write error. Not my code, not my image, just a broken host. And here is the cruel part: it was the cheapest offer. After the retry delay, my autoscaler would have looked at the market, picked the cheapest offer, and rented the same broken machine again. Cheap and broken go together more often than you'd hope. The provider now remembers where each instance landed and avoids a host whose container never started for 24 hours.

Three more rentals failed on the host side: registry blocked, Docker storage error, GPU that couldn't be attached to the container. All three were unverified hosts. Verified-only became the default that day.

RunPod was politer about it. The first rental just said "no instances currently available": the one GPU type I had asked for was out of stock. Then my CUDA version filter excluded every host on the platform, because it matches exact versions and the API rejects the versions recent hosts actually report. I removed the filter and got a card instantly. Both providers now take a list of acceptable GPUs instead of a single name, because on a marketplace "the one I want" is often "the one nobody has".

## A machine every hour, forever

The failure I was most afraid of showed up on staging first. A queue whose machines never got a working worker rented a new machine at every boot timeout. Forever. With a real provider, that is a machine you pay for on every cycle, to do nothing.

My first fix was an exponential backoff capped at one hour. It looked reasonable. It still meant one doomed machine per hour, indefinitely, for as long as nobody noticed.

The thing I had missed is that there are two very different failures hiding behind "the machine didn't boot". If the host is broken, try another one quickly. If my image is broken, it will be exactly as broken in an hour, and tomorrow, and next week. A backoff is patience, and patience doesn't fix a bad image.

So now, after five boot failures in a row, the autoscaler stops renting for that queue for 24 hours and sends an alert saying so. When I've fixed the cause, `runqy autoscale resume <queue>` wakes it up.

The marketplace kept a few more lessons for later:

- **The advertised download speed is the host's own speed test.** A host that passed my filter was too slow to finish booting, and nothing stopped the next search from picking it again. A boot timeout now blacklists the host for 24 hours.
- **The cheapest offer may not exist anymore.** The search index lags behind rentals, so an offer can stay listed after someone else took it. Renting it fails, and my code retried the same ghost. It happened twice in a row on a 48 GB queue, where offers are rare. The fix is merged and ships in the next release.
- **A destroyed machine takes its logs with it.** So the server saves them first. I started with the last 200 lines. That turned out to be one Python traceback through a deep model stack, printed twice, and nothing about what led to it.

## The queue was lying to the autoscaler

This is the part I didn't expect. An autoscaler reads signals from the queue, and building one is a very effective way to find out which of those signals were wrong all along.

**Priorities did nothing.** I found this while preparing a low-priority sub-queue. The worker built its queue list once, from a Go map, so each worker polled its sub-queues in a random order decided at startup. Redis serves the first non-empty queue in the list. The weights were never read by anything. Priorities were a documented feature, with numbers people had chosen on purpose, and none of it did anything.

**A dead process looked like a healthy queue.** When a worker's Python process crashed, the worker kept taking tasks and failed each one in milliseconds. From the outside: nothing pending, low latency, fresh heartbeat. A perfect queue. No alert fired, the autoscaler saw no reason to help, and every task quietly burnt its retries. A worker whose process is down now leaves tasks alone, so they wait, latency grows, and somebody (or something) reacts.

**Rented boxes have bad clocks.** A task could finish normally, deliver its image, and then run a second time. Lease expiry used each host's local clock, and any worker could judge any other worker's lease. A rented machine with a clock running ahead decided its neighbours' tasks had expired. Leases now use the Redis server clock, the only one everybody shares.

**Scale-down cut tasks off.** Destroying an idle machine is easy. Destroying one that picked up a task half a second ago is someone's generation failing for no reason they'll ever understand. The server now asks the worker to drain first, and destroys the machine once it says it's done.

Alerts needed the same care. A latency hovering around its threshold sent me a FIRING email, then RESOLVED, then FIRING, every few minutes. I had built a very expensive way to teach myself to ignore my inbox. An alert now resolves only after things have stayed calm for five minutes.

## What I'd tell myself a month ago

Read the provider's API reference before the first rental, and make your fake server follow the real responses. A fake that agrees with your code proves nothing.

Only destroy what you created. Runqy names every instance with a `runqy-` prefix and refuses to touch anything else, because the same account also runs production machines that were started by hand, and I would like to keep them.

Don't trust what a marketplace says about itself. Bandwidth, availability, GPU names: all of it is a claim, and you find out by renting.

And expect the autoscaler to audit everything underneath it. Most of this month went into fixing the queue, not the scaling.

The latest release, v0.4.6, is the first one that is about policy instead of damage control: a sub-queue with `priority: 0` is served only by spare capacity, and can be left out of the scale-up and idle triggers, so background work never rents a machine and never keeps one alive. It took a month to get to the feature I thought I was building on day one.

The autoscaling guide is at [runqy.com/docs](https://runqy.com/docs/guides/autoscaling/).
