## Project
Wispr Flow clone. Tauri desktop app, Rust core, Groq Whisper for
transcription, Gemini Flash for cleanup, API keys in the system keychain.
SPEC.md defines behavior and module ownership. GitHub issues track the work.

## Advisor
- Consult the advisor before locking any plan that touches more than one module.
- If the same test or compiler error fails twice, consult the advisor before retrying.
- Consult the advisor on the full diff before marking an issue done or merging.

## GitHub workflow
- GitHub issues are the source of truth for tasks.
- Each teammate works in its own git worktree under ../wispr-worktrees/<name>
  on its own branch. Never switch branches in the main checkout.
- Only edit the directories your role owns in SPEC.md. If you need a change
  elsewhere, message that owner.
- One PR per issue, linked with "Closes #N". Merge only after CI passes and
  the reviewer approves. Never push directly to main.

## Milestones
- At the end of each milestone, stop and give me a short checklist to test by
  hand (hotkey, mic, text appearing in other apps).

## Discovery
- Use the scout subagent for repo searches and docs lookups.
