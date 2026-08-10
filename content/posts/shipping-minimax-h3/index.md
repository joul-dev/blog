---
title: "Shipping MiniMax H3 three days after the weights dropped"
date: 2026-08-07
description: "Riding an unmerged diffusers PR, converting ComfyUI int8 checkpoints, and teaching a task queue that a video job is not an image job."
tags: ["ai-infrastructure", "video-generation", "diffusers", "runqy", "imference"]
cover:
  image: "minimax-h3.webp"
  alt: "A frame from a MiniMax H3 video generated on imference"
---

MiniMax released the open weights for H3, their omni-modal video model, on August 3. By August 6 it was running in production on our own GPUs — text-to-video and image-to-video, with joint audio, served through [gen-image](https://gen-image.com), the [imference API](https://imference.com) and Imference Desktop. We're two developers working on this part-time.

This post is about the unglamorous middle: what it actually takes to serve a frontier video model three days after the weights drop, when you don't write custom kernels and don't have a lab's GPU budget.

## Riding an unmerged PR

The honest version of "we use Hugging Face diffusers" is: H3 support lives in [diffusers PR #14355](https://github.com/huggingface/diffusers/pull/14355), which is not merged and not released. We pin the exact commit we validated. The PR imports `torch.nn.functional.ScalingType`, which only exists since torch 2.10 — so the stack is torch 2.11.0+cu128, torchao 0.18, and a diffusers build that officially doesn't exist yet.

The price of that speed: the H3 worker can't share a venv with the rest of our fleet, which is pinned to diffusers 0.39.0. Dedicated pod, dedicated stack, and we'll converge when the PR ships in a release. We don't write inference kernels — we ride Hugging Face's work while it's still warm. That's the whole strategy, and the commit-level pin is what it costs.

## The actual three days: converting checkpoints

Here's what nobody tells you about day-3 support: the weights that exist aren't the weights you need.

The only usable int8 quantization of H3 was published by Comfy-Org in their ConvRot format — block Hadamard rotations with per-channel scales, laid out for ComfyUI. Diffusers expects a different world. So the real work of the three days was a converter: de-quantize ConvRot, re-serialize as torchao int8, resynthesize the audio VAE's `weight_g`/`weight_v` parameters (the published file has weight_norm fused), and rebuild the whole thing as a diffusers model tree.

My favorite detail: H3's text encoder is a Qwen3-VL truncated to 50 layers, but the loader guard insists on more than 50. The converter synthesizes a fake 51st layer — norms at 1, projections near zero — purely to walk past the check. It does nothing. It exists to satisfy an assertion.

The output is a ~63 GB tree across 27 files, verified against the original safetensors headers via HTTP range requests — you can check tensor shapes and dtypes without downloading 63 GB, which felt like cheating in the best way. We pushed the tree to our own R2 mirror; production pods run with `HF_HUB_OFFLINE=1` and never touch Hugging Face at runtime.

Was int8 worth the trouble? We A/B'd it against bf16 on the same seed: the frames are visually identical. int8 is the production profile, no debate. (int4 was attempted and abandoned — torchao 0.18 removed its old int4 path and the new one depends on a library that isn't published yet. Comfy's "pruned" bf16 variant turned out to be a modified architecture, low-rank AdaLN and all — not convertible, rejected.)

## What it runs on

The surprise of H3 isn't the VRAM. With block-level offload it peaks around 21 GB; with leaf-level offload, 10.7 GB — and since throughput is bound by memory bandwidth rather than compute, the aggressive offload barely costs you. A 12-16 GB gaming card can run this model.

The floor isn't VRAM. It's host RAM: you need roughly 75 GB of it to hold the weights for offloading. That single number explains most of our deployment decisions — including why Desktop gets H3 as a cloud model (more on that below).

We validated on a rented RTX 6000 Ada (48 GB, ~14.9 s/step at SD resolution) and serve production on RTX 5090 pods (~8.1 s/step — nearly 2× the Ada, GDDR7 bandwidth doing the work). Our wider fleet spans A4000s to 5090s, which forced one careful choice: CUDA 12.8 is the last channel whose builds cover the entire range in a single image, Turing through Blackwell. torch 2.12 dropped it. So `pytorch:2.11.0-cuda12.8` is, as far as we can tell, the newest "runs everywhere" combo that exists.

One deployment lesson worth passing on: our GPU preflight used to check the device's compute capability against a supported list. A 4090 passed the check and then failed in ways the metadata said were impossible. The fix was embarrassingly simple — run a kernel, add 1 to `zeros(8)`, assert the sum is 8. Test behavior, not metadata.

## Latency, honestly

Real numbers, RTX 5090, 6.58-second clip at 24 fps, fixed seed: SD (960×544) at 20 steps takes about 4 minutes. HD (native 1344×768) takes about 9. HD at 30 steps takes 14 — and was rejected, because in a blind A/B I couldn't see the difference from 20 steps. Attention cost scales with the square of the token count, so resolution is brutally expensive.

This is why we serve 540p and 720p, and nothing above: we don't have the GPU budget to serve more at a latency we'd accept. SD is the default; HD is opt-in and priced accordingly. (The 2K module never entered the discussion — MiniMax kept it API-only.) Cold start on a fresh pod is 2 min 15 — 63 GB pulled from the CDN mirror plus load.

One operational footnote: queue timeouts tuned for image jobs don't survive contact with video. A ceiling that's generous for an SDXL render is fatal for a nine-minute clip — our video rails now run with a much longer active timeout than the image ones. A video job is a different animal, and every layer of the stack needs to know it.

## One engine, one queue, three products

H3 now runs on the same rail as everything else we serve: one inference engine, one queue ([runqy](https://github.com/Publikey/runqy), our open-source Go task queue), fanned out to gen-image, the imference API, and Imference Desktop. On Desktop it's a cloud model — the local stack stays on stable diffusers, and 75 GB of host RAM is not a laptop.

One more first for us, almost in passing: H3 generates audio jointly with the video, in the same DiT — the mp4 comes out with AAC muxed in. It's the first audio+video rail in the house, and it cost us nothing extra to ship.

Three days, one converter, one fake layer. The model is live — go make something with it.