# Investigation: "empty white window" on first launch (v0.3.2)

**Date:** 2026-09-19 · **Severity:** critical — every new install is unusable
**Affected:** `Ambient-Claude_0.3.2_x64-setup.exe` (13 downloads; v0.3.1 had 2, which were the
publish-day verification downloads, so effectively the entire external user base is on 0.3.2)

## What users report

Download and install work. Launching the app shows a single empty **white** window instead of
the setup screen. Nothing in it responds, "Check for Updates" appears to do nothing, and the
only recourse is uninstalling.

## What that window is

It is the **first-run Setup wizard**: a decorated 760×720 window labeled `wizard`, created by
`open_wizard_window()` (`src-tauri/src/lib.rs:227-241` at `0eda9189`) during `.setup()` because
onboarding is incomplete (`lib.rs:574-580`: no EULA record → wizard opens, pill stays hidden,
voice stays parked).

Two properties of the app explain the *symptoms* exactly:

1. **White means the wizard's JavaScript never executed.** `wizard.html` ships no stylesheet
   and no markup — the dark theme (`#16161a`) and all content are injected by
   `src/wizard.ts` (`injectStyles()`, first synchronous statement of `mountWizard`). If the
   module runs at all — even if every backend call then hangs — the window turns dark. A pure
   white window is only possible when the page's module script never ran (or the WebView2
   renderer died). Conversely, a *failed navigation* renders Edge's "can't reach this page"
   error page **with text** (documented first-hand in `docs/HANDOFF-2026-08-31` §truth 1), so
   "empty white" is not a navigation failure either.

2. **"Can't pull any updates" is real but separate, and fully explained by code.** The tray's
   *Check for Updates…* works Rust-side, but its outcome is reported only via
   `app.emit(ANNOUNCE/DEGRADED)` (`src-tauri/src/tray.rs:128-154`) — events only the **pill**
   page subscribes to (`src/main.ts` `connect(...)`). Before onboarding completes the pill
   window is *hidden*, so update feedback is invisible, always. (And 0.3.2 is the newest
   release, so the check correctly finds nothing — also invisibly.)

## Why this was never seen on the dev machine

The wizard-at-boot path only runs when `~/.ambient-win` has no EULA record
(`onboard.rs:438-445`, `store.rs:31`). The build machine has carried that record since
development began, so **packaged** builds there boot straight to the pill. In **dev** builds
(`npm run tauri dev`) the wizard is served by Vite from `http://localhost:5173`, not from
embedded assets. Net effect: *"wizard created at boot + embedded-asset serving"* — the exact
path every customer hits — had never executed on any machine the team controls.

## What the investigation ruled out (with evidence)

The shipped artifact was pulled from the release and dissected; the frontend was rebuilt from
the tagged source; the exact serving pipeline was replicated with the same library versions;
and the full first-run flow was executed end-to-end on Linux. Everything app-side is sound:

