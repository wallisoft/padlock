# Padlock

A genuinely unbreakable messenger built on a **real one-time pad** — Shannon-perfect secrecy, not "strong until
it isn't." Pads are generated locally from true randomness, shared **air-gapped** (never over a network),
consumed once, and burnt as they're used. No cloud, no servers, no computational assumptions.

> **Status: working prototype.** `index.html` proves the core — a real Poly1305 one-time MAC, the A/B pad split,
> word-salad encoding, the burn-ledger, and a passphrase vault. See **[SPEC.md](SPEC.md)** for the full design:
> air-gapped pad exchange (AirShare's QR / an SD card), the bind-both at-rest key, email-driven transport, and
> the honest security boundaries.

## Try the prototype

Serve `index.html` over **https / localhost** (WebCrypto needs a secure context), then: generate a pad →
compose → *Encrypt & burn* → paste the output into **Receive** → *Decrypt*. Flip a word in the received text to
watch the MAC reject it.

## Licence

Dual-licensed: free software under **GPL-3.0** (see [LICENSE](LICENSE)); a **commercial licence** is available
for closed / proprietary use (see [LICENSE-COMMERCIAL.md](LICENSE-COMMERCIAL.md)). Contact hello@tiny-web.uk.

Made by [Wallisoft](https://github.com/wallisoft) — Eastbourne.
