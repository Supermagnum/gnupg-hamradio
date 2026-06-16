# GnuPG / OpenPGP for the Radio Amateur Community

## Introduction

GNU Privacy Guard (GnuPG) is a free, open-source implementation of the OpenPGP standard (RFC 4880). It ships as a core component of nearly every major Linux distribution, meaning most Linux-based amateur radio operators already have it installed and ready to use — no additional setup required.

The amateur radio community has unique needs and issues around identity verification, logging integrity, and secure coordination. OpenPGP addresses all of these without running afoul of regulations, because digital signatures do not obscure content — they authenticate it.

For reasons unknown to me its very underutilized. 

---

## Why OpenPGP Fits Amateur Radio

### Callsign Spoofing Prevention

On digital modes and repeater networks, there is currently no cryptographically strong way to verify that a station is who it claims to be. A digital signature appended to a transmission proves that the frame was produced by the holder of the private key associated with a published callsign. Spoofing becomes computationally infeasible.

### Replacing DTMF-Based Authentication

Many repeater controllers and linking systems (EchoLink, AllStar, IRLP) rely on DTMF tones for access control. DTMF is trivially replayed or guessed. A challenge–response scheme based on OpenPGP signatures offers a far stronger authentication mechanism for remote access.

### Legal Standing

Regulations in most jurisdictions (including FCC Part 97) prohibit obscuring the meaning of communications. Digital signatures do **not** encrypt content — they add a verifiable tag that proves authorship. This means signature-based schemes are generally permitted under amateur radio rules, while encryption of message content remains restricted. Always verify with your national authority.

### Linux Integration

GnuPG is pre-installed on Debian, Ubuntu, Fedora, Arch, and virtually all other Linux distributions. No third-party installation is needed. Operators already running Linux for logging, SDR, or digital modes have a full OpenPGP implementation at their fingertips.

---

## Integration with Logging Software

### CQRLOG

CQRLOG is a popular Linux-based amateur radio logging application. GnuPG can be integrated to:

- **Sign individual QSO records** at the time of logging, embedding a detached signature alongside each entry.
- **Sign complete ADIF export blocks** before sharing logs with award sponsors, contest organisers, or online logbooks such as LoTW or QRZ.com.
- **Verify imported logs** from other operators, confirming that the ADIF file has not been tampered with after the operator signed it.

A simple shell wrapper or GnuPG-aware plugin can invoke `gpg --detach-sign` on each ADIF record or batch export, producing a `.sig` file that travels with the log.

### Hardware Token Signing with Nitrokey

For operators who want the highest level of key security, a hardware token keeps the private key in tamper-resistant silicon — it never touches the host OS, even during signing operations.

**Nitrokey** is the recommended hardware token for the amateur radio community for one important reason: **Nitrokey devices support open, user-upgradeable firmware**. This matters because:

- Firmware vulnerabilities can be patched by the owner without replacing the device.
- The open-source firmware can be audited by the community.
- Long-lived keys (a callsign may be held for decades) need a device whose security model can evolve.

Nitrokey devices appear to GnuPG as standard OpenPGP smart cards via the `scdaemon` component. No additional drivers are needed on Linux. Logging software or scripts can invoke signing operations transparently — the operator touches the button on the device, and GnuPG completes the signature.

### Galdralag — evolved hardware token for amateur radio