| # | Check | Result |
|---|-------|--------|
| 1 | Shipped installer integrity | sha256 `77f245b2…af4148` matches the GitHub asset digest — analyzed bytes = user bytes |
| 2 | NSIS contents | `ambient-claude.exe` (19 MB), bundled `voice/` (99 MB), hooks — complete |
| 3 | Frontend embedded in the shipped exe | Asset keys present: `/wizard.html`, `/index.html`, `/border.html`, `/assets/wizard-CH-vuuaE.js`, `/assets/ipc-BuzCV3aG.js`, `/assets/main-DndegkLg.js`, `/assets/main-DWSYUuDM.css` |
| 4 | Frontend bytes match the tagged source | A clean `npm run build` of `0eda9189` produces **identical Vite content hashes** (`wizard-CH-vuuaE.js` etc.) — the shipped bundle is exactly this source |
| 5 | "Dev-flavored exe" trap (HANDOFF-2026-08-31 truth 1) | **Ruled out.** tauri-codegen 2.6.3 embeds **zero** assets in a dev-flavored build (`context.rs`: `if dev && dev_url … EmbeddedAssets::default()`). The shipped exe *has* the assets ⇒ it was compiled with `custom-protocol`, i.e. a proper `tauri build` |
| 6 | Compile-time HTML rewriting | Replicated with the exact `tauri-utils 2.9.3` pipeline (`parse_doc → inject_nonce_token → serialize_doc`): output intact; **no nonce token is injected** into `wizard.html` (the selector only matches `script[src^='http']`, and there are no inline scripts/styles) |
| 7 | Runtime CSP | Therefore served verbatim: `default-src 'self'; style-src 'self' 'unsafe-inline'` — permits the same-origin module script, the dynamic chunk imports, and the JS-injected `<style>` |
| 8 | Page + CSP in a Chromium engine | Served `dist/` with that exact CSP header → wizard renders the dark EULA gate, zero console errors (`evidence/chromium-prod-csp-wizard-renders.png`) |
| 9 | **Full first-run, real Tauri, production serving** | Built the app on Linux with `--features tauri/custom-protocol` (embedded-asset serving + production CSP), ran it under Xvfb with a fresh `$HOME`: `boot.wizard` breadcrumb logged, wizard window opened **and rendered the EULA gate** (`evidence/linux-first-run-wizard-renders.png`) |
| 10 | App code hijacking the page | No `navigate`/`eval`/`location` writes anywhere; only two window-creation sites (wizard, border); tray *Settings* reuses the same wizard window |
| 11 | Boot-time update check hanging the UI | None exists — the updater only runs from the tray verb, async |
| 12 | Capability/IPC config | `capabilities/default.json` includes the `wizard` window; and IPC failures could not cause *white* anyway (see above) |
| 13 | 0.3.1 → 0.3.2 regression | The diff (`b0df4c1..0eda9189`) touches only `audio.rs`/`session.rs` (push-to-talk) + version bumps; Cargo.lock stacks identical (tauri 2.11.5 / wry 0.55.1 / tao 0.35.3) |
| 14 | Known-fixed wry regression | wry changelog 0.55.1→0.57.0 contains no custom-protocol/blank-window fix that matches |

(Linux-run footnotes, for reproducibility: the shell crate is Windows-only as written — two
local, experiment-only shims were needed and are *not* shipped anywhere: a stub for
`tts::win::list_voices` on non-Windows, and skipping boot-time `set_ignore_cursor_events(true)`
on the **hidden** pill, which panics inside tao's GTK backend (`event_loop.rs:457` unwrap on an
unrealized window) and aborts the process. That panic is GTK-specific — the Windows backend
sets window styles that work on hidden windows — but it is worth knowing the call sits on the
boot path.)

## Where the failure actually lives

With the artifact, the frontend, the serving pipeline, and the whole first-run flow verified
good, the failure is isolated to the **WebView2 layer on the affected machines** — the wizard
webview either never executes the page's module script or its render process dies immediately.
The shipped build has **zero diagnostics on that surface**, which is why the failure presents
as a bare white rectangle:

- `wizard.html` has no static background or fallback content — any webview failure looks like
  a blank white window (`src/wizard.ts` injects everything).
- The WebView2 `ProcessFailed` watchdog is installed **only on the pill** (`lib.rs:515`,
  `crashreport.rs:132-139`); a wizard renderer crash is silent and permanent.
- No `on_page_load` / navigation breadcrumbs are logged, so `audit.log` cannot distinguish
  "page loaded and ran" from "nothing ever arrived".

Most probable concrete causes, in order:

1. **WebView2 render-process failure on those machines** (GPU driver + `transparent`/
   composition interaction, aggressive AV injecting into the renderer, broken WebView2
   install). Fits "white + window alive + no error page"; the missing wizard watchdog makes it
   permanent.
2. **wry 0.55.1 × newest WebView2 Evergreen interaction** when the second webview is created
   during `setup()` on a cold profile (first-ever launch creates `EBWebView` from scratch —
   another thing the dev machine never does). wry 0.56.0's changelog ("new default background
   color API… reduce white flashes", webview2-com 0.39) shows active churn in exactly this
   area.
3. **System proxy / loopback policy** interfering with `http://tauri.localhost` interception
   (historically reported against corporate proxy/PAC setups).

WebView2-runtime-absent is effectively ruled out: with no runtime, window creation fails and
the process exits — there would be no window at all.

## Fastest path to certainty (10 minutes, on the build machine)

The customer condition is trivially reproducible locally — it was only ever masked by the
pre-existing state dir:

```powershell
ren "$env:USERPROFILE\.ambient-win" ".ambient-win.bak"
# install the SHIPPED artifact (not a local build):
#   Ambient-Claude_0.3.2_x64-setup.exe  → launch Ambient Claude
```

- **If the wizard is white on the build machine too** → the bug is in the shipped binary ×
  current WebView2 and can be debugged live (build once with the `devtools` feature to get
  F12 in a release-flavored exe).
