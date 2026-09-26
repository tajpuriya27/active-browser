# ActiveBrowser — Architecture & Implementation Plan

A macOS menu-bar agent that registers itself as the system default browser and forwards every clicked link to the browser the user most recently had in focus. If no browser has been focused yet, it falls back to a browser the user chose.

## 1. System Architecture

The app runs as an agent process (`LSUIElement = true`): no Dock icon, no main window, launched at login.

```
┌─────────────────────────────────────────────────────────────┐
│                       macOS System                          │
│                                                             │
│  [Any App / Terminal] ──(Clicks Link)──> Launch Services    │
│  [User switches App]  ───────────────> NSWorkspace Events   │
│  [Login]              ───────────────> SMAppService         │
└──────────────┬───────────────────────────────┬──────────────┘
               │                               │
               ▼ (open urls)                   ▼ (didActivateApplication)
┌─────────────────────────────────────────────────────────────┐
│                       ActiveBrowser                         │
│                                                             │
│  ┌─────────────────────────┐   ┌──────────────────────────┐ │
│  │    BrowserRegistry      │   │      FocusObserver       │ │
│  │ (all installed http     │   │ (touch stack when an     │ │
│  │  handlers, minus self)  │   │  included browser gets   │ │
│  └────────────┬────────────┘   │  focus)                  │ │
│               │                └─────────────┬────────────┘ │
│               ▼                              ▼              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Settings (UserDefaults)                               │ │
│  │   • includedBrowsers: Set<bundleId>  (user picks)      │ │
│  │   • defaultBrowser:   bundleId       (user picks)      │ │
│  └────────────────────────────┬───────────────────────────┘ │
│                               ▼                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  BrowserStack (LRU, @MainActor)                        │ │
│  │   head = most recently focused *included* browser      │ │
│  └────────────────────────────┬───────────────────────────┘ │
│                               ▼                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  URLDispatcher                                         │ │
│  │   first running browser in stack                       │ │
│  │   → else head of stack                                 │ │
│  │   → else settings.defaultBrowser                       │ │
│  │   → NSWorkspace.open(urls, withApplicationAt:)         │ │
│  └────────────────────────────┬───────────────────────────┘ │
└───────────────────────────────┼─────────────────────────────┘
                                ▼
              Target Browser (Brave / Arc / Safari / Chrome / …)
```

### Core Components

| Component | Responsibility |
|---|---|
| **BrowserRegistry** | Discovers every installed app that can open `https://` via `NSWorkspace.shared.urlsForApplications(toOpen:)`. Excludes `Bundle.main.bundleIdentifier`. Exposes `(bundleId, displayName, appURL)`. |
| **Settings** | `UserDefaults`-backed. `includedBrowsers` (which registry entries participate in routing) and `defaultBrowser` (fallback). First launch: include everything, default = first entry. |
| **BrowserStack** | `@MainActor` ordered list. `touch(id)` promotes to head; `prune(keeping:)` drops excluded/uninstalled ids. `resolveTarget(fallback:)` returns first *running* browser in the stack, else head, else fallback. |
| **FocusObserver** | `NSWorkspace.didActivateApplicationNotification` → if bundle id ∈ `includedBrowsers` → `stack.touch(id)`. |
| **URLDispatcher** | `application(_:open:)` → `resolveTarget` → `NSWorkspace.shared.open(urls, withApplicationAt:configuration:)`. |
| **MenuBarManager** | `NSStatusItem` showing current target; submenus to include/exclude browsers and pick the fallback; "Set as Default Browser", "Launch at Login", Quit. |

### Runtime rules

- **Every handler runs on the main thread.** Both inputs (`didActivateApplication` notification and `application(_:open:)`) are delivered on main; all state is `@MainActor`. No locks, no GCD queues.
- **Never route to self.** Registry and settings both exclude the app's own bundle id.
- **Never drop a URL.** `resolveTarget` always ends at `settings.defaultBrowser`; if even that is missing, open with the first registry entry.
- **No polling.** State changes only from system notifications and menu actions.
- **Login item.** `SMAppService.mainApp.register()` on first launch so the app is alive to observe focus before the first link is clicked — but only when `Bundle.main.bundleURL` is under `/Applications/`. A copy running from `build/` must never register, or `make clean` leaves a login item pointing at a deleted app.

