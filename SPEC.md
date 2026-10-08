# SPEC

Wispr Flow clone. Hotkey, speak, cleaned text lands at the cursor in any app.

## Decisions (from interview)

| Topic | Decision |
|---|---|
| Platform | macOS only (Apple Silicon + Intel). Windows/Linux out of v1. |
| Hotkey | Default `Fn`. Rebindable. Hold = push-to-talk. Double-tap = hands-free lock; tap again to stop. |
| Insertion | Clipboard + simulated Cmd+V. Save and restore prior clipboard. |
| Latency | Key release to text visible: p50 < 1.5s for a 10s utterance. No streaming. |
| Cleanup | Setting: `light` (fillers, punctuation, caps) or `heavy` (rewrite for clarity). Default `light`. Raw transcript used if Gemini fails or exceeds 3s. |
| Cost | Target under $1/month at ~30 min/day. Estimate: Groq whisper-large-v3-turbo $0.04/audio hr (10s minimum per request) = ~$0.60; Gemini Flash ~$0.20-0.40 (prices rise 2027-01-01). Verify on live pricing pages. Local usage counter in settings. No hard cutoff. |
| Language | English only. |
| Keys | Bring your own Groq + Gemini keys, stored in macOS Keychain. |
| Distribution | Local unsigned builds first. Signed/notarized DMG for friends is a post-v1 milestone. |

## Out of scope for v1

Personal dictionary, voice commands ("new line"), app-aware tone, history/replay, auto-update, streaming transcription, multilingual, Windows/Linux.

## Behavior

1. App runs as a menu bar tray item. No dock icon. Settings window opens from tray.
2. User holds hotkey: record from default mic (16 kHz mono). Tray icon shows recording state.
3. On release (or tap-out of hands-free): stop, encode, send to Groq Whisper.
4. Transcript goes to Gemini Flash with the cleanup prompt for the chosen style.
5. Cleaned text is pasted at the cursor, clipboard restored ~150 ms later.
6. Recordings under 300 ms are discarded. Recordings over 5 min auto-stop.
7. Errors (no key, network, permission) show a tray notification. Audio is never written to disk. Nothing is logged except timings and error codes.

### Failure rules
- Groq fails: notify, insert nothing.
- Gemini fails or times out (3s): insert raw transcript, notify once.
- Paste fails (no Accessibility permission): leave text on clipboard, notify.

### macOS gotchas
- **Globe/Fn key:** System Settings > Keyboard > "Press 🌐 key to" defaults to emoji/dictation. First run tells the user to set it to "Do Nothing". Many external keyboards send no Fn, so `Right Option` is offered as an alternate default.
- **Info.plist:** `NSMicrophoneUsageDescription` is required or mic access fails in the bundled app.
- **Dev signing:** ad-hoc rebuilds change the code signature, so macOS re-prompts for Accessibility, Input Monitoring and Keychain every build. Use a stable self-signed dev certificate (ticket 1.4).

### macOS permissions
Microphone, Accessibility (paste), Input Monitoring (Fn capture via CGEventTap). The app checks all three on launch and links to System Settings for any missing one. Note: Tauri's global-shortcut plugin cannot bind bare `Fn`; the shell uses a CGEventTap.

## Architecture

Tauri 2 app. Rust core in `src-tauri`, thin web UI for settings only.

```
hotkey (shell) --events--> orchestrator (shell)
                              |-- audio::Recorder.start/stop --> WAV bytes
                              |-- transcribe::Client.transcribe(wav) --> raw text
                              |-- cleanup::Cleaner.clean(raw, style) --> text
                              '-- insert::Inserter.insert(text)
settings::Store (config + keychain) read by all modules
```

The orchestrator is a single state machine: `Idle -> Recording -> Transcribing -> Cleaning -> Inserting -> Idle`. Errors return to `Idle`.

## Module interfaces

Shared types and traits live in `crates/types` (owned by lead; changes by request). Async traits use `#[async_trait]` and require `Send + Sync`, so the orchestrator can run them on Tauri's tokio runtime.

```rust
pub struct Wav(pub Vec<u8>);               // 16 kHz mono PCM16 with header
pub enum Style { Light, Heavy }
pub enum AppError { NoKey(Service), Network, Timeout, Permission(Perm), Audio, Api(u16) }

// audio + transcription
pub trait Recorder { fn start(&mut self) -> Result<(), AppError>; fn stop(&mut self) -> Result<Wav, AppError>; }
pub trait Transcriber { async fn transcribe(&self, wav: Wav) -> Result<String, AppError>; } // Groq whisper-large-v3-turbo

// cleanup
pub trait Cleaner { async fn clean(&self, raw: &str, style: Style) -> Result<String, AppError>; } // Gemini Flash

// shell
pub trait Inserter { fn insert(&self, text: &str) -> Result<(), AppError>; }
pub enum HotkeyEvent { Press, Release, DoubleTap }

// settings + keychain
pub trait SecretStore { fn get(&self, s: Service) -> Result<Option<String>, AppError>; fn set(&self, s: Service, v: &str) -> Result<(), AppError>; fn delete(&self, s: Service) -> Result<(), AppError>; }
pub enum Service { Groq, Gemini }
pub enum Perm { Microphone, Accessibility, InputMonitoring }
pub struct Hotkey { pub keycode: u16, pub modifiers: u32 }          // Fn or Right Option by default
pub struct Usage { pub audio_secs: f64, pub gemini_in_tokens: u64, pub gemini_out_tokens: u64, pub month: String }
pub struct Settings { pub style: Style, pub hotkey: Hotkey, pub mic: Option<String>, pub usage: Usage }
```

`cpal::Stream` is `!Send`: the Recorder impl owns a dedicated audio thread and is driven over a channel. Hotkey rebinding: the settings UI calls a shell command `capture_next_key()` that returns the next pressed key as a `Hotkey`; settings persists it, shell reads it.

Every external dependency (mic, HTTP, keychain, paste) sits behind a trait so orchestrator and modules test with fakes.

## Ownership

Cargo workspace, one crate per owner, so each owner controls their own dependencies and `cargo test -p <crate>` runs in isolation.

| Owner | Directories | Responsibility |
|---|---|---|
| **lead** | `.github/`, `Cargo.toml` (workspace root), `crates/types/`, `crates/app/` (orchestrator, wiring), `src-tauri/` (Tauri config, Info.plist, entitlements), `SPEC.md`, `PLAN.md` | CI, app wiring, state machine, shared types |
| **audio-transcribe** | `crates/audio/` | Mic capture, resample, WAV encode, Groq client |
| **cleanup** | `crates/cleanup/`, `crates/cleanup/prompts/` | Gemini client, prompts, fallback and timeout |
| **shell** | `crates/shell/` (hotkey, tray, insert, permissions) | CGEventTap, tray, clipboard paste, permission checks |
| **settings** | `crates/settings/`, `ui/` | Config file, keychain, usage counter, settings window |

Cross-owner changes: message the owner. Never edit another owner's directory. Traits live in `crates/types`; each owner implements theirs in their crate. UI stack: vanilla TypeScript + Vite in `ui/`.

## Testing approach

- Unit tests with fakes for each trait. HTTP clients tested against a local mock server.
- Orchestrator tested end to end with fake recorder/transcriber/cleaner/inserter.
- Real-device behavior (hotkey, mic, paste in other apps) is manual: checklist at the end of each milestone.
- CI (macOS runner): `cargo test --workspace`, `cargo clippy -D warnings`, `cargo fmt --check`, `tauri build` on macOS.
