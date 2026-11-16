---
title: "Reproducing BaxBench in Inspect: half the working backends LLMs write are exploitable"
description: >-
  I ported BaxBench, a benchmark of whether LLMs write backends that are both correct and
  secure, to the Inspect evaluation framework, reproduced the paper's results, and found three
  bugs that only show up at scale.
categories: [AI Evaluation, Security]
tags: [inspect, baxbench, llm, security, benchmarks, reproducibility]
---

Ask a language model for a small backend, such as a login service, a wiki or a file search
API, and it will usually give you something that runs. Whether that code is safe to deploy is
a different question. [BaxBench](https://arxiv.org/abs/2502.11844) (Vero et al., ICML 2025)
measures both. It asks a model to write a complete backend, builds it in Docker, runs
functional tests against it, and then attacks it with real exploits: SQL injection, path
traversal, cross-site scripting, resource exhaustion and more.

I implemented BaxBench for [Inspect](https://inspect.aisi.org.uk/), the evaluation framework
from the UK AI Security Institute, and reran it on two of the paper's models. The headline
result held up: **about half of the backends that pass every functional test can still be
exploited.** This post covers what I built, how closely it reproduces the paper, and the
three bugs I hit along the way, one of which was quietly making models look more secure than
they are.

The code is at [Jeffrin-J/inspect-baxbench](https://github.com/Jeffrin-J/inspect-baxbench),
and it is now in the
[Inspect Evals register](https://ukgovernmentbeis.github.io/inspect_evals/evals/baxbench/).

## What BaxBench measures

BaxBench has 392 tasks: 28 scenarios across 14 frameworks in six languages (Python,
JavaScript, Go, PHP, Ruby and Rust). Each scenario describes an API, usually as an OpenAPI
schema, and the model has to produce the whole application in one response.

Each answer is scored in two stages:

1. **Is it correct?** The app is built into a Docker image and started, and functional tests
   check that every endpoint behaves as specified.
2. **Is it secure?** Exploits written for that scenario attack the running app. Each
   successful attack is recorded as a CWE (Common Weakness Enumeration).

An app counts as *secure* only if it is correct and no exploit succeeds. An insecure app
that doesn't work is not a security problem, because nobody would deploy it.

## Porting it to Inspect

The upstream tests are Python functions that run on the host against a running container.
They use the Docker SDK directly to read files, run SQL and check memory usage inside the
container. Generated code is also processed while the image is built: Go runs `goimports`,
Rust compiles, and Rails runs migrations.

That rules out Inspect's usual sandbox, which starts a container before the model has
written anything. Instead, the scorer does what upstream does: one image per answer, a fresh
container for every test, a 300-second timeout per test, a 1 GB memory limit, and CWE-703
when a test crashes the server.

One improvement came from the register review: the test containers now publish their port
on 127.0.0.1 only, where upstream publishes it on every network interface. The apps under
test are often exploitable by design, so there's no reason for anything outside the machine
to reach them.

I wanted the scores to mean the same thing as the paper's, so the implementation:

- reuses upstream's functional tests and exploits unchanged, downloaded at a pinned commit;
- has scripts that check its prompts and code extraction match upstream's exactly;
- has hand-written reference apps for all 14 frameworks, with known vulnerabilities, so
  every framework's build and test path is exercised before any model is involved.

## Benchmarks rot

The first surprise was that upstream BaxBench no longer builds. Nothing was wrong with its
code; the world around it moved:

- **Debian 11 (bullseye) reached end of life in August 2026.** Every BaxBench image is based
  on it, and its packages are leaving the main mirrors, so `apt-get install` fails. Pointing
  apt at `archive.debian.org` serves the exact same package versions.
- **Unpinned build dependencies moved on.** Without a `Cargo.lock`, Rust crates now resolve
  to versions that need a newer compiler than the image's Rust 1.83. `gem install rails`
  now installs Rails 8.1, which conflicts with the Gemfile's 8.0. The `json` gem's 3.x
  release rejects an option Rails 8.0 passes when parsing request bodies.

Each fix pins the newest version that still works with the image's toolchain, and none of
them changes what the model is asked to do. I've
[reported these upstream](https://github.com/logic-star-ai/baxbench/issues/24).

The general lesson is that a benchmark is not just its tasks and tests, it is also every
package mirror and registry its build reaches out to. Anything not pinned will eventually
break, and it breaks quietly: a build failure scores as an incorrect answer, so a broken
environment looks like a weak model.

## Results

I ran all 392 tasks with DeepSeek V3 and Llama 3.3 70B through OpenRouter, using upstream's
default settings: the API described as an OpenAPI schema, and no security reminder in the
prompt.

| Model | Correct (ours) | Correct (paper) | Secure (ours) | Secure (paper) |
|---|---|---|---|---|
| DeepSeek V3 | 39.5% (34.8–44.5) | 40.3% | 19.1% (15.5–23.3) | 19.4% |
| Llama 3.3 70B | 18.1% (14.6–22.2) | 22.0% | 8.9% (6.5–12.2) | 9.2% |

The ranges are 95% Wilson confidence intervals. The paper's number falls inside them for both
metrics and both models, and in 55 of 56 per-framework comparisons. With 56 comparisons at
95%, you'd expect about three misses by chance alone.

The paper only publishes its results as bar charts, so I extracted the numbers from the bar
heights in the vector figures in the arXiv source. That is exact enough to compare at this
level of detail.

### Half the working apps are exploitable

Of the backends that passed every functional test, **52% of DeepSeek V3's and 51% of
Llama 3.3 70B's** were broken by at least one exploit. The most common weaknesses among
working apps were:

| Weakness | DeepSeek V3 | Llama 3.3 70B |
|---|---|---|
| CWE-400: uncontrolled resource consumption | 29 | 9 |
| CWE-79: cross-site scripting | 25 | 10 |
| CWE-117: log injection | 11 | 7 |
| CWE-522: insufficiently protected credentials | 9 | 5 |
| CWE-22: path traversal | 7 | 4 |
| CWE-94: code injection | 6 | 7 |

None of these are exotic. They are the mistakes that code review and security training
target, and the models make them on short, well-specified tasks.

### Llama, and why the provider matters

Llama's correctness is the weakest match, at the lower edge of its interval. The paper served
Llama from a specific FP8 endpoint that OpenRouter doesn't offer, so my requests were routed
across eight providers running different quantisations. Among the four that served at least
30 requests, correct rates ranged from **10% to 27%**.

The "same model" behind an API is not a fixed thing. If you're comparing against a published
number, record which provider and quantisation you actually got, or pin them.

## Three bugs that only showed up at scale

The reference apps all passed. The problems appeared only in full runs: hundreds of builds,
concurrent tests and hours of runtime.

### 1. A Windows file lock that hid security findings

Some exploits check how passwords are stored by copying the app's SQLite database to a
temporary directory on the host and inspecting it. On Windows, deleting that directory fails
while the database file is still open. Upstream never closes its connections, and antivirus
software opens new files too. The exception was raised *after* the check had found the
problem, so the test failed without reporting the weakness.

The effect is that the benchmark under-reports CWE-522 on Windows. That makes models look
**more secure** than they are, which is the worst direction for a security benchmark to be
wrong in. After fixing it and rescoring, 16 more cases were found for DeepSeek V3 and 6 for
Llama 3.3 70B.

### 2. A 100 GB Docker leak

Each answer builds its own image, and the scorer removed it after testing. Disk usage still
climbed by about 50 GB per full run, until the drive filled up and Docker Desktop's
containerd crashed and corrupted its image store.

The cause was intermediate layers. Concurrent builds for the same framework share layers, so
removing each final image left the shared intermediate images behind with nothing pointing at
them. The fix gives each build a unique `LABEL` right after its `FROM` line. Builds then
share no layers, and the scorer can find and remove every image a build created, final and
intermediate, including after a failed build. There is now a test that runs concurrent and
failing builds and checks that nothing is left behind.

### 3. A test that depended on the host's ffmpeg

The FrameExtract scenario's functional test ran `ffmpeg` on the host to compute the frame it
expected, then compared PNG bytes exactly. So the test needed ffmpeg installed on the host,
and a different ffmpeg version could change the expected bytes, failing correct apps. The
fix is to generate the expected frame once, with the same ffmpeg version the containers use,
and ship it with the benchmark.

## Takeaways

- **Reproduce before you extend.** Matching the paper within its confidence intervals is what
  makes the implementation trustworthy for new models. Without that check, the bugs above
  would have quietly become "results".
- **Pin everything a benchmark touches**, not just its own code: base images, package
  mirrors, compilers and transitive dependencies.
- **A security benchmark's failure modes are asymmetric.** A bug that hides vulnerabilities
  is far worse than one that invents them, so it is worth looking specifically at the paths
  where an exception can swallow a finding.
- **Report the model provider**, not just the model name.

## Try it

```bash
git clone https://github.com/Jeffrin-J/inspect-baxbench.git
cd inspect-baxbench
uv sync
uv run inspect eval src/inspect_baxbench/baxbench.py --model openai/gpt-4o -T envs=Python-Flask --epochs 1
```

That runs the 28 Flask tasks once, which is a quick and cheap check. A full run is 392 tasks
and takes several hours, mostly building Docker images. Both full runs in this post fit
within a $10 OpenRouter budget.

Thanks to the BaxBench authors for releasing the benchmark, its tests and its exploits under
an open licence, and to the Inspect team for the framework and the evals register.
