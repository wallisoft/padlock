# Padlock — Design Spec & Build Blueprint
*Wallisoft · drafted 20 Nov 2026 · a true one-time-pad messenger*

---

## 1. What Padlock is

A genuinely unbreakable messenger built on a **real one-time pad** — not "crypto that's strong until it isn't," but Shannon-perfect secrecy. Pads are generated locally from true randomness, shared **air-gapped** (never over a network), consumed **once**, and **burnt** as they're used. No cloud, no servers, no computational assumptions, nothing to subpoena.

Android app, built in VML. One-time cost: you and your correspondent have to be in the same room (or trust one courier) **once** to swap a pad. After that you talk securely until it runs out.

---

## 2. Security model — and the honest boundaries

**What Padlock guarantees:**
- **Perfect secrecy of content** — OTP; mathematically unbreakable, forever, regardless of the attacker's compute.
- **Integrity / unforgeability** — a one-time (Carter–Wegman) MAC, also information-theoretic, so an active attacker can't tamper or forge. *(Plain XOR-OTP has zero integrity — bit-flipping an "unreadable" message is trivial without this. Non-negotiable.)*
- **Forward secrecy** — spent pad is destroyed on use; a later device seizure can't recover past messages.
- **MITM-proof by construction** — the pad crosses an air gap, so there is no "middle" to attack.