## 2. Technical Stack

- Swift 6, strict concurrency, AppKit + ServiceManagement only. Zero third-party dependencies.
- macOS 13.0+ (needed for `SMAppService`; `urlsForApplications(toOpen:)` and `setDefaultApplication` are macOS 12+).
- Built with SwiftPM; the `.app` bundle is assembled by `make`.
- Targets: < 5 MB binary, ~10–15 MB idle RAM.

## 3. Implementation Plan

One PR per task; a task never spans phases. Suggested split (the planner may split further, never merge):

| Task | Phase | Scope | First runnable check |
|---|---|---|---|
| 01 | 1 | `BrowserStack`, `BrowserRegistry`, `Settings`, `Package.swift` | `swift build` |
| 02 | 2 | `AppDelegate` observer + `application(_:open:)` | `swift build` |
| 03 | 3 | `Support/Info.plist`, `make build` / `bundle` / `run` / `clean` | bundle assembles, menu bar item appears |
| 04 | 3 | `make install` incl. `lsregister -u` / `-f` and `pkill` | appears in default-browser list |
| 05 | 4 | Set as Default Browser + routing | links route (needs one `user` click) |
| 06 | 4 | Launch at Login (`SMAppService`, `/Applications` gate) | Login Items shows the app |
| 07 | 5 | Status item: *Routing to* / *Recent*, `menuWillOpen` rescan | menu reflects focus changes |
| 08 | 5 | *Browsers* include/exclude + last-item guard, *Fallback* radio | exclusion changes routing |
| 09 | 6 | `make release`, `install.sh` | one-liner installs from a local zip |
| 10 | 6 | `.github/workflows/release.yml` | tag builds and publishes assets |
| 18 | 6 | .github/workflows/pages.yml, _config.yml | pages workflow green; site shows README, no install.sh |
| 19 | 6 | media/, README.md, .github/workflows/pages.yml | site autoplays showcase video; github.com shows poster |


### Phase 1 — Core (`Core/`)
- `BrowserStack` (`@MainActor`, array-backed LRU).
- `BrowserRegistry` (Launch Services query, self-exclusion, display names).
- `Settings` (UserDefaults keys `includedBrowsers`, `defaultBrowser`).

**Verify (Phase 1)** — no bundle exists yet, so this phase is verified by build and inspection only:
1. `swift build -c release` — expected: succeeds with zero warnings.
2. Read `Sources/ActiveBrowser/Core/BrowserStack.swift` and confirm `resolveTarget` returns, in order: first running item → head → fallback. Behaviour is exercised for real in Phase 4.

### Phase 2 — Events & Routing (`App/`)
- `FocusObserver`: subscribe to `NSWorkspace.shared.notificationCenter`, filter on `includedBrowsers`, touch stack.
- `application(_:open:)`: resolve target, open URLs. Fallback chain as above.
- On launch: `registry.refresh()`, seed settings if empty, `stack.prune(keeping: includedBrowsers)`.

**Verify (Phase 2)** — still no bundle; build-only:
1. `swift build -c release` — expected: succeeds with zero warnings.
2. Confirm `AppDelegate` registers exactly one `didActivateApplication` observer and that `application(_:open:)` never returns without calling `NSWorkspace.shared.open`. End-to-end routing is tested in Phase 4.

### Phase 3 — App Bundle & Makefile (before anything Launch Services related)
- `Support/Info.plist`: `CFBundleIdentifier`, `LSUIElement = YES`, `CFBundleURLTypes` for `http`/`https`.
- `Makefile`:
  - `make build` → `swift build -c release`
  - `make bundle` → `build/ActiveBrowser.app/Contents/{MacOS/ActiveBrowser, Info.plist, Resources/{AppIcon.icns, MenuBarIconTemplate{,@2x,@3x}.png}}` + ad-hoc `codesign -s -`
  - `make install` → copy to `/Applications`, `lsregister -f` to register URL schemes
  - `make run`, `make clean`
