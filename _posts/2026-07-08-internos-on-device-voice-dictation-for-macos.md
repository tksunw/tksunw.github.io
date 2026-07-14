---
title: "InterNos: On-Device Voice Dictation for macOS"
date: 2026-07-08 13:30:00 -0400
categories: [Automation]
tags: [macos, swift, speech, dictation, privacy, apple]
---

Like most people, I talk faster than I type. When I talk, my ideas tend to flow more naturally than when I type because typing forces my brain to try and organize my thoughts into words that make sense.  Tools like Wispr Flow and Whispersync already figured this out: hold a key, speak, release, and your words appear at the cursor. The catch is that your voice goes to their servers. Every offhand remark, every half-dictated Slack message, every NDA protected dictation, all of it leaves your machine and goes somewhere you have no control over, to someone that you just have to blindly trust is going to uphold their data privacy claims.

macOS 26 shipped the `SpeechAnalyzer` and `SpeechTranscriber` APIs, which expose the same on-device speech engine that powers system dictation. That removed the only technical excuse for cloud round-trips. So I built InterNos: a menu bar app that does the whole hold-speak-release workflow with zero network calls in the transcription path.

- Repo: [github.com/tksunw/InterNos](https://github.com/tksunw/InterNos) (MIT, signed and notarized DMG in [Releases](https://github.com/tksunw/InterNos/releases))

The name is Latin, roughly "between us." Dictation that stays between you and your Mac.  Transcriptions are never sent to a cloud for processing or post-processing.

## What it does

Hold the right Option key (configurable, toggle mode available), speak, release. The transcript is inserted at the cursor in whatever app has focus. Measured release-to-inserted-text latency runs 50 to 70 milliseconds for typical speech patterns.

The pipeline is three stages, all local:

1. **Capture.** An `AVAudioEngine` mic tap, converted to the analyzer's native format (16 kHz mono).
2. **Transcribe.** Apple's `SpeechAnalyzer` plus `SpeechTranscriber`, streaming results as you speak.
3. **Insert.** Clipboard swap with a synthetic ⌘V, then your original clipboard is restored.

The insert stage checks for macOS Secure Input first. If a password field has focus, InterNos refuses to inject anything, plays the error sound, and leaves the transcript on the clipboard so nothing is lost. A dictation tool that types into password fields is just a keylogger with extra steps.

There's also a small post-processing pass for spoken commands.  For example: "hashtag yard" becomes `#yard`, "at sign" becomes `@`, and "emoji thumbs up" becomes 👍. The emoji substitution requires the spoken word "emoji" as a prefix, so telling someone "she sent me a smiley face" stays literal text.  Other keywords that work like this include "start quote" and "end quote", "open parenthesis" and "close parenthesis", and "verbatim" for not post-processing whatever is said next.

There are also two custom lists, stored in the user Library.  The first is **Replacements**, which allow you to set some static replacements for things the transcriber gets wrong or doesn't know about.  For example "cube control" can be fixed as "kubectl" or "t k sun w" is fixed as "tksunw".  The second list is **Snippets** which works like keyboard shortcuts, so I could set the phrase "t k g h" to be replaced with "https://github.com/tksunw".

## Things that bit me

This was my first serious macOS app, and the Apple stack has some sharp edges that produce silence instead of errors. I very literally could not have built this without Claude's assistance, and the release of Fable-5 seemed like a perfect way to put Fable through it's paces.  Building this on my own would have taken me months of evenings and weekends. For example:

**Audio format mismatches fail silently.** The analyzer wants 16 kHz mono Int16. The mic delivers 48 kHz Float32. Feed it the wrong format and you get empty transcription results, not an exception. Fable burned a chunk of a day on this before writing a spike CLI that validated every stage of the pipeline in isolation.  Silent failures are the worst in any scenario.

**`AVAudioEngine` goes stale.** Reuse an engine across utterances after stop/removeTap and it silently yields empty transcripts. A fresh engine per utterance fixed it.

**Hardened runtime plus microphone needs an entitlement.** A non-sandboxed app signed with `--options runtime` must carry `com.apple.security.device.audio-input`. Without it, the mic permission prompt fires, but no grant persists and the app never appears in the Microphone privacy pane. Accessibility and Input Monitoring don't need entitlements, so those worked while mic didn't, which is a confusing diagnostic signature. This shipped broken in v1.0.0 and got fixed in v1.0.1 within hours of my own first install.

**TCC ties grants to bundle ID, path, and code signature.** A debug build sharing a bundle ID with the installed release copy breaks Input Monitoring and Accessibility in ways the privacy panes won't admit: the toggle shows granted, the app can't see it. Debug builds now use a separate bundle ID and a separate output path. Related: `CGRequestListenEventAccess()` can fail to create a TCC record at all, leaving the app absent from the Input Monitoring pane with no way for the user to grant anything. The reliable workaround is attempting a throwaway `CGEvent.tapCreate` when the request reports not-granted, which forces the record to exist.  This one was particularly annoying, even with Fable's help.

**Styled DMG installers are cursed.** The classic Finder AppleScript layout trick fails silently in headless builds, and baking a .DS_Store doesn't survive because the background image reference is a Finder alias tied to the original volume's identity. `dmgbuild` writes the .DS_Store programmatically per build and just works. But a nice installer is pretty cool to have when you finally get it right.

## Privacy posture

No accounts, no telemetry, no transcript storage. Settings are the only persisted state. The speech model is downloaded once by macOS itself from Apple's asset CDN and shared system-wide. The only optional network call in the whole app is "Check for Updates," which hits the GitHub releases API on demand, with an off-by-default toggle for checking at launch. All of this is verifiable with Little Snitch or any packet monitor, which is the point: a privacy claim you can't verify is marketing. 

## Requirements and install

macOS 26 (Tahoe) or later on Apple Silicon, English (US) in v1. Grab the DMG from [Releases](https://github.com/tksunw/InterNos/releases), drag to Applications, and the onboarding walks you through the three permissions it needs (Microphone, Input Monitoring, Accessibility) plus the one-time model download.

Or build from source:

```sh
git clone https://github.com/tksunw/InterNos.git
cd InterNos/App
./scripts/make-app.sh release   # → App/build/Internos.app
```

My wife and son use it daily, which is a higher bar than any test suite. Issues and PRs welcome, especially additions to the emoji table.

## Update: v1.1 and v1.2

A week of dogfooding produced two more releases.

**v1.1 is all bug fixes**, mostly races in the dictation lifecycle. Back-to-back dictations now always insert in recording order even when a later utterance finishes transcribing first. Clipboard restore only puts your original clipboard back if the pasteboard still holds the injected transcript, so it can't clobber something you copied right after dictating. The transcript pastes only into the app that was frontmost when you released the hotkey; switch apps during finalization and InterNos refuses, keeping the text on the clipboard instead. Pause now cancels all in-flight work. The injected transcript is also tagged with the standard transient/concealed pasteboard markers so well-behaved clipboard managers skip it.

**v1.2 adds the customization layer:**

- **Personal dictionary.** Spoken-phrase replacements ("cube control" becomes `kubectl`), matched whole-word and case-insensitive, with JSON import/export. This is the answer to `AnalysisContext.contextualStrings` being inert (see above): if the engine won't take a custom vocabulary, post-process one.
- **Voice-triggered snippets.** "Snippet address" inserts saved text verbatim. Static text only, on purpose.
- **Structured voice commands.** "New line", "new paragraph", "bullet point", "numbered item" with automatic numbering, quotes and parentheses, plus a "literal" escape when you want the words themselves.
- **Smart cleanup (opt-in, off by default).** Removes filler, repetitions, and false starts, and applies self-corrections ("meet at five, no, six" becomes "meet at six"). Runs on-device via Apple Intelligence Foundation Models, bounded by a two-second deadline with output validation, and always falls back to the deterministic transcript. Same privacy posture: nothing leaves the machine.
- **Last-dictation recovery.** Menu bar items to copy or re-paste the last transcript, held in memory only.

**Final Thoughts** 

Both the README and privacy policy now say plainly that the transcript passes briefly through the general pasteboard during insertion, where Universal Clipboard or a clipboard manager could observe it. InterNos's own network behavior is unchanged (still zero calls in the dictation path), but a privacy tool should describe its actual mechanics, not just the flattering parts.

Full details in the [CHANGELOG](https://github.com/tksunw/InterNos/blob/main/CHANGELOG.md).
