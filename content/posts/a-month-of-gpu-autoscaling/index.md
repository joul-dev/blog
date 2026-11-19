---
title: "My autoscaler rented a machine every hour, forever"
date: 2026-11-19
description: "A month of renting GPUs automatically on Vast.ai and RunPod: hosts that never boot, offers that no longer exist, alerts that ping-pong, and queue priorities that did nothing."
tags: ["ai-infrastructure", "runqy", "gpu", "autoscaling", "task-queue"]
draft: true
---

On October 16 I shipped GPU autoscaling in [runqy](https://github.com/Publikey/runqy), our open-source task queue: when work arrives on a queue, the server rents a machine on Vast.ai or RunPod, and gives it back when it goes idle. Two weeks into production, one queue had rented 24 machines in a row that never ran a single task. Each one was billed for a full boot timeout.

This post is the list of things the marketplace taught me between v0.3.0 and v0.4.6. Most of them are not about scaling policy. They are about what "renting a GPU" actually means when the host is a stranger's computer.

TODO(Joul): one or two sentences on why we needed autoscaling in the first place (what the fleet looked like before, what was costing too much or waiting too long).

## The tests passed, the API returned 404

Before the first real rental I checked the Vast.ai provider against the live API. The offer search called an endpoint that answers 404. Every provisioning attempt would have failed. The unit tests were green, because the fake server in the tests reproduced the same wrong endpoints as the code.

The GPU filter had the same kind of problem: `RTX_3060`, the spelling shown in Vast's docs, matches no offer. Their CLI converts the underscore client-side. The API wants `RTX 3060`.

RunPod had its own surprise. I did not send a volume size, so RunPod would have created its default 20 GB persistent volume for every pod, and billed it. A hidden default on a billing API is the kind of thing you only find by reading the reference line by line before spending money.

## Four rentals, four dead hosts

The first real Vast.ai rental landed on a machine in China that never started the worker image. Pulling several GB from ghcr.io is slow or blocked there. I added country filters and a minimum download speed (offers down to 7 Mb/s exist).

The second landed in Japan and failed with a Docker daemon write error on the host. It was also the cheapest matching offer, so after the backoff the autoscaler would have rented the same broken machine again. The provider now remembers which machine each instance landed on, and excludes a host whose container never started for 24 hours.

Three more rentals on unverified hosts failed on the host side: registry blocked, Docker storage error, GPU not attachable. Verified hosts only became the default.

On RunPod, the first rental failed with "no instances currently available": the single GPU type I had configured was out of stock. Then my CUDA version filter excluded every host, because it matches exact versions and the API rejects the values recent hosts actually report. Without the filter, the same request got a card at once. Both providers now take a list of acceptable GPUs instead of one name.

## A machine every hour, forever

On staging I saw the obvious failure first: a queue whose machines never got a ready worker rented a new one every boot timeout, forever. I added an exponential backoff, capped at one hour.

That cap is the bug in the title. In production, a queue whose instances never became ready rented one machine per hour, indefinitely. 24 in a row on one queue before I looked. TODO(Joul): what was actually wrong on that queue (image, code, host class?) and roughly what it cost.

Since v0.4.4, after five boot failures in a row the autoscaler suspends provisioning for that queue for 24 hours and says so in an alert. `runqy autoscale resume <queue>` lifts it once the cause is fixed. A backoff is the right answer to a provider hiccup. It is the wrong answer to a broken image, which will be exactly as broken in an hour.

Two related lessons from the same weeks:

- **The advertised download speed is the host's own speed test.** A host passing my filter was too slow in practice to finish booting, and nothing stopped the next search from renting it again. A boot timeout now blacklists the host for 24 hours too.
- **The search index lags behind rentals.** The cheapest offer can stay listed after someone else took it. Renting it fails, and the next attempt picked the same dead offer again. This hit twice in a row on a 48 GB queue, where offers are few. The fix (skip the offer, try the next one in the same call) is merged and ships in the next release.

I also had to make failures readable. A failed machine is destroyed, and its logs with it, so the server now saves them first. 200 lines turned out to be too few: one Python traceback through a deep model stack is about a hundred lines, and it was printed twice.

## The queue was lying to the autoscaler

An autoscaler is only as good as the signals it reads, and several of mine were wrong.

**Priorities did nothing.** I found this while preparing a low-priority sub-queue. The worker built its queue list once from a Go map, so each worker polled its sub-queues in a random order fixed at startup. `BRPOP` serves the first non-empty key, so the weights had no effect at all. Priorities had been a documented feature the whole time. They are now redrawn for every task, with an optional strict mode.

**A dead process looked like a healthy queue.** When the Python process of a worker crashed, the worker kept taking tasks and failed each one in milliseconds. Nothing pending, low latency, fresh heartbeat: no alert fired, and the autoscaler saw no reason to rent a replacement. Meanwhile every task burnt its retries. A worker whose process is down now leaves tasks pending, so latency grows and the triggers react.

**Rented boxes have bad clocks.** A task could finish normally, deliver its image, then run a second time. Lease expiry used each host's local clock, and any worker could judge another worker's lease. A rented machine whose clock ran ahead moved running tasks to retry. Leases now use the Redis server clock, the one clock every worker shares.

**Scale-down cut tasks off.** Destroying an idle machine is easy. Destroying one that just picked up a task is a failed generation for a user. Since v0.4.0 the server drains the worker first and destroys the machine once it reports drained.

Alerts needed the same treatment. A latency hovering around its threshold sent a FIRING / RESOLVED pair every few minutes. A firing alert now resolves only after its condition has stayed clear for five minutes.

## What I'd tell myself a month ago

- Read the provider's API reference before the first rental, and make the test doubles follow the real response shapes. A fake server that agrees with your code proves nothing.
- Only destroy what you created. Runqy names every instance with a `runqy-` prefix and refuses to destroy anything else, because the same account also runs machines I started by hand.
- Separate "the host is broken" from "my image is broken". The first deserves a quick retry on another machine. The second deserves a full stop.
- Never trust a marketplace's metadata about itself: advertised bandwidth, listed offers, GPU names.

The last release, v0.4.6, is the first one that is about policy instead of damage control: a sub-queue with `priority: 0` is served only by spare capacity, and can be left out of the scale-up and idle triggers, so background work never rents a machine and never keeps one alive.

TODO(Joul): a closing number if you have one. How many machines the autoscaler rents on a normal day now, or what the GPU bill looks like compared to before.

The autoscaling guide is at [runqy.com/docs](https://runqy.com/docs/guides/autoscaling/).