- Verify: `Bundle.main.bundleIdentifier` is non-nil when launched from the bundle (self-filter depends on it).
- Why SPM alone is not enough: `swift build` emits a bare Mach-O; macOS only reads `Contents/Info.plist` from a `.app`, so `LSUIElement`, `CFBundleURLTypes`, the bundle id, and the default-browser list all depend on the `make bundle` step.
- Duplicate-registration gotcha: `make run` (opens `build/…app`) and `make install` (opens `/Applications/…app`) both register the same bundle id with Launch Services, and `setDefaultApplication` may bind to either copy. Test default-browser behaviour only from the installed copy; `make install` should `lsregister -u build/ActiveBrowser.app` first, and `make clean` should do the same before deleting it.

**Verify (Phase 3)** — first time the app runs as an app:
1. `make bundle` — expected: `build/ActiveBrowser.app` exists; `codesign -dv build/ActiveBrowser.app` prints `Signature=adhoc`; `plutil -lint build/ActiveBrowser.app/Contents/Info.plist` prints `OK`.
2. `make install` — expected: `/Applications/ActiveBrowser.app` exists, a menu bar item appears, **no** Dock icon appears.
3. `lsregister -dump | grep -A3 'com.local.activebrowser'` — expected: entries listing `http` and `https` under the bundle's claimed schemes.
4. System Settings → Desktop & Dock → *Default web browser* dropdown — expected: **ActiveBrowser** is listed (do not select it yet).
5. Quit from the menu bar; `make clean`; `lsregister -dump | grep -c 'build/ActiveBrowser.app'` — expected: `0` (build copy unregistered).

### Phase 4 — Launch Services Integration
- "Set as Default Browser" → `NSWorkspace.shared.setDefaultApplication(at: Bundle.main.bundleURL, toOpenURLsWithScheme: "http")` (macOS shows its own confirmation). The row reads *✓ Default Browser* (checked, disabled) when `urlForApplication(toOpen:)` resolves both `http` and `https` to this bundle id, evaluated in `menuWillOpen`.
- "Launch at Login" → `SMAppService.mainApp.register()` / `.unregister()`; reflect `.status` in the menu.
**Verify (Phase 4)** — this phase changes your system default browser; the reset step at the end restores it. Use the installed copy only.
1. Menu bar → *Set as Default Browser* — expected: macOS asks to confirm; accept. System Settings → Desktop & Dock now shows ActiveBrowser as default.
2. Click on Brave, then back to Terminal, run `open https://example.com` — expected: opens in Brave.
3. Click on Arc, then back to Terminal, `open https://example.com` — expected: opens in Arc.
4. With Arc still the most recent, quit Arc, then `open https://example.com` — expected: opens in the next most recently focused *running* browser (Brave).
5. Quit every browser, `open https://example.com` — expected: the fallback browser launches and opens the page.
6. Click a link inside a non-browser app (Slack, Mail, Notes) — expected: same routing as above.
7. Menu bar → *Launch at Login* on — expected: System Settings → General → Login Items lists ActiveBrowser. Toggle off — expected: it disappears.
8. Quit ActiveBrowser, then `open https://example.com` — expected: macOS relaunches ActiveBrowser and the link still opens in the fallback browser (nothing is dropped).

Reset: System Settings → Desktop & Dock → *Default web browser* → pick your real browser. Turn *Launch at Login* off before `make clean`.

