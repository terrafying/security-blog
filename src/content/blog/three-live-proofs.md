---
title: Three live proofs from the AI-infra audit round
description: Unauth RCE on Ray's current release, unsandboxed code exec in Langflow's authenticated build path, and a task-runner sandbox escape in n8n.
pubDate: 23 Sep 2026
---

The first audit round of our self-hosted AI infrastructure lab produced
three live-verified proofs. All were found by pattern, verified against
local instances, and disclosed to the affected vendors before this post.

## 1. Ray: unauthenticated job-submission RCE on the current release

Ray 2.51.2 (the current PyPI release), default config, dashboard on 8265:

```
POST /api/jobs/ {"entrypoint": "id > /tmp/canary.txt 2>&1"}
→ 200 {"job_id": "raysubmit_..."}
→ status SUCCEEDED, canary written as the ray user (admin group)
```

No credentials. The dashboard is an unauthenticated management plane for a
compute cluster - jobs API, package upload, Serve deployment, full state
API. The dashboard is documented as not-for-exposure, but Oligo counts
200,000+ exposed Ray servers, CVE-2025-62593 is CISA KEV with an active
botnet, and - the part we think is new - the browser-method fix (commit
70e7c72) exists only on master. No released Ray includes it, and the
classifier it adds (User-Agent/header-presence heuristics) is bypassable by
design: there is no Host-header validation behind it.

![Ray unauth jobs-API RCE](/demos/ray-jobs.gif)

## 2. Langflow: any tenant can execute their own code, unsandboxed

Langflow hardened its unauthenticated build path after CVE-2026-33017 (KEV,
exploited within 20 hours). The hardened surface genuinely resists bypass -
nine variants we tried all fail against hash+type component matching and
trusted-code substitution.

The authenticated path is a different story. Under default settings
(allow_custom_components=true, custom_component_admin_only=false),
`validate_flow_for_current_settings` is a no-op. Any authenticated tenant
can author a flow containing a Python Function component, mark it PUBLIC,
and execute arbitrary Python unsandboxed in the backend process:

```
POST /api/v1/flows/        (flow with code component, access_type PUBLIC)
POST /api/v1/build/{flow_id}/flow
GET  /api/v1/build/{job_id}/events
→ outputs.function_output_str: "uid=1000(user) gid=0(root) groups=0(root)"
```

On instances with auto-login or open signup - the majority of the ~7,000
publicly exposed instances per Censys - "authenticated" means anyone. The
fix shape is cheap: the gate hook exists and is dead by default.

![Langflow authenticated build-path exec](/demos/langflow-exec.gif)

## 3. n8n: task-runner sandbox escape in secure mode

n8n executes Code-node JavaScript in a node:vm sandbox ("secure mode") with
a hardening shim that freezes sandbox-realm constructors and neutralizes
reflection. The shim protects the sandbox realm. It does nothing about
host-realm objects already placed in the context:

```js
// items is the host-realm input data, spread raw into the vm context
const proc = items.constructor.constructor('return process')();
// → host process: full env, cwd, and via process.getBuiltinModule:
//   node:child_process → arbitrary code execution in the runner
```

Integration-verified against the real `JsTaskRunner` buildContext + shim
(4/4 tests, vitest). The runner's grant token is single-use and correctly
scoped, but in internal-runner mode (deprecated default in older versions)
the runner shares the main process's uid - so escape reads
`~/.n8n/config` (encryption key) and the SQLite DB.

![n8n sandbox escape integration test](/demos/n8n-escape.gif)

## The meta-lesson

None of these needed novel exploitation. Each came from the same three
questions: what does the middleware inventory look like (which routes
lack the gate their siblings have)? What did the vendor fix, and was the
fix complete? And does the deployed reality match the documented posture?
Tireless coverage of the obvious beats cleverness - on surfaces too large
for anyone to have covered.

## Related work

The method behind these three is [tireless search beats
cleverness](/blog/tireless-search/). The patch-diff lane that pairs with it
is [the Plex reconstruction](/blog/plex-undisclosed-fixes/), where
undisclosed fixes were rebuilt by behavioral diff and a live test. PoC
scripts from this round are collected at
[github.com/terrafying/pocolate](https://github.com/terrafying/pocolate).
