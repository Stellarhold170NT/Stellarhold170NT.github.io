---
title: "vanguard-spec: Locking down architectural invariants before line one"
date: 2026-11-03 12:00:00
categories:
  - Architecture
tags:
  - Software Architecture
  - Spec-Driven Development
  - AI Engineering
author: Huy Nguyen
---

I used to be skeptical of spec-driven development with AI.

In most teams, the workflow looks the same: prompt an LLM with a product requirement, wait ten seconds, and get back a 30-page markdown document. It has every architectural heading imaginable, but nobody actually reads it. You jump straight from an initial prompt to an exhaustive document, skipping the deliberate pushback where design risk actually gets eliminated.

Things changed for me when I started using Dave Rensin's Elephant-Goldfish Model (EGM).

<!-- more -->

### The Elephant and the Goldfish

The concept behind EGM is straightforward:

1. **The Elephant** is a long-context session with the model. It holds the entire history: architecture trade-offs, constraints, and why alternative paths were rejected. It doesn't write implementation code; its job is to interview you, push back on your assumptions, and record decisions.
2. **The Goldfish** is a stateless session with zero prior memory. You hand it only the finalized design document. If the goldfish misunderstands how a subsystem works, the problem isn't the model—the document simply failed to explain itself.

Code is only written once a goldfish can accurately reconstruct the implementation plan from the document alone. As Dave put it: *“Design is the new code.”* When models generate code faster than humans can review it, the primary artifact you need to govern is the design.

### Where specs fail over time

EGM solves the question: *is this document clear enough to build from?*

However, in any active codebase, you quickly run into a harder problem: **how do you know when a document stops being true?**

Consider a repository with 50 markdown specifications and a dozen engineers shipping daily. Code in `src/payments/` changes thirty times across three sprints. Does `docs/payments-spec.md` still reflect reality? Who owns it? When did someone last verify it against the implementation?

Git doesn't solve this. Git records that both files were touched in the commit log, but it has no semantic awareness that one file describes the invariants of the other. Proximity in the file tree creates a false sense of safety.

Left unchecked, specifications become monuments to things that were once true but are no longer.

### Invariant tracking with Vanguard

I built `vanguard` to address this single failure mode deterministically: **identify which documents are stale, and trace the exact commits that caused the drift.**

There is no LLM inside `vanguard`. It operates strictly as a state machine tracking file paths and commit hashes.

Each specification declares what it watches in its frontmatter:

```yaml
---
id: payments-spec
watches:
  paths: ["src/payments/**"]
  review_after: 90d
---
```

When an engineer reviews the document against the implementation and confirms it is accurate, they attest to it:

```bash
vanguard observe payments-spec
```

This writes an attestation entry into a lockfile (`vanguard.lock`), tied to the current commit hash. If files matching `src/payments/**` change later in Git, or if 90 days elapse without re-attestation, `vanguard check` flags the document as stale and outputs the exact commits responsible.

```bash
vanguard check --path src/payments --format json
```

This gives CI pipelines and AI agents a machine-readable signal: before an agent or developer touches `src/payments`, it can verify whether the architectural contract is current.

### Freshness is not correctness

It is important to be precise about what this mechanism does and does not guarantee:

* **Fresh** does not mean *correct*. It means a human verified the document, and nothing in the watched paths has changed since that commit.
* **Stale** does not mean *broken*. The underlying code change might have been an internal refactor that didn't violate the spec. Stale simply means the premise of the last human review has changed, requiring human attention.

The tool does not judge semantics; it manages accountability. It forces doc-to-code drift to appear as an explicit Git diff, where it cannot rot silently in the background.

In practice, this shifts how day-to-day maintenance actually feels. When a document is flagged stale, nobody is blamed, and no alarmist alert fires. The tool simply tells you: *here is what changed, and here is when a human last verified it.* 

You open a session, walk through the recent diffs, patch the document where intent has drifted, and run `observe` again. The updated lockfile is committed alongside the changes, and you move on with confidence.

If we accept that design is the new code, we also have to accept the operational reality that comes with it: code rots in days, but design rots in silence. Keeping architectural invariants honest shouldn't depend on discipline or memory—it should simply be another check that passes before you write line one.
