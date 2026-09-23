---
title: Reconstructing undisclosed Plex fixes by behavioral diff
description: Plex patched multiple undisclosed vulnerabilities in 1.43.3 with no CVEs and no technical detail. We diffed the binaries and verified the fix behavior against a live instance.
pubDate: 23 Sep 2026
---

On September 1, Plex released Plex Media Server 1.43.3 and Plex Desktop
1.115.0 to address "a number of security issues," urging all users to update
immediately. CVEs were requested. No identifiers, no CVSS, no technical
detail, no exploitation status - just a patch and an unusual all-user email.

Three weeks later the CVEs are still unpublished. So we reconstructed what
the fixes do, ourselves.

## Method

Plex is closed source, so this was behavioral diff plus binary symbol
analysis, not source review:

1. Obtain both versions (1.43.2.10687 and 1.43.3.10896) from Plex's public
   download endpoints.
2. String/symbol diff across the two main binaries. Whole-package rebuild
   churn means file-hash triage localizes nothing, but release notes named
   two fix identifiers: PM-5763 (CompanionProxy) and PM-5766 (transcoder
   preferences).
3. Behavioral verification against live local instances of both versions,
   authenticated as local admin via the newer `.LocalAdminToken` mechanism.

![Plex pref modification, pre-fix vs post-fix](/demos/plex-prefs.gif)

## What the fixes do

### PM-5766: transcoder preference injection, closed by capability removal

On 1.43.2, an authenticated principal can set transcoder options over HTTP:

```
PUT /:/prefs?TranscoderH264OptionsOverride=AUDITCHANGE
→ 200, Setting value="AUDITCHANGE" (verified from loopback and LAN)
```

These options feed ffmpeg arguments. Unvalidated transcoder option injection
is a classic path to file read/write and RCE through crafted ffmpeg
arguments. On 1.43.3 the identical request - valid admin token, loopback -
returns 403, and the capability is simply gone: the previously injected
value stays frozen in the settings, unchangeable through the API. The fix
does not distinguish network from local; the endpoint refuses writes for
everyone, which is the strongest form of this fix.

### PM-5763: CompanionProxy

The Companion symbol surface is byte-identical between versions - the
reachable handler is `CompanionProxyRequestHandler::handleControllerPoll`,
an EndpointRouter-mounted handler in the main binary. No new routes, no new
error strings, so the fix is silent validation logic inside compiled code.
Full decompilation diff is the follow-up.

## Caveats

- Unauthenticated probes are identical across versions: the fixes are not
  reachable without a valid token, so the pre-auth attack surface is
  unchanged and the vulns require authentication. That matters less than it
  sounds given the deployment profile (media servers on home LANs,
  shared-machine tokens, and 36,000+ exposed instances per BleepingComputer).
- We have not confirmed the ffmpeg-argument-injection chain end to end;
  the pref-to-ffmpeg seam is the candidate RCE path and the decompiler work
  is pending.

## Disclosure

Coordinated with Plex (security@plex.tv) per their disclosure policy. This
post publishes after the advisory window. Their program is discretionary
and their policy is private-only disclosure - respect it.

## Takeaway

You do not need a CVE number, a writeup, or a PoC to audit a fix. A patch
diff plus a behavioral test against two live versions reconstructs most of
what the vendor withheld - and tells you whether the fix is complete before
the CVEs land.