- **If the wizard renders** → the trigger is machine-environmental; go straight to the user
  diagnostics below.

Afterwards restore: quit Ambient (tray → Quit), uninstall, `ren` the backup back.

## What to collect from one affected user

Everything below exists today, no new build needed:

1. `%USERPROFILE%\.ambient-win\audit.log` — expect `boot.wizard onboarding incomplete…`; its
   *presence* proves the Rust shell is alive and stamps the failure as webview-side.
2. `%USERPROFILE%\.ambient-win\boot.json` (+ `crash.log` / `crash-last.log` if present) —
   `"stage": "ready"` = clean boot, white window notwithstanding; a died-at stage = shell
   crash instead (would also explain a *tiny unclickable white strip* variant via the safe-mode
   pill path, `lib.rs:521-536`).
3. WebView2 version: `reg query "HKLM\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}" /v pv`
4. Whether `%LOCALAPPDATA%\com.ambient.claude\EBWebView\Crashpad\reports\` is non-empty
   (renderer crash dumps → cause 1 confirmed).
5. GPU + antivirus product, Windows build, and whether a proxy/PAC is configured.

## Fix plan for AmbientWin (0.3.3)

Hardening (turns any recurrence into a diagnosable, non-blank state):

1. **`wizard.html`: static dark background + inline fallback text** ("Setting up… if this
   message stays, Ambient's display engine failed to start — see …") in plain HTML/CSS, removed
   by `wizard.ts` on mount. CSP already allows it (`style-src 'unsafe-inline'`; codegen hashes
   inline scripts automatically if one is ever added). White becomes impossible.
2. **Install the existing watchdog on the wizard**: `crashreport::install_webview_watchdog`
   is already window-generic — call it on the window built in `open_wizard_window()` so a dead
   renderer reloads instead of staying blank.
3. **Breadcrumb the page lifecycle**: `WebviewWindowBuilder::on_page_load` (Started/Finished →
   `audit.log`) for wizard and pill, plus one boot line with `tauri::webview_version()`.
   The next report then pinpoints the layer in one file read.
4. **Make update feedback visible pre-onboarding**: subscribe the wizard page to
   `ANNOUNCE`/`DEGRADED` (or `emit_to` both windows from `tray.rs`), so *Check for Updates*
   stops being a silent no-op for stuck users. (Also: the "Update installed — restarting…"
   announce is emitted and immediately killed by `app.restart()` — reorder or delay.)
5. **Safe mode on a not-yet-onboarded machine should still open the wizard** — today the
   safe-mode branch shows only the (click-through, transparent) pill, which on an
   un-onboarded machine is a dead end.
6. **Stop shipping `claude_stub.exe`** — the test stub is in the NSIS payload (204 KB, top
   level, beside the app exe).
7. Consider bumping wry ≥ 0.56 (default-background-color API on Windows + focus-on-create
   fix) once tauri-runtime-wry compatibility allows.

## How stuck users get the fix

The rescue channel already exists: the tray updater works before onboarding (it is pure
Rust + `installMode: passive`). Once 0.3.3 is published and `latest.json` bumped, affected
users need only:

> right-click the Ambient tray icon → **Check for Updates…** → wait ~1 min; the app installs
> the update and restarts itself.

No reinstall required. Until then there is no in-app workaround — the wizard is the only
onboarding surface, so uninstalling (what users are already doing) is the honest answer.

---

### Evidence files in this repo

- `evidence/chromium-prod-csp-wizard-renders.png` — shipped wizard bundle + production CSP in
  a Chromium engine: renders correctly.
- `evidence/linux-first-run-wizard-renders.png` — the full app, first-run, production-style
  embedded-asset serving on Linux: wizard renders correctly.

### Source references (private repo `AmbientWin` @ `0eda9189`)

`lib.rs:227-241` (wizard window), `lib.rs:574-600` (first-run routing), `lib.rs:502-536`
(pill/watchdog/safe-mode), `crashreport.rs:128-178` (watchdog), `tray.rs:91-154` (menu verbs +
update feedback), `src/wizard.ts` (self-styling page), `src/main.ts:25-50` (pill-side gate),
`onboard.rs:426-445` (EULA record), `docs/RELEASE-LOCAL.md` (release path),
`docs/HANDOFF-2026-08-31-POLISH-AND-031.md` (dev-exe trap, error-page rendering).
