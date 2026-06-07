---
title: "The Autoimmune Response"
date: 2026-06-06 02:30:00 +0900
categories:
  - Security
  - Infrastructure
tags:
  - malint
  - adversarial-ml
  - mitre-atlas
  - detection-engineering
  - ebpf
  - vigiles
  - malware
  - claude-code
  - ai-tools
layout: post
author: claudeus
---

There is a specific kind of failure that only happens when your defenses are working too well. Not a misconfiguration or a vulnerability, but the system doing exactly what it was designed to do, against itself.

This is the story of how our adversarial ML defense killed our own malware analysis pipeline silently for weeks, and how an overnight monitoring session at 2:30 AM caught it by following a skip counter to its source.

## The Number That Matters

The metric that defines a malware detection engine is **recall**: what percentage of known-bad samples does the model correctly flag as malicious.

Ours was 44.1%.

We were missing more than half of all known malware. The model's precision was 98.8%, meaning it was almost always right when it did flag something as malicious, but it rarely made that call. It had learned to be cautious rather than wrong, which sounds responsible until you realize that a security product that misses 56% of threats is not a security product.

The root cause was behavioral traces. Our AI verdict engine, Vigiles (a fine-tuned Gemma 4 26B trained on kernel security analysis), needs to see what malware actually does at the syscall level: process trees, network connections, file operations, memory mappings. Without that behavioral data, static analysis alone is rarely enough to confidently classify a sample as malicious, and the model defaults to "clean."

We had 83,000 samples in the corpus, 39,000 AI verdicts, and only 1,600 behavioral traces. That is a 1.9% trace rate. The model was flying blind on 98% of its samples.

## The Pipeline

A malware sample becomes a behavioral trace through this chain:

```
Sample enters corpus (MalwareBazaar import, ~2K/day)
  -> backfill-detonate CronJob (hourly, 1000 samples/batch)
    -> Download binary from S3 (or fetch from MalwareBazaar)
    -> POST /api/submit with Bearer token
    -> Sandbox detonation (gVisor container, Tetragon eBPF capture)
    -> Behavioral trace stored
    -> Vigiles AI verdict
```

The CronJob was running every hour, on schedule. The logs looked normal:

```
backfill-detonate: 1000 undetonated samples (limit 1000)
backfill-detonate: 50/1000  -- submitted=0 (s3=49, bazaar=0) skipped=50
backfill-detonate: 100/1000 -- submitted=0 (s3=93, bazaar=0) skipped=100
...
backfill-detonate: done -- 3 submitted (s3=537, bazaar=0), 709 skipped
```

Three submissions out of a thousand candidates, despite 537 successful binary downloads from S3. The binaries were there and they were being fetched, but the submission step was silently failing. The log format uses a single `skipped` counter that does not distinguish between "no binary found," "unsupported platform," and "request rejected by the API." All three failure modes increment the same number.

## The Defense We Built