**What it does NOT do (say so plainly, in-app):**
- It does **not hide metadata** — that Alice and Bob communicate is still visible to carriers/observers. OTP hides *what*, not *that*.
- It does **not survive a live/coerced unlock** — if they seize the phone unlocked, or make you open it, at-rest crypto is moot (mitigations below shrink the blast radius, they don't eliminate it).
- **Disguise is cosmetic**, not security (see §8).

**Core principle that governs every decision:** the only secret is the pad. Security = secrecy of the pad + never reusing a byte. The whole app is in service of *protect the pad, never reuse, burn as you go.*

**Decisions already locked (don't relitigate):**
- ❌ No public-source pads (books/ISOs/MP4s). Public data = **zero** secret entropy; direct-XOR against it is a broken running-key cipher; fetching it leaves fingerprints. Dead idea, for good reasons.
- ✅ True OTP only. No ChaCha/AES fallback in the core (those were "strong until they're not").
- ✅ Pad shared air-gapped only. No network pad transport, ever (a passive tap defeats it).
- ✅ One-time MAC on every message. Mandatory.

---

## 3. The pad

- **Generation:** hardware CSPRNG (`SecureRandom`/getRandomValues) → true random bytes. Nothing derived, nothing public.
- **Size:** user-chosen. KB = a few messages; MB = thousands; GB on an SD card = effectively a lifetime.
- **A/B split:** divide the pad into an **A-half** and a **B-half**. The initiator sends from A, the responder from B — so two people drawing from one shared pad *can never collide*. Each side keeps a **send-pointer** into its own half and a **receive-pointer** into the other's.
- **MAC reservation:** a slice of pad is set aside for one-time authenticators. Every message consumes **encryption bytes + MAC bytes**. Budget for it when sizing.

So the pad is really four streams: `A-encrypt · A-mac · B-encrypt · B-mac`.

---

## 4. Pad exchange (air-gapped)

The hero mechanism, MITM-impossible because there's no channel to intercept:

- **Animated / fountain QR (primary).** Sender's screen streams a rapid sequence of QR frames; recipient films it. Fountain-code the frames (Luby-transform style) so you don't need a clean capture of every frame — it just keeps cycling until enough arrive. Transfers **megabytes optically**, no pairing, no accounts, no RF.
- **NFC tap** — centimetre range, good for KB–MB.
- **SD card / USB courier** — gigabytes; the full diplomatic-bag model, and how you'd seed a small group.
- **Avoid Bluetooth / Wi-Fi-direct** — they reintroduce an interceptable radio channel and an attackable pairing step.

Both devices finish with a **bit-identical pad** and agreed roles (initiator = A, scanner = B).

---

## 5. Message format

**Header (cleartext — safe, useless without the pad):**
`role (A/B) · encrypt-offset · length · mac-offset`

**Body:** `plaintext XOR pad[encrypt-offset .. +length]`

**Tag:** one-time MAC over `(header ‖ body)` using `pad[mac-offset ..]` — Carter–Wegman / polynomial-evaluation MAC, information-theoretically unforgeable.

**Receive order:** verify MAC **first** (fail → reject as tampered), *then* XOR-decrypt.

**No-reuse ledger (the make-or-break):**
- Each side tracks its consumed offsets and **never reuses a byte** — one reuse = total break.
- The offset travels in the (cleartext) header, so both sides stay in sync even with lost or out-of-order messages.
- Refuse to decrypt anything that would imply pad overlap (tamper/replay guard).

---

## 6. At-rest protection (the vault)

Protects a **locked/powered-off** phone:

- **Bind the key to BOTH a passphrase AND the hardware** — don't let the vault alone hold it. Derive the pad-wrapping key as `key = KDF( passphrase ‖ hardware_secret )`, where `hardware_secret` is random, **non-exportable**, held in Android **StrongBox**/TEE and released only after the **biometric + PIN** gate. Consequences: *no passphrase* → can't derive it **and** can't brute-force offline (the hardware part only joins after the gate, which rate-limits and self-wipes); *no device* → can't derive it (`hardware_secret` is non-exportable); *and if the secure element itself is broken* → the attacker **still needs the passphrase**, the un-stored backstop that refuses to trust the vault. (Biometrics can't *be* key material — they're fuzzy, so they can only *gate*; the passphrase is the real second secret.)
- **Hardware rate-limit / auto-wipe** after N failed auths; **invalidate-on-new-biometric-enrollment** (so they can't add their own fingerprint).
- **Passphrase required from cold boot / after a timeout** — so an opportunistic grab of an already-unlocked phone still can't re-open the vault cold.
- **Crypto-shred burn (important — overwrite doesn't work on flash):** store the pad in **chunks, each wrapped by its own keystore key**. Burning a spent chunk = **destroy its key** → the leftover ciphertext is permanent noise. Flash-safe, instant, and *this is the forward-secrecy mechanism*.
- **Panic-wipe PIN** — a duress code that destroys the master key = instant crypto-death of the whole pad.
- **Optional decoy pad** behind a second passphrase. *Honest caveat:* hidden-volume deniability is imperfect (sometimes forensically detectable; weak once an adversary knows the app offers it). Panic-wipe is cleaner.
- **No OS backup, no cloud sync, no swap.** `allowBackup=false`; exclude from Google backup/keychain sync. Decrypt only the chunk in use; zeroise RAM after.

---

## 7. Messaging UI & transport

The ciphertext is safe on **any** channel (it's OTP — the provider literally cannot read it), so transport is a UX/metadata choice, not a content-security one.

**Chat-first.** The app opens on *conversations* — the daily job. Pad generation, exchange, vault and disguise all live in **Settings → secure contacts**; set up once, then just chat. Each contact = a shared pad + role (A/B) + the two pointers.

**Email-driven, transparently.** Primary transport is the user's own email over IMAP/SMTP, shown as an ordinary chat window — the plumbing is invisible. Padlock IMAP-IDLEs for new mail, decrypts, displays, then **deletes it server-side** (EXPUNGE, inbox + Sent) so the mailbox stays clean. Ciphertext rides in the mail body. Honest caveat: deletion clears the mailbox, not the SMTP relay's logs — email hides content perfectly but still leaks **metadata** (who ↔ who, when).

**Bring-your-own-IMAP — NOT our mail.** Padlock does not route through TinyMail: a secure-comms relay invites abuse reports, blocklisting and legal pressure we keep off our own mail reputation. The user connects their own provider, with presets:
- **Works (mobile IMAP + app-password):** Fastmail · mailbox.org · Posteo · iCloud · Mailfence · StartMail · **Custom IMAP**.
- **Ironic exceptions (no plain mobile IMAP):** ProtonMail = desktop-Bridge-only (desktop Padlock only); Tutanota = no IMAP at all.
- **Gmail / Outlook:** possible but OAuth-only now, with provider verification, and app-passwords being phased out through 2026 — offer as an *advanced* OAuth path, never the default.
- Because content is OTP-safe, provider choice is only about **easy app-password IMAP + metadata stance + won't-lock-you-out**, not secrecy.

**Share-sheet = the fallback.** For ad-hoc channels (WhatsApp, Signal, a paste) Padlock still produces the blob to share out, and registers as a share target to receive. Email is the live-chat spine; share-sheet is the escape hatch. (SMS auto-read needs default-SMS-handler status on Play → self-builds only; WhatsApp has no read API at all.)

**Wake-ups** via the existing VML FCM push + tiny-web relay, so mail lands even when backgrounded. **Desktop SMS:** none, unless routed through the user's own TinySMS gateway.

---

## 8. Disguise (optional, honestly labelled)

"Random bytes raise flags," so offer levels — but be honest in-app that this lowers *casual* flags only:

- **Raw base64** — compact, obviously encrypted.
- **Word-list encoding (default)** — map bytes to common words (BIP39-style). Survives **every** text channel (no re-compression to corrupt it) and reads like a passphrase/word-salad, not a crypto blob. Best bang for buck.
- **Steganography into a lossless image/audio** — looks like a photo, but **dies over WhatsApp/MMS** (they re-compress and shred the payload) → email-attachment only, and detectable by steganalysis.

**In-app banner:** *disguise reduces casual flags; it is not a defence against analysis, and it does not hide that you two communicate.*

---

## 9. Crypto primitives (no heavy deps — in keeping with the house rule)

- **RNG:** platform hardware CSPRNG.
- **Cipher:** XOR with the pad. That's the OTP — nothing to implement.
- **MAC:** one-time Carter–Wegman (polynomial MAC over GF(2^128)) keyed from pad bytes — ~a page of code, no library, information-theoretically pure (matches true-OTP spirit).
- **At-rest wrap:** platform keystore AES-GCM via StrongBox — native, no dep.
- **QR:** Android `BarcodeDetector` for scanning (native); a small QR encoder for the animated display.

No crypto library required anywhere. The strength is the pad, not a dependency.

---

## 10. Build notes (VML / Android / Play)

- VML Android app; the native bits (keystore, camera/QR, share intents, NFC) sit in the Android layer.
- **Play publishing:** goes under the Wallisoft **organisation account** (DUNS in progress — avoids the 12-tester tax). Expect the standard **encryption export-compliance** question on submission (OTP messenger = answerable, just paperwork). **No SMS permissions requested** (share-sheet instead) — keeps it off Play's rejection radar.
- Could also ride the future **VML Player** for distribution, but Padlock's native needs (keystore, camera, NFC) make it a proper per-app build — tier-two, its own APK.

---

## 11. Open questions for tomorrow (decide fresh)

1. **Pad-low UX:** warn + prompt re-exchange at what threshold? Show remaining message budget?
2. **Ratchet vs re-meet:** let the last pad bytes seed/deliver the next pad (ratchet) so you needn't physically re-meet — *or* keep it purist (always re-exchange in person)? Ratchet trades a sliver of the purity for huge convenience.
3. **Groups:** v1 = pairwise pads only (simplest, cleanest). N-way later?
4. **Exact MAC construction** — pin the polynomial/field choice.
5. **Lost/out-of-order message handling** — confirm the offset-in-header sync covers all cases.

---

## 12. The soul of it

**Unbreakable by mathematics, not by obscurity.** No cloud, no backdoor, no "trust our servers," no cipher that ages badly. The entire app is three honest promises kept ruthlessly: *generate real randomness, never reuse a byte, burn as you go.* Everything else is UX around those.

🔐
