# Clicky repo audit (2026-04-09)

## Repo context
- Remote: https://github.com/0xNyk/clicky.git
- Branch: main
- Worktree cleanliness at end of audit: clean

## What this project is
Clicky is a macOS menu bar companion app (`LSUIElement=true`) implemented in Swift/SwiftUI + AppKit panels. Voice pipeline: push-to-talk -> streaming transcription -> screenshot capture -> Claude response (SSE) -> ElevenLabs TTS playback -> optional cursor pointing on-screen. A Cloudflare Worker proxy handles upstream API keys and routes.

## Stack and runtime
- App: Swift 5, SwiftUI, AppKit (`NSPanel`, `NSStatusItem`), AVFoundation, ScreenCaptureKit
- Package dependencies (SPM): Sparkle 2.9.0, PostHog iOS 3.47.0, PLCrashReporter 1.12.2
- Proxy: Cloudflare Worker TypeScript (`worker/src/index.ts`)
- Worker toolchain: `wrangler` pinned to `^3.0.0`

## Layout map
- `leanring-buddy/`: main app code
- `worker/src/index.ts`: API proxy (`/chat`, `/tts`, `/transcribe-token`)
- `scripts/release.sh`: build/sign/notarize/Sparkle/GitHub release pipeline
- `appcast.xml`: Sparkle feed
- `AGENTS.md`: architecture and coding conventions

## Runtime architecture (high-level)
1) `leanring_buddyApp.swift` bootstraps app delegate and `CompanionManager`.
2) `MenuBarPanelManager.swift` controls status bar panel lifecycle.
3) `BuddyDictationManager` + `GlobalPushToTalkShortcutMonitor` manage input and hotkey transitions.
4) `CompanionScreenCaptureUtility` captures screen(s).
5) `ClaudeAPI` sends chat+images via Worker `/chat` (SSE stream).
6) `ElevenLabsTTSClient` sends text via Worker `/tts`, plays MPEG locally.
7) `OverlayWindow` renders cursor/response and performs pointing animation.

## Quality gate results
- `xcodebuild ... build`: failed in this environment due Xcode plugin load failure (`IDESimulatorFoundation`, exit 70). Build health could not be validated from CLI here.
- Worker validation:
  - `npm ci`: pass
  - `npm run deploy -- --dry-run`: pass
  - `npm audit --json`: 4 vulns (3 moderate, 1 high), all transitive through pinned Wrangler v3 stack; suggested fix path is Wrangler v4.

## Findings

### P0
1) Public worker endpoints have no request authentication/rate-limit boundary in repo code.
   - `worker/src/index.ts` proxies expensive upstream APIs for any POST caller.
   - Risk: abuse/cost burn if worker URL is discovered/shared.

### P1
2) App and docs claim all external API access is via Worker, but codebase still contains direct Anthropic-call path + API-key build setting.
   - `ElementLocationDetector.swift` directly calls `https://api.anthropic.com/v1/messages` with `x-api-key`.
   - `project.pbxproj` contains `INFOPLIST_KEY_AnthropicAPIKey = "sk-ant...6AAA"` in Debug + Release build settings.
   - Even if placeholder, this is architectural drift and a secret-handling smell.

3) Documentation drift / product identity drift.
   - README/AGENTS call product Clicky; release scripts/appcast/repo target still `makesomething` (`julianjear/makesomething-mac-app`).
   - Increases release/operator error risk.

4) Test coverage is minimal and mostly scaffold.
   - Unit tests: 3 focused tests around permission-gating helper behavior.
   - UI tests are template launch/perf scaffolds.
   - No tests for voice pipeline, worker contract, SSE parsing, or coordinate mapping logic.

### P2
5) Large high-coupling files are risk hotspots.
   - `CompanionManager.swift` (~1026 LOC), `OverlayWindow.swift` (~881), `DesignSystem.swift` (~880), `BuddyDictationManager.swift` (~866), `CompanionPanelView.swift` (~761).
   - Increases regression risk and review burden.

6) Forced casts/unwraps exist in critical paths (mostly acceptable but brittle under malformed state).
   - Notably in accessibility/window positioning logic and URL initializers.

## Contributor entry points (where to start)
- Core interaction state machine: `CompanionManager.swift`
- Voice capture/transcription: `BuddyDictationManager.swift`, `BuddyTranscriptionProvider.swift`, `AssemblyAIStreamingTranscriptionProvider.swift`
- UI panel/overlay: `MenuBarPanelManager.swift`, `CompanionPanelView.swift`, `OverlayWindow.swift`
- Proxy contracts: `worker/src/index.ts`

## Recommended next actions
- P0: Add worker auth guard (signed client token or CF Access/JWT), plus rate limiting and origin checks.
- P1: Remove or gate direct `ElementLocationDetector` API path, and eliminate any API-key-bearing build settings from project file.
- P1: Align naming/docs/release targets (Clicky vs makesomething), or document intentional split explicitly.
- P1: Add worker contract tests and a small suite for transcript-finalization and SSE parsing logic.
- P2: Begin modular split of `CompanionManager` and `OverlayWindow` into feature-focused collaborators.
