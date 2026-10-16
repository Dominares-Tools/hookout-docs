# hookout — Signed webhooks out, with an SSRF guard

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/hookout**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

Send HMAC-signed webhooks to URLs your customers typed in, without letting one point at your own network, and check inbound webhooks while you rotate the secret with no outage.

## Get it

Buy hookout at https://dominares.org/tools/hookout: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python package and the hookout command, a subscribers-file sample, an example receiver with rotation, and the full test suite.

Requirements: Python 3.10 or newer, standard library only.

This page documents version 1.0.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

**Send.** You keep a file of subscribers (URL, the events they want, a secret each). `deliver` builds one compact JSON body, `{"event": ..., "data": ...}`, signs those exact bytes with each subscriber's own secret (HMAC-SHA256, hex, in the `X-Hookout-Signature` header) and POSTs it. The receiver recomputes the HMAC over the bytes it got. One failing subscriber never stops the others, and `deliver` never raises; it returns counts and a per-subscriber status (`200`, `500`, `"blocked"`, `"unresolvable"`, `"error"`), never a secret.

**Guard.** A subscriber URL is something an outside party typed into your system, so it is checked twice, when it is registered and again just before every send (a name that was public yesterday can point inside today). Refused: anything but http(s); `localhost`, `*.localhost`, `*.internal`, `*.local` and the metadata host names; and any host that is, or resolves to, an address that is not publicly routable: loopback, RFC 1918 private ranges, link-local (so the 169.254.169.254 metadata address), carrier-grade NAT, multicast, reserved, unspecified, and IPv4 wrapped in IPv6. Decimal, octal and hex spellings of an IP are caught because the host is resolved rather than pattern-matched. If a name resolves to several addresses and ANY is internal, the name is refused. A URL with a backslash, whitespace, a control character or a `user@` part is refused too: HTTP libraries disagree about which part of such a URL is the host, so a check made by one parser would not bind another. The URL that is stored, and the one handed to your `post_fn`, is rebuilt from its parsed parts (`hookout.canonical_url`: lower-case host, no user part, no fragment).

The default sender goes further than a check-then-connect: it resolves the name once, checks every address, then connects to those checked addresses in the order the system returned them, trying the next one only if a connection cannot be made (TLS still verifies the certificate against the host name), so DNS cannot change its answer between the check and the connection and an unchecked address is never contacted. It does not follow redirects; a 3xx counts as a failure.

**Inbound, with rotation.** A shared secret that is hard to change never gets changed. Every inbound check here takes a comma list, new value first: add the new secret, update the sender at leisure, then remove the old one.

- `is_valid_signature_any(secrets, body, signature)` for hookout-style signatures.
- `hookout.twilio`: `compute_signature`, `is_valid`, `is_valid_any` for Twilio-style request signatures (HMAC-SHA1 over the URL plus sorted POST parameters, standard library only; the tests pin the vendor's documented example).
- `hookout.rotation`: `credential_ok` and `matched_slot` for plain shared tokens (constant-time over every candidate, no early exit, fail closed when nothing is set, a trailing comma never makes the empty string valid); `rotation_status` (a plain-English warning while two values are live, or an informational note if you decided to keep both, with your reason); `record_slot_use` and `slot_use_summary`, which record which slot was used (the slot number and a time, never a credential) and answer "can the old value be removed yet?": only after 5 calls on the new value and none on the old.

`examples/receiver.py` is a complete standard-library receiver that does this with `HOOKOUT_SECRETS="new,old"`.

### Use it from Python

```python
import hookout

sub = hookout.register_subscriber("subs.json", url="https://partner.example/hook", events=["order.paid"])
print(sub["secret"])                      # shown once; give it to the receiver

hookout.deliver("subs.json", event="order.paid", payload={"id": 7})
# {'event': 'order.paid', 'delivered': 1, 'failed': 0, 'results': [{'id': ..., 'url': ..., 'status': 200}]}

# on the receiving side, over the raw request bytes:
hookout.is_valid_signature_any(os.environ["HOOKOUT_SECRETS"], raw_body, request.headers["X-Hookout-Signature"])
```

`deliver(..., post_fn=fn)` replaces the network with your own `fn(url, *, data, headers, timeout) -> status`, for tests or for a queue you already run. `resolver=` replaces DNS. `header=` renames the signature header. `allow_private=True` switches the address guard off, for a trusted internal network and for local tests.

### Command

```
hookout sign --secret-env NAME [--body TEXT | --body-file F]      # reads stdin by default
hookout verify --secret-env NAME --signature SIG [--body ...]     # secret may be "new,old"; exit 1 if invalid
hookout twilio --token-env NAME --url URL --signature SIG --param K=V ...
hookout check-url URL [--allow-private] [--strict]                # exit 1 if the guard would refuse it
hookout subscribers add|list|remove [--file F] [--url U --events a,b] [--id ID]
hookout deliver --event E --data '{...}' [--file F] [--dry-run] [--header NAME]
hookout rotation status --value-env NAME [--accepted --reason TEXT]
hookout rotation record|summary --value-env NAME --state FILE [--presented P] [--label L]
```

`--secret`, `--token` and `--value` also work but are visible in shell history and process lists; prefer the `-env` forms. The subscribers file defaults to `hookout-subscribers.json` in the current folder. `deliver` exits 1 if any delivery failed; `--dry-run` always exits 0. `subscribers add` prints the new secret once, to stdout.

### Limits

- **Not a queue.** One attempt per subscriber, in order, with a 10-second socket timeout on each step (connect, send, read), which is not a deadline for the whole call. No retries, no backoff, no dead-letter store, no parallelism: a slow subscriber delays the ones after it. Put a retry or a queue around `deliver` if you need one (`post_fn` is the hook).
- **No replay protection in the signature.** The signature covers the body only. A receiver cannot tell a replay from a first delivery unless you put a timestamp or an id in `payload` and check it.
- **Pinning belongs to the default sender.** With your own `post_fn`, only the pre-send URL check applies. That check resolves the name itself; a library that resolves it again can get a different answer (DNS rebinding), so use the default sender when that matters. The default sender uses the addresses DNS returned (all checked), in order, and ignores proxy settings.
- **The file is for one writer.** `register_subscriber` and `remove_subscriber` read, change and rewrite the whole file with no lock; two processes doing it at once can lose a change. Writes are atomic, so a crash never leaves half a file. A file that exists but is not valid (a stray comma after a hand edit) is never overwritten: `register_subscriber` and `remove_subscriber` raise `SubscribersFileError` and `hookout subscribers add|remove` exit 2, while `deliver` and `list` read it as empty. Secrets are stored in the clear (the file is mode 0600 on POSIX; Windows is untested).
- **A name that does not resolve is allowed at registration** (it is unreachable anyway, and the send-time check looks again). `check-url --strict` and `deliver` refuse it (status `"unresolvable"`); only `deliver --dry-run` lets it through, so a dry run cannot tell you whether DNS will work.
- **Twilio-style checks** cover the form-parameter signature, with one value per key. The variant that signs a JSON body hash is not implemented.
- **Rotation state is yours to keep.** The functions take and update a plain dict; `hookout rotation record` keeps it in a JSON file, unlocked. Nothing records slot use for you inside your own request handler: call `record_slot_use` there.
- The guard judges the address you are about to contact, not what the far side answers. Standard library only, Python 3.10 or later.

### Tests

```bash
python3 -m pytest
```

The suite sends real signed requests to local servers (and a local TLS server when the `openssl` command is present).

---

Docs mirror of the private repository Dominares-Tools/hookout; updated with each release.
