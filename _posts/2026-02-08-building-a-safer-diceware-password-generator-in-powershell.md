---
title: "Building a Safer Diceware Password Generator in PowerShell"
date: 2026-02-08 20:45:00 -0500
categories: [Automation]
tags: [powershell, security, passwords, passphrases, diceware]
---

I recently cleaned up and hardened a small PowerShell utility in my tools repo: `Get-DiceWords.ps1`.

The script generates Diceware-style passphrases using the EFF large wordlist and is now organized under its own folder:

- `DiceWords-Password-Generator/Get-DiceWords.ps1`

## Why I Reworked It

The original goal was simple: generate memorable passphrases that are still strong enough for real-world use.

As I reviewed the script, a few things stood out:

- random generation should be unbiased and cryptographically secure
- downloaded wordlists should be validated instead of trusted blindly
- output should reflect the actual entropy model being used
- docs should make operational behavior obvious

## Security Improvements

The updated script now:

- uses `System.Security.Cryptography.RandomNumberGenerator.GetInt32()` to avoid modulo bias
- validates wordlist structure strictly (7,776 entries, valid `11111`-`66666` keys, no duplicates, non-empty words)
- uses a local trust-on-first-use hash file (`eff_large_wordlist.sha256`) for managed wordlists
- requires explicit opt-in (`-AllowWordlistChange`) before accepting a changed managed wordlist hash

This is a practical middle ground when an upstream source does not publish an official signed hash.

## Behavior and Accuracy Improvements

A couple of operational details were fixed as well:

- entropy/combinations now match the script's unique-roll behavior
- download progress preference is restored safely
- path handling works correctly for custom wordlist file locations

## Usage

```powershell
pwsh ./DiceWords-Password-Generator/Get-DiceWords.ps1 -NumberofWords 6
```

If the managed wordlist content changes and you trust the new version:

```powershell
pwsh ./DiceWords-Password-Generator/Get-DiceWords.ps1 -AllowWordlistChange
```

## Source

The script and README live in my tools repository:

- [tksunw/tools](https://github.com/tksunw/tools)
