# Changelog

## 1.0.1

Fixes from an independent review.

- **Security: URLs that parsers read differently.** `check_url` now refuses a URL with a backslash,
  whitespace, a control character or a `user@` part, and `register_subscriber` stores a canonical
  rebuild of the URL (`canonical_url`, new). Before, a URL such as `http://127.0.0.1:8811\@example.com/`
  passed the guard (read as host `example.com`) and an HTTP library used through `post_fn` could send it
  to `127.0.0.1`. `deliver` also hands `post_fn` the canonical form. The default sender was not affected.
- **Address order.** The default resolver keeps the order the system returned (it was sorted as text), and
  the default sender moves on to the next checked address when a connection cannot be made. It still never
  contacts an address that was not checked.
- **A damaged subscribers file is no longer overwritten.** `register_subscriber` and `remove_subscriber`
  raise `SubscribersFileError` (and `hookout subscribers add|remove` exit 2) when the file exists but is not
  valid; `deliver` and `list_subscribers` still read it as empty. An empty file counts as no subscribers.
- **`deliver` reports `"unresolvable"`** for a host name with no address (it was `"blocked"`), and checks
  resolution before calling `post_fn`. New keyword `allow_unresolvable` (the CLI's `--dry-run` sets it).
- A subscriber whose `events` is not a list is skipped (it matched by substring). The example receiver binds
  to 127.0.0.1. The README's timeout wording is exact.
- Tests: the no-early-exit property of the signature and Twilio slot loops is now pinned.

## 1.0.0

First release.

- **Sign and deliver.** `sign_payload`, `is_valid_signature`, `build_body`, `deliver`: HMAC-SHA256 of the exact
  bytes posted, one secret per subscriber, one failing subscriber never stops the rest.
- **SSRF guard.** `check_url` / `is_safe_subscriber_url`: refuses non-http(s), internal host names and any
  host that is or resolves to a non-public address (loopback, private, link-local and the cloud metadata
  address, carrier-grade NAT, multicast, reserved, IPv4-in-IPv6). It runs at registration AND again at
  send time. The default sender resolves once, checks every address, connects to the checked address and
  does not follow redirects.
- **Subscribers file.** `register_subscriber`, `list_subscribers`, `remove_subscriber`: one JSON file,
  written atomically, mode 0600; the secret is returned once.
- **Inbound checks with rotation.** `is_valid_signature_any` and `hookout.twilio` (`compute_signature`,
  `is_valid`, `is_valid_any`) accept a comma list of secrets; `hookout.rotation` (`accepted_values`,
  `credential_ok`, `matched_slot`, `rotation_status`, `record_slot_use`, `slot_use_summary`) says in plain
  words whether the old value can be removed.
- **`hookout` command** with `sign`, `verify`, `twilio`, `check-url`, `subscribers`, `deliver` (with
  `--dry-run`) and `rotation`.