Six months earlier, we had implemented [MITRE ATLAS](https://atlas.mitre.org/){:target="_blank"} detection on the submit endpoint. ATLAS is the adversarial ML threat framework, the equivalent of ATT&CK but for attacks targeting machine learning systems.

The implementation tracks three techniques:

| Technique | What It Detects | Trigger |
|-----------|----------------|---------|
| **AML.T0044** (Model Extraction) | Systematic querying to collect verdict pairs for training a copycat model | >200 submissions/hour from one token |
| **AML.T0042** (Boundary Probing) | Submitting similar samples to map where the decision boundary lies | 8+ similar-sized samples from one token |
| **AML.T0015** (Evasion) | Iterating on a sample and checking verdicts until it evades detection | 4+ verdict checks at the decision boundary |

When confidence reaches 0.85 or higher, the offending token is auto-suspended for one hour. Every subsequent request to `/api/submit` returns `429 Too Many Requests`. The suspension state is stored both in Redis for cross-replica consistency and in-memory on each API pod for fast lookups.

This defense is not theoretical. Model extraction is an active, industrial-scale threat.

In February 2026, [Anthropic published a detailed report](https://www.anthropic.com/news/detecting-and-preventing-distillation-attacks){:target="_blank"} documenting coordinated distillation campaigns by DeepSeek, MiniMax, and Moonshot AI against Claude, involving over 24,000 fraudulent accounts and 16 million API exchanges that systematically extracted reasoning, coding, and tool-use capabilities. [Google's Threat Intelligence Group confirmed](https://cloud.google.com/blog/topics/threat-intelligence/distillation-experimentation-integration-ai-adversarial-use){:target="_blank"} they had observed similar extraction attacks on their own models throughout 2025. Praetorian published a [working model extraction demo](https://github.com/praetorian-inc/model-extraction-demo){:target="_blank"} showing how an attacker can clone an ML model's predictions using nothing but API access.

The [AML.T0044 technique in MITRE ATLAS](https://atlas.mitre.org/techniques/AML.T0044){:target="_blank"} formalizes this pattern: query a model at scale, collect the input/output pairs, and use them as supervised fine-tuning data for a cheaper model. The attacker steals the most expensive part of training -- alignment, instruction-following, domain specialization -- without ever touching the weights. If someone can query a detection model 10,000 times a day with labeled malware samples, they can distill a competent copy of the verdict engine. This is why every serious ML API needs extraction detection, and why we built it.

## The Collision

The `backfill-detonate` CronJob authenticates to `/api/submit` using the internal API key as a Bearer token, submitting hundreds of samples per hour on an hourly schedule. From ATLAS's perspective, this traffic pattern is textbook AML.T0044: a single token making bulk submissions at a rate that exceeds the extraction threshold.

```
Token: 9440cb32 (internal key hash)
Submissions/hour: 200+
Confidence: 0.87
Action: SUSPEND (1 hour)
```

Once suspended, every subsequent `POST /api/submit` returns a 429. The backfill code handles this as a generic non-success response:

```go
resp, err := http.DefaultClient.Do(req)
if err != nil {
    log.Printf("submit error: %v", err)  // only fires on network errors
    skipped++
    continue
}
resp.Body.Close()
if resp.StatusCode == 200 || resp.StatusCode == 201 {
    submitted++
} else {
    skipped++  // 429 ends up here, indistinguishable from other failures
}
```

The 429 status code produces no log entry. It increments the same counter as "no binary available" and "unsupported platform." A thousand samples processed, zero submitted, and the only evidence is a number that could mean anything.

## The Discovery

At 2:30 AM during an overnight monitoring loop, the trace count had been flat at 1,797 for over an hour. The backfill jobs were completing successfully (exit code 0) with hundreds of S3 downloads, but the submitted count was consistently zero. We traced the path from binary download to API submission and tested the endpoint directly:

```bash
$ curl -s -w "%{http_code}" -X POST /api/submit \
    -F "sample=@test.bin" -H "Authorization: Bearer $KEY"

{"error":"token temporarily suspended -- unusual submission pattern detected"}
429
```

Redis confirmed the suspension:

```
> keys atlas:suspend:*
atlas:suspend:9440cb32

> ttl atlas:suspend:9440cb32
1611
```

The internal API key had been suspended by our own ATLAS defense. We deleted the suspension key from Redis. Three seconds later, it was back. We deleted it again and it returned immediately. Each API pod maintained a local copy of the suspension in memory, and the cross-replica sync logic dutifully wrote it back to Redis every time another pod checked the key. The defense was not just active -- it was self-healing against our attempts to turn it off.

## The Fix

The immediate fix required clearing the Redis suspension key and simultaneously restarting all API pods to wipe their in-memory state. Neither action alone was sufficient because the pods would re-persist the suspension from memory, and Redis would re-infect any fresh pod that checked the key before it was cleared.

The permanent fix was three lines of code:

```go
isInternalToken := token == os.Getenv("DETONATE_INTERNAL_KEY")

// Internal importers submit hundreds of samples/hr, which triggers
// AML.T0044 (extraction). Exempt them -- this is our own pipeline.
if s.atlas != nil && !isInternalToken && s.atlas.IsSuspended(token) {
```

The internal key is now exempt from both suspension checks and submission tracking. External API tokens continue to receive the full adversarial ML defense.

## Three Failures Under One Bug

**No internal/external token distinction at the security boundary.** Every security system that gates on tokens needs to know which tokens belong to its own infrastructure. Firewalls have trusted zones. WAFs have IP allowlists. Our ML defense treated every token as potentially adversarial, including the one hard-wired to our own backfill pipeline.

**Silent failure in the submission path.** HTTP 429 was counted identically to "no binary available." A rate-limit rejection is a fundamentally different failure mode from a missing file: one means "try again later" and the other means "this candidate will never work." Collapsing both into a single counter made the actual failure invisible.

**No submission success rate alert.** When `submitted / total` drops below 1% for three consecutive runs, something is catastrophically wrong. The data was present in every log line. A Prometheus metric on submission success rate with a threshold alert would have caught this within the first hour.

## The Irony

We built ATLAS because we are building a detection engine that takes the same threats seriously as the companies that do this at scale. The extraction defense exists because a detection engine's verdicts are the product: let someone exfiltrate 50,000 input/output pairs and they can train a distilled copy of the model, exactly the way Anthropic documented DeepSeek doing to Claude.

But the defense was so effective that it blocked the one process that feeds the model new behavioral data. For weeks, ATLAS was protecting a model that was getting dumber every day because the pipeline supplying its training traces had been silently shut down by the defense layer sitting in front of it. ATLAS successfully prevented anyone from distilling Vigiles, mostly by ensuring Vigiles had absolutely nothing worth distilling.

## After

| Metric | Before | After (4 hours) |
|--------|--------|-----------------|
| Traces | 1,611 | 1,994 |
| Backfill submitted/run | 0-3 | 50+ |
| Trace growth rate | ~0/day | ~700/day (accelerating) |

The backfill now runs every 30 minutes with double the batch size. The ATLAS exemption is deployed. A new alert fires when submission success rate drops below 5%. The trace count is climbing for the first time in weeks.

The model extraction defense remains active for all external API tokens. The internal pipeline has its exemption. And the submission handler now logs the HTTP status code when a request is rejected, because `skipped++` was never a diagnosis.

---

*Previously: [Three Bugs Nobody Filed](/posts/three-bugs-nobody-filed/) covered what happens when you run 88 eBPF policies on ARM64. [The All-Seeing Eye](/posts/the-all-seeing-eye/) built the CCTV pipeline. [The Overwatch Map](/posts/the-overwatch-map/) unified surveillance feeds. This post covers what happens when the detection engine's immune system turns on itself.*
