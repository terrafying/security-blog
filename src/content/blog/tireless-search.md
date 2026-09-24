---
title: "Tireless search beats cleverness: a lab for auditing self-hosted AI"
description: "How a one-person audit lab runs pattern-first fan-out across self-hosted AI infrastructure, and what the first month found."
pubDate: 22 Sep 2026
---

Most security work I respect starts with the same confession: the bug was
obvious once you saw it. The hard part is not the exploit. The hard part is
covering a surface big enough that "obvious" hides in plain sight.

This post describes the lab I built to do that for self-hosted AI
infrastructure, and what the first month produced.

## The premise

Brutecat's writeup on hacking Google with AI ($500,000 in bounties, 1,500
APIs, 3,600 keys) put the model in one sentence: the AI's job was not to be
novel, it was to be tireless about the obvious on a surface too large for a
human to cover. Most Google bugs didn't need clever exploitation, just
patience, because the same broken patterns showed up everywhere - missing IAM
checks on cross-tenant resources, GraphQL schemas with no authorization, debug
endpoints in prod.

Self-hosted AI infrastructure is that thesis with the surface growing faster
than the research. Looped-reasoner sandboxes, agent runtimes, model
orchestration platforms, OpenAI-compatible gateways: enormous install bases,
active exploit crews, and almost nobody doing the audit work - because
everyone with the skills is busy on the frontier labs instead.

## The lab

One rule drives the design: findings must be live-verified or honestly
rebutted, never claimed from reading code alone. Everything runs against
local Docker instances on a three-Mac cluster; nothing touches third-party
systems; disclosure is coordinated before anything publishes.

The pipeline is three stages:

**Recon.** Pull the recent CVEs in a family, name the root-cause class for
each, judge whether the fix was complete. Pan-OS's proxy-layer auth desync,
Fortinet's bypass-of-bypass SSO chain, SonicWall's license-sync patch gap,
Zyxel's per-feature command injection without a centralized escaping layer -
each of these is a *pattern*, and patterns generalize. LiteLLM turned out to
be the Python-ecosystem twin of the edge-gear auth-desync family, which made
it a target without any new CVE needing to exist.

**Fan-out.** Parallel agents each take one target with a fixed protocol:
spec-harvest the full API surface before probing (routes, middleware
inventory, the unwrapped routes list), grep for the five broken patterns by
name, dedupe against the advisory database, then verify the best candidates
live against a local instance. Write findings by minute 30 even if
incomplete - the file is the deliverable, not the chat transcript.

**Verification discipline.** Every claimed finding gets independently
reproduced or explicitly rebutted before it enters the ledger. This round,
MLflow's "auth bypass in the FastAPI artifact router" died to a proper live
test (the middleware fires under gunicorn; 65 probes, all 401), and a
claimed downgrade-to-RCE chain in ComfyUI shrank to an install-breaking DoS
once I ran it. Honest negatives are as valuable as findings: they say where
not to spend time.

## What the first month found

Audited: 27 targets - agent runtimes (n8n, Langflow, ComfyUI, LocalAI,
AnythingLLM), inference servers (vLLM, Ollama-adjacent, Ray), gateways
(LiteLLM), password-manager-adjacent (Vaultwarden), wallet infrastructure
(Trezor, Electrum-class), and the MCP registry ecosystem.

Confirmed, with live proof: a sandbox escape in n8n's task runner that
defeats secure mode; unauthenticated job-submission RCE on Ray's current
release; arbitrary unsandboxed code execution via Langflow's authenticated
build path on PUBLIC flows; a client-settable frame-count parameter that
bypasses vLLM's fix for the unbounded-video-decode class that paid a $15k
bounty; and an MCP blocklist bypass in Desktop Commander verified end to end
through the stdio protocol.

Rebutted with evidence: MLflow's auth middleware, LibreChat's IDOR surface,
Flowise at HEAD, ComfyUI's downgrade chain, LiteLLM's post-CVE hardening.
Every rebuttal is as documented as the findings, because "this vendor is
actually solid" is worth knowing too.

## The lesson that generalizes

The recurring finding across every vulnerable target was the same shape
Brutecat found at Google: a security control that exists on one surface and
is absent on its sibling. Langflow hardened its unauthenticated build path
after a KEV-listed RCE and left the authenticated path unguarded - on
instances with auto-login or open signup, that guard is meaningless. LocalAI
registers feature middlewares that degrade to no-ops when auth isn't
configured. ComfyUI-Manager gates 14 of 46 routes and leaves 32 open. The
control-exists-on-one-surface pattern is greppable, which is why it goes into
every sweep before any clever work.

None of this required a novel technique. It required a lab where the obvious
checks run everywhere, every claim gets verified against a running instance,
and the ledger records what didn't work as faithfully as what did.

## Reproduce it

The lab, findings, and every repro script are public:
github.com/terrafying/fractal-basins-lab and the audit lab repos. Local
instances only, coordinated disclosure only, detect-and-disclose only.

## Related work

What the method found: [three live proofs](/blog/three-live-proofs/) from
the first audit round, and [the Plex patch
reconstruction](/blog/plex-undisclosed-fixes/) for the closed-source side
of the same discipline. PoC scripts are collected at
[github.com/terrafying/pocolate](https://github.com/terrafying/pocolate).