### Phase 5 — Menu Bar UI (`UI/MenuBarManager.swift`)
```
[Icon: current target name]
  Routing to: Brave                (disabled)
  Recent: Brave › Chrome › Safari  (disabled)
  ──────────
  Browsers            ▸  ☑ Brave  ☑ Chrome  ☑ Safari  ☐ Zoom …   (toggle inclusion)
  Fallback Browser    ▸  ● Brave  ○ Chrome  ○ Safari           (radio, included only)
  ──────────
  Set as Default Browser   (or "✓ Default Browser", disabled, when already default)
  ☑ Launch at Login
  ──────────
  Quit
```
- Toggling a browser off removes it from `includedBrowsers` and prunes the stack. If it was the fallback, fallback moves to the first remaining included browser.
- The last included browser cannot be unticked: its menu item is disabled while it is the only one. `includedBrowsers` is never empty after first launch.
- Registry re-scans when the menu opens (`NSMenuDelegate.menuWillOpen`) so newly installed browsers appear.

**Verify (Phase 5)** — ActiveBrowser set as default (Phase 4 step 1):
1. Focus Brave, open the menu — expected: *Routing to: Brave* and *Recent* lists Brave first. Focus Arc, reopen — expected: *Routing to: Arc*.
2. *Browsers* submenu: untick Arc. Focus Arc, then `open https://example.com` from Terminal — expected: opens in Brave (Arc excluded), and *Recent* no longer shows Arc.
3. Re-tick Arc — expected: it participates in routing again on next focus.
4. *Fallback Browser*: pick Safari; quit all browsers; `open https://example.com` — expected: Safari launches.
5. Untick the browser currently set as fallback — expected: fallback radio moves to the first remaining ticked browser.
6. Untick every browser except one — expected: routing always goes to that one. (Unticking the last one must be refused or must keep it as fallback; verify the menu does not allow an empty set.)
7. Quit ActiveBrowser, relaunch it — expected: the tick states and fallback survive (UserDefaults).
8. *Quit* — expected: menu bar item disappears, process gone (`pgrep -x ActiveBrowser` prints nothing).

Reset: same as Phase 4.

### Phase 6 — Distribution (final deliverable: `curl | sh` install)
Goal: a user with no toolchain runs one command and has ActiveBrowser in `/Applications`, launched and registered with Launch Services.

- `install.sh` at the repo root:
  1. `set -euo pipefail`; refuse to run on anything but macOS 13+ / arm64 or x86_64 as built.
  2. Resolve the latest release tag via `https://api.github.com/repos/hexember/active-browser/releases/latest`.
  3. `curl -fsSL` the `ActiveBrowser.app.zip` asset to a temp dir; verify the `SHA256SUMS` asset published alongside it.
  4. `pkill -x ActiveBrowser || true`; `rm -rf /Applications/ActiveBrowser.app`; `ditto -x -k` the zip into `/Applications`.
  5. `lsregister -f /Applications/ActiveBrowser.app`, then `open -a ActiveBrowser` so Launch Services indexes the URL schemes and the menu bar item appears.
  6. Print next step: open the menu bar item → "Set as Default Browser".
  Usage: `curl -fsSL https://raw.githubusercontent.com/hexember/active-browser/main/install.sh | sh`
- `make release`: `make bundle`, then `ditto -c -k --keepParent build/ActiveBrowser.app build/ActiveBrowser.app.zip` and `shasum -a 256` → `build/SHA256SUMS`.
- GitHub Actions `release.yml` on tag `v*`: `macos-latest` runner, `make release`, attach zip + `SHA256SUMS` to the Release with `gh release create`.
- Project site: `https://hexember.github.io/active-browser/` is `README.md` rendered by Jekyll (`jekyll-theme-cayman`) via `.github/workflows/pages.yml` on push to `main`. Only README, CONTRIBUTING, SECURITY, CODE_OF_CONDUCT and CHANGELOG are published, plus `media/showcase.mp4` and its poster (embedded in README; github.com, which strips `<video>`, shows a poster fallback that Liquid removes from the site) (`_config.yml` `exclude:`, plus a post-build allowlist check in the workflow). `install.sh` is never served from Pages; the one install URL stays `raw.githubusercontent.com`.
- Signing: ad-hoc for v1. `curl` does not set the quarantine attribute, so an ad-hoc-signed bundle opens without Gatekeeper prompts via `install.sh`. Browser downloads and Homebrew *do* quarantine; if those paths are added later, add `make sign` (Developer ID) and `make notarize` (`notarytool`) targets first.
- Homebrew Cask: out of scope for v1.

