# PLAN

One ticket = one GitHub issue = one PR ("Closes #N"). Owners per SPEC.md. After each milestone: stop and hand over a manual test checklist.

## M1: CI + bare app (lead)

**1.1 Scaffold bare Tauri 2 app + workspace**
- AC: Cargo workspace with empty crates `types`, `audio`, `cleanup`, `shell`, `settings`, `app`; Tauri 2 shell in `src-tauri/`; `ui/` is vanilla TS + Vite; `NSMicrophoneUsageDescription` set; `cargo tauri build` succeeds on macOS; app launches and shows an empty window.
- Test: CI build job; manual launch.

**1.2 CI workflow** (`.github/workflows/ci.yml`, `macos-latest`)
- AC: runs `cargo fmt --check`, `cargo clippy -D warnings`, `cargo test`, `cargo tauri build` on PR and push to main. Cargo cache enabled. Job name: `ci`.
- Test: green run on a PR; a deliberately failing test turns it red.

**1.3 Branch protection on main**
- AC: PRs required, status check `ci` required, no direct pushes, no force pushes, branch up to date before merge.
- Test: `gh api repos/AllumStrat/wispr-clone/branches/main/protection` shows the rules; direct push is rejected.
- Note: needs repo admin; depends on 1.2 having run once so the check name exists. The repo is private: branch protection on private repos needs GitHub Pro/Team. Verify at 1.3; if unavailable, the user chooses (upgrade or make public).

**1.4 Stable dev signing**
- AC: `docs/DEV_SIGNING.md` gives step-by-step Keychain Access instructions for the user to create a self-signed code-signing certificate (name, Certificate Type: Code Signing, trust settings). `scripts/build-signed.sh` builds and signs using the cert by name (default `AllWispr Dev`, overridable via `SIGNING_IDENTITY`); it fails with a clear message if the cert is missing. Agents never create, modify or read the user's Keychain.
- Test: user creates cert, runs the script, grants Accessibility, rebuilds, confirms no re-prompt. Script tested for the missing-cert error path without touching the Keychain (`security find-identity` read-only is the only check, run by the script on the user's machine).

**Manual checklist:** app builds locally, window opens, CI is green, push to main is blocked.

## M2: Settings + keychain (settings)

**2.1 SecretStore (keychain)**
- AC: get/set/delete for Groq and Gemini keys via macOS Keychain. Missing key returns `None`.
- Test: unit tests against an in-memory fake; one `#[ignore]` real-keychain test run manually.

**2.2 Settings store**
- AC: JSON in app config dir; defaults on first run; corrupt file falls back to defaults; style, hotkey, mic persisted.
- Test: unit tests on tempdir (roundtrip, corrupt, missing).

**2.3 Settings window UI**
- AC: key entry (masked, saved to keychain), style toggle light/heavy, hotkey field, mic dropdown, usage counter display.
- Test: Tauri command tests; manual UI pass.

**Manual checklist:** enter keys, quit, relaunch, keys persist; confirm keys appear in Keychain Access; toggle style persists.

## M3: Audio + transcription (audio-transcribe)

**3.1 Recorder**
- AC: start/stop via cpal; resample to 16 kHz mono; returns valid WAV; under 300 ms returns `Audio` error; 5 min cap.
- Test: unit tests on resample/encode with synthetic samples; manual mic test via a dev command.

**3.2 Groq transcriber**
- AC: POST WAV to Groq Whisper; returns text; maps 401 to `NoKey`, timeout to `Timeout`, other to `Api(code)`; 10s timeout.
- Test: mock HTTP server for 200/401/500/timeout.

**Manual checklist:** dev command records 5s and prints the transcript.

## M4: Cleanup (cleanup)

**4.1 Prompts**
- AC: `light` and `heavy` prompts in `prompts/`; forbid adding content, answering the text, or commentary.
- Test: golden fixtures (5 messy transcripts each) checked manually and by simple assertions (no preamble, no quotes).

**4.2 Gemini cleaner**
- AC: calls Gemini Flash; 3s timeout; on any error returns `Err` so orchestrator falls back to raw; usage tokens reported.
- Test: mock server (200, 429, timeout, malformed body).

**Manual checklist:** run dev command on 3 sample transcripts in each style.

## M5: Shell (shell)

**5.1 Permission checks**
- AC: detect Microphone, Accessibility, Input Monitoring; deep links to System Settings; checked at launch.
- Test: manual (revoke each permission, observe prompt).

**5.2 Hotkey (CGEventTap)**
- AC: emits Press/Release/DoubleTap for `Fn` or Right Option; rebindable via `capture_next_key()`; first-run prompt tells the user to set "Press 🌐 key" to Do Nothing; double-tap window 300 ms; does not swallow other keys.
- Test: state-machine unit tests on synthetic key event streams; manual.

**5.3 Tray**
- AC: menu bar icon with idle/recording/busy states; menu: Settings, Quit; notifications for errors.
- Test: manual.

**5.4 Inserter**
- AC: saves clipboard, sets text, sends Cmd+V, restores clipboard after 150 ms; if paste is not permitted, text stays on clipboard and error returned.
- Test: fake pasteboard unit tests; manual in Notes, Slack, browser field, VS Code.

**Manual checklist:** hold Fn and see tray change; paste a fixed string into 4 apps; clipboard restored.

## M6: Orchestrator + end to end (lead)

**6.1 State machine**
- AC: implements flow and failure rules from SPEC; hold and hands-free modes; re-entry during processing ignored.
- Test: fakes for all four traits; cover happy path, Groq fail, Gemini fail (raw fallback), paste fail, short recording, key missing.

**6.2 Wire real implementations + latency log**
- AC: timings logged per stage; p50 under 1.5s on a 10s utterance on the author's machine.
- Test: manual run of 10 utterances.

**Manual checklist:** hold Fn, speak, text appears in Notes/Slack/browser/terminal; double-tap hands-free works; unplug network gives an error; remove Gemini key gives raw text.

## M7: Polish for daily use (lead + owners)

- 7.1 Launch at login. AC: toggle in settings (settings). Test: toggle, reboot, app starts in tray.
- 7.2 Cost counter accuracy. AC: tokens and audio minutes tallied, monthly estimate shown (settings). Test: unit tests on tally math; compare against provider dashboards after a day of use.
- 7.3 First-run flow. AC: missing keys/permissions guided in order (shell + settings). Test: wipe config and keychain entries, walk through on a clean launch.

**Manual checklist:** one full workday of use; note misses.

## M8: Distribution to friends (post-v1)

- 8.1 Apple Developer signing + notarization in CI. AC: release build passes `spctl --assess` and opens on a clean Mac without a Gatekeeper warning. Test: install on a second Mac.
- 8.2 DMG release workflow on tag. AC: pushing `v*` tag attaches a signed DMG to a GitHub release. Test: tag a prerelease and download it.
- 8.3 Auto-update (deferred unless needed). AC: TBD when scheduled.

## Dependencies

M1 first. M2, M3, M4, M5 are independent after M1 and can run in parallel in separate worktrees. M6 needs M2 to M5. M7 needs M6.