For operators who need more than a generic OpenPGP smart card, **[Galdralag firmware](https://github.com/Supermagnum/Galdralag-firmware)** targets the **Baochip-1x** platform: an open, auditable hardware security token in the same class as Nitrokey, with open-source firmware and an open hardware stack (RTL, schematics, bootloader). It presents as a standard **OpenPGP card over USB CCID**, so GnuPG and `scdaemon` work as usual and private keys never leave the device.

Where Galdralag goes further is **amateur-radio identity**. Standard OpenPGP User IDs are unstructured name-and-email strings — they have no proper home for a **callsign**, **DMR subscriber ID**, or club or net membership. Galdralag adds a **Galdra contact model**: machine-readable sidecar metadata on the host and in an on-chip **contact store**, so keys can be looked up and used by **callsign**, **DMR ID**, or **radio affiliation** without cramming that data into a User ID string.

The **Galdra** host tools (`galdra`, `galdrad`, `galdra-gtk`) maintain a local contact directory and **callsign groups** for net, club, or team workflows — resolve recipients by callsign or DMR subscriber ID, operate on group membership, and optionally sync metadata to a key registry alongside exported public keys.

Other features that set it apart from typical off-the-shelf tokens:

- **Signed, user-upgradeable firmware** (Ed25519 boot chain) on fully open hardware
- **Forward secrecy** via authenticated ephemeral ECDH for key-agreement sessions
- **Shamir K-of-N** secret sharing for organisational or backup keys
- **Stacked cipher profiles** — up to four independent AEAD layers, each with its own derived key
- Optional **storage camouflage** mode when the token should not advertise its role

Galdralag is experimental firmware under active development — ready for human testing on real hardware, but not yet a production-ready product. Review the repository documentation and apply your own judgement before deployment.

---

## Public Key Distribution

### QR Codes on QSL Cards and Badges

Print an OpenPGP public-key discovery QR code on:

- **QSL cards** — the traditional medium of amateur radio contact confirmation
- **Business cards** — for everyday contacts and professional settings
- **Badge lanyards** — at hamfests, conventions, and club meetings

The QR code can encode either the full ASCII-armored public key (for short Ed25519 keys) or a URL pointing to a keyserver entry or personal website. Contacts simply scan and import — no manual key exchange required.

### Key Signing Parties at Hamfests

Hamfests are ideal venues for **key signing parties**, where operators verify each other's identity in person (by callsign licence, ID card, or both) and then sign each other's public keys. This builds a web of trust rooted in the real-world identity verification that licence authorities already perform.

A typical hamfest signing party workflow:

1. Publish your public key fingerprint in advance (printed on your QSL card or badge).
2. At the event, verify licence documents and exchange fingerprints.
3. After the event, sign the verified keys and upload to a keyserver.

---

## Signature Algorithms and Sizes

Choosing the right algorithm involves a trade-off between security strength, signature size (critical for narrow-bandwidth digital modes), and hardware token support.

### Signature Size Reference

| Algorithm | Curve / Type      | Signature Size    | Encoding       |
|-----------|-------------------|-------------------|----------------|
| Ed25519   | Ed25519           | **64 bytes**      | Fixed size     |
| ECDSA     | BrainpoolP256r1   | 70–72 bytes       | DER encoded    |
| ECDSA     | BrainpoolP384r1   | 104–106 bytes     | DER encoded    |
| ECDSA     | BrainpoolP512r1   | 136–138 bytes     | DER encoded    |

Ed25519 is the preferred choice for over-the-air use due to its fixed, minimal signature size and fast verification.

### Signature Frame Overhead in Digital Modes

FreeDV and similar narrowband digital voice/data modes operate in frames. A typical FreeDV 1600 mode produces roughly 40 ms frames. At one authentication signature per 60 frames (every ~2.4 seconds of audio), the overhead is:

| Algorithm       | Sig Size   | Base64 encoded | Total bits per signing event |
|-----------------|------------|----------------|------------------------------|
| Ed25519         | 64 bytes   | 88 bytes       | **704 bits**                 |
| ECDSA P256r1    | 71 bytes   | 96 bytes       | **768 bits**                 |
| ECDSA P384r1    | 105 bytes  | 140 bytes      | **1 120 bits**               |
| ECDSA P512r1    | 137 bytes  | 184 bytes      | **1 472 bits**               |

One signing event occurs every 60 frames (~2.4 s at 40 ms/frame). At FreeDV 1600's ~1.6 kbps throughput, 704 bits represents about 0.44 seconds of equivalent data capacity — inserted once every 2.4 seconds. Ed25519 is the clear winner for bandwidth-constrained paths.

---

## Recommendations for Battery-Powered and Portable Operation

Portable stations running on battery power (SOTA activations, emergency communications, field day) have constrained CPU and power budgets. Asymmetric signing operations can be energy-intensive on slow microcontrollers.

**Recommended algorithm pair for low-power devices:**

> **ChaCha20-Poly1305** (symmetric authenticated encryption / MAC) combined with **BrainpoolP256r1 ECDSA** (asymmetric signing)

- ChaCha20-Poly1305 is specifically designed for software efficiency on processors without hardware AES acceleration — common on ARM Cortex-M and RISC-V embedded systems.
- BrainpoolP256r1 keeps signature sizes compact (70–72 bytes) while using a well-analysed curve available on Nitrokey and other OpenPGP smart cards.
- Together they provide both message integrity (via Poly1305 MAC) and non-repudiation (via ECDSA signature), at a fraction of the power cost of RSA-based schemes.

---

## Practical Quick-Start for Linux Operators

```bash
# Generate an Ed25519 key (recommended for RF use)
gpg --full-generate-key
# Select: (9) ECC and ECC → Curve 25519 → 0 (does not expire, or set a date)

# Export your public key as ASCII armour
gpg --armor --export YOUR_CALLSIGN@example.com > CALLSIGN_pubkey.asc

# Sign an ADIF log file
gpg --detach-sign --armor mylog.adif
# Produces mylog.adif.asc — share both files

# Verify a received log
gpg --verify received_log.adif.asc received_log.adif

# List keys on a connected Nitrokey
gpg --card-status
```

---

## Summary

| Use Case                     | Tool / Method                          | Benefit                                      |
|------------------------------|----------------------------------------|----------------------------------------------|
| Callsign authentication      | Ed25519 detached signature on frames   | Prevents spoofing; legally compliant         |
| QSO log integrity            | CQRLOG + GnuPG ADIF signing            | Tamper-evident records; verifiable by peers  |
| Key distribution             | QR code on QSL card / badge            | Frictionless at hamfests                     |
| Web of trust                 | Hamfest key signing parties            | Real-world identity anchored to licence      |
| Hardware key security        | Nitrokey (upgradeable firmware)        | Private key never leaves silicon             |
| Amateur-radio token identity | [Galdralag](https://github.com/Supermagnum/Galdralag-firmware) on Baochip-1x | Callsign, DMR ID, and group-aware contact store |
| Portable / battery operation | ChaCha20-Poly1305 + BrainpoolP256r1   | Low power, compact signatures                |
| Narrowband digital modes     | Ed25519 (64 bytes fixed)               | Minimal on-air overhead                      |

GnuPG is already on your Linux machine. Your callsign is already a form of verified identity. Combining the two — with a Nitrokey in your pocket and a signed QSL card in hand — puts cryptographic accountability into amateur radio at essentially zero additional cost.