**Verify (Phase 6)** — proves a stranger's machine can install it. Steps 4–6 need the repo to be public.
1. `make release` — expected: `build/ActiveBrowser.app.zip` and `build/SHA256SUMS` exist; `shasum -a 256 -c build/SHA256SUMS` (run from `build/`) prints `OK`.
2. `git tag v0.1.0 && git push origin v0.1.0` — expected: the `release` workflow runs green on GitHub Actions and the Release page for `v0.1.0` shows both assets.
3. Simulate a fresh machine: quit ActiveBrowser, `rm -rf /Applications/ActiveBrowser.app`, `lsregister -kill -r -domain local -domain user`.
4. Run the one-liner: `curl -fsSL https://raw.githubusercontent.com/hexember/active-browser/main/install.sh | sh` — expected: script prints each step, finishes with the "Set as Default Browser" hint, menu bar item appears.
5. `xattr -l /Applications/ActiveBrowser.app` — expected: **no** `com.apple.quarantine` line (this is why curl works without notarization).
6. Run the one-liner again with the app running — expected: it replaces the app in place without error (idempotent upgrade path).
7. Repeat Phase 4 steps 1–3 on the curl-installed copy — expected: routing works identically.

Reset: same as Phase 4.

No XCTest target. Every phase above ends with a **Verify** block; the planner turns it into per-task Test Steps tagged `ai` (run by the main session before the PR opens) or `user` (run by the human at the PR, and collected in `tasks/TEST-PLAN.md`). See `CLAUDE.md`.

## 4. Directory Structure

```
active-browser/
├── CLAUDE.md
├── Project.md
├── Package.swift
├── Makefile
├── install.sh                       # Phase 6
├── _config.yml                      # Phase 6: GitHub Pages (Jekyll) config
├── media/                           # Phase 6: README showcase video + poster (published on Pages)
├── .github/workflows/release.yml    # Phase 6
├── .github/workflows/pages.yml      # Phase 6: README → GitHub Pages
├── .claude/{agents,rules}/          # subagents + always-on rules
├── docs/skills.md
├── tasks/                           # one file per task, from TEMPLATE.md (see CLAUDE.md)
│   └── TEMPLATE.md
├── assets/                          # icon artwork
│   ├── AppIcon.icns                 # copied into Contents/Resources by `make bundle`
│   ├── menubar/MenuBarIconTemplate{,@2x,@3x}.png
│   ├── icon.svg, menubar-icon.svg   # sources of truth for both marks
│   ├── icon-1024.png, README.md     # flat preview; regeneration recipe
│   └── tools/render.swift           # AppKit-only SVG→PNG rasteriser, never compiled
├── Support/
│   └── Info.plist
└── Sources/
    └── ActiveBrowser/
        ├── App/
        │   ├── AppDelegate.swift        # @main, no main.swift
        │   └── FocusObserver.swift
        ├── Core/
        │   ├── BrowserRegistry.swift
        │   ├── BrowserStack.swift
        │   ├── Settings.swift
        │   └── URLDispatcher.swift
        └── UI/
            └── MenuBarManager.swift
```

## 5. Code Skeleton

### Support/Info.plist
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleIdentifier</key>        <string>com.local.activebrowser</string>
    <key>CFBundleName</key>              <string>ActiveBrowser</string>
    <key>CFBundleIconFile</key>          <string>AppIcon</string>
    <key>CFBundleExecutable</key>        <string>ActiveBrowser</string>
    <key>CFBundlePackageType</key>       <string>APPL</string>
    <key>CFBundleShortVersionString</key><string>1.0.0</string>
    <key>CFBundleVersion</key>           <string>1</string>
    <key>LSMinimumSystemVersion</key>    <string>13.0</string>
    <key>LSUIElement</key>               <true/>
    <key>CFBundleURLTypes</key>
    <array>
        <dict>
            <key>CFBundleURLName</key>    <string>Web Link Handler</string>
            <key>CFBundleURLSchemes</key> <array><string>http</string><string>https</string></array>
        </dict>
    </array>
