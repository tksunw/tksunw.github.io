---
title: "Understanding TOTP by Building One in PowerShell"
date: 2026-02-08 21:00:00 -0500
categories: [Automation]
tags: [powershell, security, totp, mfa, 2fa, rfc]
---

You've probably used a TOTP code thousands of times — open your authenticator app, read the six digits, type them in before the timer runs out. But have you ever stopped to think about what's actually happening? I did, and the best way I know to understand something is to build it.

The result is `Get-TOTPDigits.ps1`, a PowerShell script that generates RFC 6238-compliant Time-based One-Time Passwords. It's not meant to replace your authenticator app. It's a learning tool that made the whole mechanism click for me.

- [`tksunw/tools/TOTP/Get-TOTPDigits.ps1`](https://github.com/tksunw/tools/tree/main/TOTP)

## What Is TOTP?

TOTP is defined in [RFC 6238](https://tools.ietf.org/html/rfc6238) and builds on the earlier HOTP (HMAC-Based One-Time Password) standard from [RFC 4226](https://tools.ietf.org/html/rfc4226). The core idea is straightforward:

1. You and the server share a secret key (the Base32 string you scan as a QR code during MFA setup)
2. Both sides agree on a time step (almost always 30 seconds)
3. Both sides independently compute the same HMAC hash using the shared key and the current time interval
4. The hash is truncated down to a short numeric code

Because both sides have the same key and the same clock, they arrive at the same code at the same time — no network communication needed after the initial key exchange. That's the elegance of it. The code proves you have the key without transmitting the key.

## The Algorithm Step by Step

The RFC defines TOTP as:

```
TOTP(K) = HOTP(K, T)
        = Truncate(HMAC-SHA1(K, T)) mod 10^d
```

Where `K` is the shared key, `T` is the current time interval (Unix seconds divided by the step, usually 30), and `d` is the number of digits (usually 6).

Here's what that looks like in practice:

**Decode the key.** Shared secrets are Base32-encoded (A–Z, 2–7). The script decodes this into a byte array, accumulating 5 bits per character using a `BigInteger` shift-and-OR loop.

**Compute the time interval.** Divide the current Unix timestamp by the interval (default 30 seconds) and encode the result as an 8-byte big-endian value.

**HMAC it.** Feed the key bytes and the time interval bytes into HMAC-SHA1 (or SHA-256/SHA-512). This produces a 20-byte (SHA-1) hash.

**Dynamic truncation.** Take the low 4 bits of the last byte of the hash — that's your offset. Extract 4 bytes starting at that offset, mask off the sign bit, and take the result modulo 10^6. Pad with leading zeros if needed.

That's it. The whole algorithm fits in about 30 lines of PowerShell.

## What I Learned

Building this made a few things concrete that were previously abstract:

**The shared secret is the entire security model.** The time component isn't secret — anyone knows what time it is. The algorithm isn't secret — it's published in an RFC. The only thing protecting your account is that Base32 key. If someone has your key, they can generate valid codes forever, from anywhere, without you knowing. That's why backing up authenticator app data and protecting those QR codes matters so much.

**The 30-second window is a tradeoff.** A shorter interval means a stolen code is valid for less time. A longer interval is more forgiving of clock drift between client and server. Most implementations accept codes from the previous and next intervals too, effectively making the window 90 seconds. The script's `-Interval` parameter lets you experiment with this.

**HMAC choice barely matters in practice.** RFC 6238 supports SHA-1, SHA-256, and SHA-512, and the script supports all three via the `-Algorithm` parameter. But virtually every real-world TOTP deployment uses SHA-1. Despite SHA-1's weaknesses for collision resistance, HMAC-SHA-1 is still considered secure for this use case because HMAC's security depends on the key, not on collision resistance.

**Six digits is less entropy than you'd think.** A 6-digit code is only about 20 bits of entropy — roughly 1 in a million. That's why rate limiting and lockout policies on the server side are critical. The code itself is easy to brute-force without them. The `-Digits` parameter (5–9) lets you see how the code length affects the output space.

## Usage

```powershell
PS> .\Get-TOTPDigits.ps1 -SecretKey 'JBSWY3DPEHPK3PXP'
482193
```

The script validates the Base32 key strictly — whitespace, hyphens, and `=` padding are stripped, but truly invalid characters (digits 0, 1, 8, 9, or other non-Base32 characters) produce a clear error instead of silently generating the wrong code.

## Parameters

| Parameter | Description |
|-----------|-------------|
| `-SecretKey` | Base32-encoded shared secret (required) |
| `-Interval` | Time step in seconds (30–120, default 30) |
| `-Digits` | Length of the generated code (5–9, default 6) |
| `-Algorithm` | HMAC algorithm: SHA1, SHA256, or SHA512 (default SHA1) |

## Security Considerations

The script handles key material carefully: the `SecretKey` parameter is removed from scope immediately after use, HMAC objects are disposed via `try/finally`, and the decoded key byte array is zeroed after the hash is computed. These are best-effort measures — PowerShell's managed runtime doesn't guarantee that copies won't linger in memory — but they reduce the window of exposure.

This is a learning tool and a convenience script, not a replacement for a proper authenticator app. Authenticator apps have secure storage for keys, protection against shoulder-surfing, and don't expose secrets on the command line. Use this to understand the algorithm, or in automation scenarios where you understand the tradeoffs.

## Source

- [`tksunw/tools/TOTP`](https://github.com/tksunw/tools/tree/main/TOTP)
- [RFC 6238 — TOTP: Time-Based One-Time Password Algorithm](https://tools.ietf.org/html/rfc6238)
- [RFC 4226 — HOTP: An HMAC-Based One-Time Password Algorithm](https://tools.ietf.org/html/rfc4226)
