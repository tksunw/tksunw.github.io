---
title: "Building a Safer Diceware Password Generator in PowerShell"
date: 2026-02-08 20:45:00 -0500
categories: [Automation]
tags: [powershell, security, passwords, passphrases, diceware]
---

I recently cleaned up and hardened a small PowerShell utility that generates [Diceware](https://en.wikipedia.org/wiki/Diceware)-style passphrases using the EFF large wordlist. The idea is simple: string together several randomly chosen words to create a passphrase that's both strong and memorable.

The script lives in my tools repo:

- [`tksunw/tools/DiceWords-Password-Generator/Get-DiceWords.ps1`](https://github.com/tksunw/tools/tree/main/DiceWords-Password-Generator)

## Why Diceware?

A 6-word Diceware passphrase has about 77.5 bits of entropy — roughly 221 quintillion possible combinations. That's strong enough for most real-world use, and you can actually remember it because it's made of real words.

```powershell
PS> .\Get-DiceWords.ps1 -NumberofWords 6

Your words are:
Matador Wayside Reboot Unsteady Imitate Snag

Your passphrase is:
MatadorWaysideRebootUnsteadyImitateSnag

# of possible passwords with 6 rolls:
Entropy: 77.55 bits
~ 221.07 quintillion
```

Compare that to a random 12-character password that you'll never remember and will paste from a clipboard anyway.

## Why I Reworked It

The original script worked, but as I reviewed it, a few things bugged me:

- **Biased randomness** — The original used `Get-Random`, which isn't cryptographically secure. The updated version uses `System.Security.Cryptography.RandomNumberGenerator.GetInt32()`, which produces unbiased, cryptographically secure random numbers.

- **No wordlist validation** — The script downloaded a wordlist from the EFF and trusted it blindly. Now it validates the structure strictly: exactly 7,776 entries, valid five-digit dice codes (digits 1–6 only), no duplicates, no empty words.

- **No tamper detection** — A trust-on-first-use (TOFU) model now saves a SHA-256 hash of the wordlist on first download. On subsequent runs, the hash is verified. If the wordlist has changed upstream, the script refuses to run unless you explicitly opt in with `-AllowWordlistChange`.

- **Inaccurate entropy reporting** — The original entropy calculation didn't account for the script's duplicate-prevention behavior (each word is unique within a passphrase). The math now correctly reflects permutations without replacement.

## Meeting Complexity Requirements

Some sites and systems demand uppercase, lowercase, numbers, and symbols — even when your passphrase is already strong enough without them. The `-AddSpecialsAndNumbers` switch handles this by replacing the spaces between words with random special characters and appending two random digits, all using the same cryptographic RNG:

```powershell
PS> .\Get-DiceWords.ps1 -NumberofWords 6 -AddSpecialsAndNumbers

Your words are:
Matador Wayside Reboot Unsteady Imitate Snag

Your passphrase is:
MatadorWaysideRebootUnsteadyImitateSnag

Your complex password is:
Matador^Wayside$Reboot!Unsteady&Imitate#Snag47
```

That satisfies complexity policies while keeping the passphrase memorable.

## Structured Output

The script returns a `PSCustomObject`, so you can use it interactively and see the formatted display, or capture the result in a script:

```powershell
$result = .\Get-DiceWords.ps1 -NumberofWords 4 -AddSpecialsAndNumbers
$result.ComplexPassword   # just the complex password string
```

The object includes `Words`, `PassPhrase`, `ComplexPassword`, `EntropyBits`, and `Combinations`.

## Parameters

| Parameter | Description |
|-----------|-------------|
| `-NumberofWords` | Number of words in the passphrase (1–20, default 2) |
| `-AddSpecialsAndNumbers` | Replace spaces with random special characters and append two random digits |
| `-WordListFile` | Path to a custom EFF wordlist file (auto-downloaded if omitted) |
| `-AllowWordlistChange` | Accept a changed wordlist hash (required if managed wordlist content differs from the trusted hash) |

## How It Works

1. Downloads the EFF large wordlist if not already cached (platform-specific default paths for Windows, macOS, and Linux)
2. Validates the wordlist structure and verifies its SHA-256 hash
3. Generates unique five-digit dice rolls using the cryptographic RNG
4. Maps each roll to a word, title-cases them, and concatenates into a passphrase
5. Optionally generates a complex password variant with special characters and digits
6. Reports the entropy and total possible combinations
7. Returns a `PSCustomObject` with all results

The script works offline after the initial download — the wordlist is cached locally and re-verified on each run.

## Source

- [`tksunw/tools/DiceWords-Password-Generator`](https://github.com/tksunw/tools/tree/main/DiceWords-Password-Generator)