</dict>
</plist>
```

### Package.swift
```swift
// swift-tools-version:6.0
import PackageDescription

let package = Package(
    name: "ActiveBrowser",
    platforms: [.macOS(.v13)],
    targets: [
        .executableTarget(name: "ActiveBrowser", path: "Sources/ActiveBrowser")
    ]
)
```

### Core/BrowserStack.swift
```swift
import AppKit

@MainActor
final class BrowserStack {
    private(set) var items: [String] = []

    func touch(_ bundleId: String) {
        items.removeAll { $0 == bundleId }
        items.insert(bundleId, at: 0)
    }

    func prune(keeping valid: Set<String>) {
        items.removeAll { !valid.contains($0) }
    }

    /// First running browser in LRU order → head of stack → user fallback.
    func resolveTarget(fallback: String?) -> String? {
        let running = items.first {
            !NSRunningApplication.runningApplications(withBundleIdentifier: $0).isEmpty
        }
        return running ?? items.first ?? fallback
    }
}
```

### Core/Settings.swift
```swift
import Foundation

@MainActor
final class Settings {
    private let defaults = UserDefaults.standard

    var includedBrowsers: Set<String> {
        get { Set(defaults.stringArray(forKey: "includedBrowsers") ?? []) }
        set { defaults.set(Array(newValue).sorted(), forKey: "includedBrowsers") }
    }

    var defaultBrowser: String? {
        get { defaults.string(forKey: "defaultBrowser") }
        set { defaults.set(newValue, forKey: "defaultBrowser") }
    }
}
```

### Core/BrowserRegistry.swift
```swift
import AppKit

@MainActor
final class BrowserRegistry {
    struct Entry { let bundleId: String; let name: String; let url: URL }

    private(set) var installed: [Entry] = []

    func refresh() {
        let probe = URL(string: "https://example.com")!
        let me = Bundle.main.bundleIdentifier
        installed = NSWorkspace.shared.urlsForApplications(toOpen: probe).compactMap { url in
            guard let bundle = Bundle(url: url),
                  let id = bundle.bundleIdentifier,
                  id != me else { return nil }
            let info = bundle.infoDictionary ?? [:]
            let name = (info["CFBundleDisplayName"] as? String)
                ?? (info["CFBundleName"] as? String)
                ?? url.deletingPathExtension().lastPathComponent
            return Entry(bundleId: id, name: name, url: url)
        }
        .sorted { $0.name.localizedCaseInsensitiveCompare($1.name) == .orderedAscending }
    }

    func entry(for bundleId: String) -> Entry? {
        installed.first { $0.bundleId == bundleId }
    }
}
```

### App/AppDelegate.swift
```swift
import AppKit
import ServiceManagement

@main
final class AppDelegate: NSObject, NSApplicationDelegate {
    private let registry = BrowserRegistry()
    private let settings = Settings()
    private let stack = BrowserStack()
    private var menuBar: MenuBarManager?

    func applicationDidFinishLaunching(_ notification: Notification) {
        registry.refresh()

        // First launch: include every detected browser, fallback = first one.
        if settings.includedBrowsers.isEmpty {
            settings.includedBrowsers = Set(registry.installed.map(\.bundleId))
        }
        if settings.defaultBrowser == nil {
            settings.defaultBrowser = registry.installed.first?.bundleId
        }
        stack.prune(keeping: settings.includedBrowsers)

        NSWorkspace.shared.notificationCenter.addObserver(
            self,
            selector: #selector(appActivated(_:)),
            name: NSWorkspace.didActivateApplicationNotification,
            object: nil
        )

        menuBar = MenuBarManager(registry: registry, settings: settings, stack: stack)

        // Auto-register only for the installed copy; dev builds in build/ would leave a dangling login item.
        if Bundle.main.bundleURL.path.hasPrefix("/Applications/"),
           SMAppService.mainApp.status == .notRegistered {
            try? SMAppService.mainApp.register()
        }
    }

    @objc private func appActivated(_ notification: Notification) {
        guard let app = notification.userInfo?[NSWorkspace.applicationUserInfoKey] as? NSRunningApplication,
              let id = app.bundleIdentifier,
              settings.includedBrowsers.contains(id) else { return }
        stack.touch(id)
        menuBar?.refresh()
    }

    // Called by Launch Services when we are the default browser.
    func application(_ application: NSApplication, open urls: [URL]) {
        let target = stack.resolveTarget(fallback: settings.defaultBrowser)
            ?? registry.installed.first?.bundleId
        guard let target,
              let appURL = NSWorkspace.shared.urlForApplication(withBundleIdentifier: target) else { return }
        NSWorkspace.shared.open(urls, withApplicationAt: appURL,
                                configuration: NSWorkspace.OpenConfiguration())
    }
}
```

### UI/MenuBarManager.swift (outline)
```swift
import AppKit
import ServiceManagement

@MainActor
final class MenuBarManager: NSObject, NSMenuDelegate {
    private let item = NSStatusBar.system.statusItem(withLength: NSStatusItem.variableLength)
    private let registry: BrowserRegistry
    private let settings: Settings
    private let stack: BrowserStack

    init(registry: BrowserRegistry, settings: Settings, stack: BrowserStack) { … build menu; menu.delegate = self }

    func menuWillOpen(_ menu: NSMenu) { registry.refresh(); rebuild() }
    func refresh() { item.button?.title = currentTargetName }

    // Actions
    @objc func toggleBrowser(_ sender: NSMenuItem)     // flip includedBrowsers, prune stack, fix fallback
    @objc func chooseFallback(_ sender: NSMenuItem)    // settings.defaultBrowser = sender.representedObject
    @objc func setAsDefault()                          // NSWorkspace.shared.setDefaultApplication(at: Bundle.main.bundleURL, toOpenURLsWithScheme: "http")
    @objc func toggleLaunchAtLogin()                   // SMAppService.mainApp.register()/unregister()
    @objc func quit()                                  // NSApp.terminate(nil)
}
```

### Makefile (targets)
```make
APP     = ActiveBrowser
BUILD   = .build/release/$(APP)
BUNDLE  = build/$(APP).app

build:
	swift build -c release

bundle: build
	rm -rf $(BUNDLE)
	mkdir -p $(BUNDLE)/Contents/MacOS
	mkdir -p $(BUNDLE)/Contents/Resources
	cp $(BUILD) $(BUNDLE)/Contents/MacOS/$(APP)
	cp Support/Info.plist $(BUNDLE)/Contents/Info.plist
	cp assets/AppIcon.icns $(BUNDLE)/Contents/Resources/AppIcon.icns
	cp assets/menubar/MenuBarIconTemplate.png $(BUNDLE)/Contents/Resources/
	cp assets/menubar/MenuBarIconTemplate@2x.png $(BUNDLE)/Contents/Resources/
	cp assets/menubar/MenuBarIconTemplate@3x.png $(BUNDLE)/Contents/Resources/
	codesign --force --sign - $(BUNDLE)

LSREG   = /System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister

install: bundle
	-pkill -x $(APP)
	-$(LSREG) -u $(BUNDLE)
	rm -rf /Applications/$(APP).app
	cp -R $(BUNDLE) /Applications/
	$(LSREG) -f /Applications/$(APP).app
	open /Applications/$(APP).app

run: bundle
	open $(BUNDLE)

release: bundle
	ditto -c -k --keepParent $(BUNDLE) build/$(APP).app.zip
	cd build && shasum -a 256 $(APP).app.zip > SHA256SUMS

clean:
	-$(LSREG) -u $(BUNDLE)
	rm -rf .build build
```
