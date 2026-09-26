# Consolidated Manual Test Plan

Append-only, in merge order. Each entry is the `user` rows from that task's Test Steps. Run top to bottom after merging the corresponding PRs; reset steps at the end of each block.

<!-- entries added by the main session when each PR opens -->

---

## PR #3 — Task 01: Core model (`feature/core-model`, base `main`)

No `user` steps. Phase 1 is build-and-inspection only; all 8 test steps were `ai` and ran green before the PR opened. Nothing for you to run here.

---

## PR #4 — Task 02: Events & routing (`feature/events-routing`, base `feature/core-model`)

No `user` steps. Phase 2 is still build-and-inspection only — there is no `.app` bundle until task 03, so nothing is clickable. All 12 test steps were `ai` and ran green before the PR opened.

Worth knowing when you review this PR: review round 1 caught a blocking runtime defect — `@main` on an `NSApplicationDelegate` compiles but never installs the delegate, so `application(_:open:)` would never have fired and every URL would have been dropped. Fixed with an explicit `static func main()`; verified by disassembling the binary and by launching a throwaway probe bundle.

---

## PR #5 — Task 03: App bundle & Makefile (`feature/bundle-makefile`, base `feature/events-routing`)

First PR with a runnable `.app`. All 15 `ai` steps ran green before the PR opened, including the first live proof of self-filtering (`includedBrowsers` seeded with Safari/Brave/cmux/Arc and **not** ActiveBrowser). One `user` step:

| # | Action | Expected |
|---|---|---|
| 1 | From the repo root run `make run`, wait ~3 s, then look at (a) the Dock, (b) ⌘Tab, (c) the menu bar. | (a) **No** ActiveBrowser icon in the Dock. (b) **No** ActiveBrowser entry in ⌘Tab. (c) **No** menu bar item — *this absence is correct for this PR;* the status item arrives in task 07. If a Dock icon or a window appears, `LSUIElement` is wrong and this PR should be rejected. |

**Reset after this block**
- `pkill -x ActiveBrowser` — mandatory. The app has no Quit and no Dock/⌘Tab presence until task 07, so this is the only way to stop it. Verify with `pgrep -x ActiveBrowser` printing nothing.
- `make clean` — unregisters the build copy from Launch Services, then removes `.build/` and `build/`.
- `defaults delete com.local.activebrowser` — so later tasks still see a genuine first launch.
- **Default browser: nothing to restore.** This task never changes it. Do not open System Settings for this PR.

---

## PR #6 — Task 04: make install (`feature/make-install`, base `feature/bundle-makefile`)

All 15 `ai` steps pass. A duplicate-Launch-Services-registration defect was found and **fixed** in this PR (two earlier attempts failed; see `tasks/04-make-install.md` Failures for the measurements). ActiveBrowser now appears exactly **once** in the *Default web browser* list.

| # | Action | Expected |
|---|---|---|
| 1 | Open System Settings → Desktop & Dock → scroll to *Default web browser* and open the dropdown. **Look only — do NOT select ActiveBrowser.** Selecting it is task 05. | **ActiveBrowser is listed exactly once**, and your current default is still **Arc**. Phase 3 Verify step 4. Two ActiveBrowser rows would mean this PR's fix regressed. Close the dropdown with Esc. |

**Reset after this block**
- **Do NOT run the teardown between tasks.** Tasks 05-08 all need the installed copy in `/Applications`. Run the teardown once, at the very end of the whole stack (last block in this file).
- **Default browser: nothing to restore.** This task never changes it; steps 1 and 14 assert Arc before and after.
- Note: `make install` now deletes `build/ActiveBrowser.app` as part of the fix. `make bundle` recreates it if you want it back.

---

## PR #<04> — Task 04: make install (`feature/make-install`, base `feature/bundle-makefile`)

14 of 15 `ai` steps pass. **Step 11 fails — a known limitation this PR ships with**: after `make install`, macOS re-registers the rebuilt `build/ActiveBrowser.app` about 1–3 s later, so Launch Services holds two records for `com.local.activebrowser` and the *Default web browser* dropdown lists **ActiveBrowser twice**. The handler role is bound by explicit URL (`Bundle.main.bundleURL` of the running `/Applications` copy), so this cannot silently point your default browser at the disposable `build/` copy — the harm is the duplicated row. Full analysis and the verified remedy are in `tasks/04-make-install.md` under Failures.

| # | Action | Expected |
|---|---|---|
| 1 | Open System Settings → Desktop & Dock → scroll to *Default web browser* and open the dropdown. **Look only — do NOT select ActiveBrowser.** Selecting it is task 05. | **ActiveBrowser is listed exactly once**, and your current default is still **Arc**. Phase 3 Verify step 4. Two ActiveBrowser rows would mean this PR's fix regressed. Close the dropdown with Esc. |

**Reset after this block**
- **Do NOT run the teardown between tasks.** Tasks 05-08 all need the installed copy in `/Applications`. Run the teardown once, at the very end of the whole stack (last block in this file).
- **Default browser: nothing to restore.** This task never changes it; steps 1 and 14 assert Arc before and after.
- Note: `make install` now deletes `build/ActiveBrowser.app` as part of the fix. `make bundle` recreates it if you want it back.

---

## PR #<04> — Task 04: make install (`feature/make-install`, base `feature/bundle-makefile`)

14 of 15 `ai` steps pass. **Step 11 fails — a known limitation this PR ships with**: after `make install`, macOS re-registers the rebuilt `build/ActiveBrowser.app` about 1–3 s later, so Launch Services holds two records for `com.local.activebrowser` and the *Default web browser* dropdown lists **ActiveBrowser twice**. The handler role is bound by explicit URL (`Bundle.main.bundleURL` of the running `/Applications` copy), so this cannot silently point your default browser at the disposable `build/` copy — the harm is the duplicated row. Full analysis and the verified remedy are in `tasks/04-make-install.md` under Failures.

| # | Action | Expected |
|---|---|---|
| 1 | Open System Settings → Desktop & Dock → scroll to *Default web browser* and open the dropdown. **Look only — do NOT select ActiveBrowser.** Selecting it is task 05. | **ActiveBrowser is listed exactly once**, and your current default is still **Arc**. Phase 3 Verify step 4. Two ActiveBrowser rows would mean this PR's fix regressed. Close the dropdown with Esc. |

**Reset after this block**
- **Do NOT run the teardown between tasks.** Tasks 05-08 all need the installed copy in `/Applications`. Run the teardown once, at the very end of the whole stack (last block in this file).
- **Default browser: nothing to restore.** This task never changes it; steps 1 and 14 assert Arc before and after.
- Note: `make install` now deletes `build/ActiveBrowser.app` as part of the fix. `make bundle` recreates it if you want it back.

---

## PR #<05> — Task 05: Set as Default Browser + routing (`feature/set-default-routing`, base `feature/make-install`)

**This is the one block that needs you before anything else can be verified.** Zero production code changed — the routing was shipped by tasks 02/04; this PR proves it.

Steps 1 and 3–8 already ran green (routing, LRU promotion, running-first-beats-LRU, relaunch-from-all-quit, empty-stack fallback, self-filtering, 7.6 MB footprint), exercised via `open -a ActiveBrowser <url>` which reaches the identical `application(_:open:)` entry point. Steps 9–15 are the same proofs through the real Launch Services handler and are recorded **not run — needs user step 2**.

| # | Action | Expected |
|---|---|---|
| 1 | **System Settings → Desktop & Dock → *Default web browser* → select `ActiveBrowser`.** macOS shows a confirmation dialog — accept it. | ActiveBrowser becomes the default. No Dock icon, no window appears. |
| 2 | Focus Brave, then switch to Terminal, then run `open https://example.com/u2` | Opens in **Brave**. |
| 3 | Focus Arc, then Terminal, `open https://example.com/u3` | Opens in **Arc**. |
| 4 | With Arc most recent, quit Arc, then `open https://example.com/u4` | Opens in **Brave** — the next most recently focused *running* browser. |
| 5 | Quit every browser, then `open https://example.com/u5` | A browser launches and opens the page — nothing is dropped. |
| 6 | Click a link inside a non-browser app (Slack, Mail, Notes) | Same routing as above. |
| 7 | Quit ActiveBrowser (`pkill -x ActiveBrowser`), then `open https://example.com/u7` | macOS relaunches ActiveBrowser and the link still opens — nothing dropped on cold start. |

**Reset after this block**
- **Restore your default browser: System Settings → Desktop & Dock → *Default web browser* → `Arc`.** Accept the dialog. This is mandatory.
- Relaunch any browser the steps quit (they restore their tabs) and close the `example.com/*` tabs the run created.
- **Leave `/Applications/ActiveBrowser.app` installed and running** — tasks 06–08 test against it. Do not run `make clean` or delete the defaults domain; the full teardown is the last block in this file.

---

## PR #8 — Task 06: Launch at Login (`feature/launch-at-login`, base `feature/set-default-routing`)

All 7 `ai` steps pass. **This PR fixed a silent no-op**: `Project.md` §5's gate (`status != .notRegistered`) never fires on macOS 26, because `SMAppService` reports `.notFound(3)` when no login item has ever existed — so the app never registered itself. Verified after the fix: status moved `notFound(3)` → `enabled(1)`, and a `build/` copy correctly logs `skipped, bundle is not under /Applications`.

| # | Action | Expected |
|---|---|---|
| 1 | Open **System Settings → General → Login Items & Extensions** and look at the *Open at Login* list. | **ActiveBrowser** is listed, switch on. It may show as from an unidentified developer — expected for an ad-hoc-signed build, not a failure. Leave it as-is for now. |

**Reset after this block — read the ordering, it is not obvious**

This is the first task that creates state surviving a reboot. `make clean` does **not** remove a login item, and neither does quitting the app or deleting it from `/Applications`.

- **While you are still testing tasks 07–08** (which need the installed copy): switch the item **off** rather than removing it with `−`. An off switch leaves the status at `requiresApproval(2)`, which the gate deliberately never re-enables, so it stays off. A `−` removal resets the status to `notRegistered`/`notFound`, and the **next launch re-adds it**.
- **At the final teardown**, do it in this order: `pkill -x ActiveBrowser` → `rm -rf /Applications/ActiveBrowser.app` → *then* remove the entry with `−`. Removing it while the app is still installed is what lets it come back.

---

## PR #9 — Task 07: Menu bar status item (`feature/menubar-status`, base `feature/launch-at-login`)

**This is the first PR with a visible UI.** All 6 `ai` steps pass. It also fixes a defect task 06 handed over: `unregister()` leaves the login-item status at `.notRegistered`/`.notFound`, both of which task 06's launch gate registers on — so switching *Launch at Login* off would have been silently undone at the next launch. A persisted `launchAtLoginOptOut` flag now closes that; verified by the literal log line `ActiveBrowser: login item: skipped, user opted out (launchAtLoginOptOut=true)`.

Steps 12–14 are recorded `not run — needs user step 9 / 10(b) / 11` and will be re-run against your results.

| # | Action | Expected |
|---|---|---|
| 1 | Look at the right-hand side of the menu bar. | A small **globe** icon is present. **No Dock icon**, no window. Hovering it for a second shows a tooltip `Routing to: <a browser name>`. *If you cannot find the icon:* the menu bar may be full or hidden by the notch — check the overflow before calling this a failure. |
| 2 | Click **Brave Browser**, then click the menu bar icon and read the menu. Close it, click **Arc**, then reopen the menu. | Top to bottom: greyed-out `Routing to: …`, greyed-out `Recent: …`, separator, `Set as Default Browser`, `Launch at Login` (tick/dash/blank), separator, `Quit ActiveBrowser`. There are **no** *Browsers* or *Fallback Browser* submenus yet — that is task 08, not a missing feature. First open reads `Routing to: Brave Browser`; after focusing Arc it reads `Routing to: Arc`. Values changing between two opens **without relaunching** is what proves the `menuWillOpen` rebuild runs. |
| 3 | Menu bar icon → **Set as Default Browser**. macOS shows a confirmation dialog — accept it. **This changes your default browser away from Arc; the reset below restores it and is mandatory.** | System Settings → Desktop & Dock → *Default web browser* shows **ActiveBrowser**. Links clicked anywhere now route through it to your most recently focused browser. |
| 4 | (a) Open the menu, note the `Launch at Login` state. (b) If **ticked**, click to switch it OFF, reopen the menu. (c) Quit ActiveBrowser from the menu, relaunch it (`open -a ActiveBrowser`), open the menu again. (d) Click `Launch at Login` to switch it back ON. | (a) Ticked ⇔ System Settings → General → Login Items lists ActiveBrowser, switch on. (b) The tick is **gone** and it disappears from / switches off in Login Items. (c) **After the relaunch it is still off** — this is the whole point of the task. (d) Clicking re-ticks it and it reappears. *Special case:* a **dash** instead of a tick means the login item is in `requiresApproval` (you switched it off in System Settings during task 06). Clicking then **opens System Settings** instead of toggling — designed behaviour; turn it on there and re-run (a)–(d). |
| 5 | Menu bar icon → **Quit ActiveBrowser**. | The icon disappears immediately. No dialog, no crash report. (First task where the app can be quit without `pkill`.) Quitting does **not** remove the login item and does **not** undo step 3 — if ActiveBrowser is your default browser, macOS relaunches it on the next link click, which is correct. |

**Reset after this block**
- **Restore your default browser (mandatory if step 3 was run): System Settings → Desktop & Dock → *Default web browser* → `Arc`.** Accept the dialog.
- A `launchAtLoginOptOut` value you wrote at step 4 is **your** choice and is left alone. The login item itself is reset by PR #8's block (switch **off** in System Settings while still testing; `−` only at the final teardown, after the app is deleted).
- **Relaunch the agent after step 5:** `open -a ActiveBrowser` — task 08 starts from one running copy.
- Close the tabs this run created (`https://example.com/t07*`) and relaunch any browser the steps quit.
- **Leave `/Applications/ActiveBrowser.app` installed.** Do not run `make clean`, `make bundle`, `make run`, or delete the app — task 08 tests against this copy.

---

## PR #10 — Task 08: Browsers + Fallback Browser submenus (`feature/browsers-fallback-menu`, base `feature/menubar-status`)

**Completes the menu.** All 9 `ai` steps pass; this block covers `Project.md` Phase 5 Verify steps 2–7. Two safety properties are worth knowing before you click: the **last ticked browser cannot be unticked** (the guard lives in both the menu item and the writer, so `includedBrowsers` can never become empty), and **unticking the current fallback migrates it automatically** to the first remaining ticked browser in display-name order.

| # | Action | Expected |
|---|---|---|
| 1 | **The two submenus exist and read correctly.** Click the menu bar globe icon and hover **Browsers**, then **Fallback Browser**. | The menu now shows, top to bottom: greyed `Routing to: …`, greyed `Recent: …`, separator, **`Browsers ▸`**, **`Fallback Browser ▸`**, separator, `Set as Default Browser`, `Launch at Login`, separator, `Quit ActiveBrowser`. *Browsers* lists every installed web-link handler in alphabetical order, each with a checkmark; **`cmux` appearing there is correct**, it really is a registered `https` handler — untick it if you do not want links going there. *Fallback Browser* lists only the ticked browsers, with a checkmark on **Arc** and no other. (A checkmark, not a `●` — that is the macOS convention for a radio group; see D9.) |
| 2 | **Untick a browser and watch routing change (Phase 5 Verify 2).** Open **Browsers ▸** and click **Arc** to untick it. Reopen the menu and look at *Browsers*, *Fallback Browser* and *Recent:*. Then click **Arc** itself to focus it, switch to Terminal, and run `open -a ActiveBrowser https://example.com/t08u1`. | Arc's checkmark is gone; Arc has **disappeared from the Fallback Browser submenu** (it lists included browsers only); *Recent:* no longer names Arc. The link opens in the browser you most recently focused **other than Arc** (Brave, if you have been using it) — focusing Arc no longer counts. `open -a ActiveBrowser <url>` is used so this works whether or not ActiveBrowser is your system default |
| 3 | **Re-tick and confirm it rejoins on next focus (Phase 5 Verify 3).** Open **Browsers ▸** and click **Arc** again to re-tick it. Reopen the menu and read *Routing to:*. Then click Arc, reopen the menu, and run `open -a ActiveBrowser https://example.com/t08u2` from Terminal. | Immediately after re-ticking, *Routing to:* is **unchanged** — re-ticking does **not** pretend you just focused Arc (D8). After you actually click Arc, the menu reads `Routing to: Arc` and the link opens in Arc. Arc is back in the *Fallback Browser* list too |
| 4 | **Fallback Browser picks the empty-stack target (Phase 5 Verify 4).** Open **Fallback Browser ▸** and click **Safari**. Reopen the menu to confirm. Then quit **every** browser (Arc, Brave, Safari — ⌘Q, not Force Quit). **Then empty the stack — this line is required, see the note under the table:** run `pkill -x ActiveBrowser; sleep 2; open -a ActiveBrowser; sleep 3` in Terminal. Now run `open -a ActiveBrowser https://example.com/t08u3`. | The checkmark in *Fallback Browser* moves to Safari and is on Safari alone. After the relaunch, **Safari launches** and opens the page — with an empty stack the fallback is the first candidate. (If a browser you quit restores its session and steals focus *before* the `open`, re-run from the `pkill` line.) Leave the fallback on Safari for step 14 |
| 5 | **Unticking the current fallback moves the radio (Phase 5 Verify 5).** With the fallback still on **Safari**, open **Browsers ▸** and untick **Safari**. Reopen the menu and inspect **both** submenus. | Safari is unticked in *Browsers* and gone from *Fallback Browser*. The fallback checkmark has moved on its own to the **first remaining ticked browser, top-to-bottom in the submenu's order** (alphabetical — `Arc` if Arc is ticked). It has **not** vanished, and it has **not** stayed on Safari. Then re-tick Safari and set *Fallback Browser* back to **Arc** before continuing |
| 6 | **The last ticked browser cannot be unticked (Phase 5 Verify 6).** In **Browsers ▸**, untick browsers one at a time until exactly **one** remains ticked. Now try to click that last ticked item. | The last remaining ticked item is **greyed out and does not respond to the click** — it stays ticked and the menu does not let the set become empty. *Fallback Browser* at that moment contains exactly that one browser, checked. Re-tick the others afterwards so all four are ticked again |
| 7 | **Tick state and fallback survive a relaunch (Phase 5 Verify 7).** Untick exactly one browser (say **Brave**) and set *Fallback Browser* to **Safari**. Then menu → **Quit ActiveBrowser**, relaunch it (`open -a ActiveBrowser` or from /Applications), and open both submenus. | After the relaunch, Brave is still unticked and the fallback is still Safari — `UserDefaults` carries both across process death, with no save-on-quit code (D11). **Then restore: re-tick Brave so all four are ticked, and set *Fallback Browser* back to Arc** |
> **Note on step 4 (the fallback step).** Quitting every browser is *not* enough to reach the fallback: `BrowserStack.resolveTarget(fallback:)` returns the stack head even when that browser is not running, and that rung sits ahead of the fallback. The stack is in-memory only, so the `pkill -x ActiveBrowser; sleep 2; open -a ActiveBrowser; sleep 3` line in that step is what actually empties it. This is a defect in `Project.md` Phase 4 Verify step 5's wording, hit for the second time in this run — the code is correct.

**Reset after this block**
- **Restore the settings to the state tasks 09–10 expect:** all four browsers ticked in *Browsers* (Arc, Brave Browser, cmux, Safari) and *Fallback Browser* on **Arc**. You can do it from the menu, or in Terminal with the app quit:
  ```sh
  pkill -x ActiveBrowser; sleep 2
  defaults write com.local.activebrowser includedBrowsers -array com.apple.Safari com.brave.Browser com.cmuxterm.app company.thebrowser.Browser
  defaults write com.local.activebrowser defaultBrowser -string company.thebrowser.Browser
  open -a ActiveBrowser
  ```
- Close the `https://example.com/t08*` tabs and relaunch any browser step 4 had you quit (they restore their sessions).
- **Leave `/Applications/ActiveBrowser.app` installed and running.** Do not run `make clean` and do not delete the defaults domain — the full teardown is the last block in this file.

---

## PR #11 — Task 09: `make release` + `install.sh` (`feature/release-install-sh`, base `feature/browsers-fallback-menu`)

**No Swift changed.** All 11 `ai` steps pass, including the two that had to keep task 04's duplicate-registration fix closed: exactly **one** Launch Services record at T+12 s *and* T+60 s, before and after two consecutive `install.sh` runs. `make release` deletes `build/ActiveBrowser.app` after zipping, because the scanner registers a freshly built bundle ~1–3 s *after* the ~1 s target returns — `lsregister -u` alone cannot win that race.

`install.sh` **stages, validates, then replaces**: it extracts to a temp dir and checks bundle id, architecture and checksum before it kills the running app or deletes `/Applications/ActiveBrowser.app`. All three negative tests (old macOS, missing zip, one-byte-corrupted zip that still extracts) refuse with the installed app's pid and mtime unchanged.

**Your `/Applications/ActiveBrowser.app` was reinstalled by `install.sh` during these steps — that was the test.** It is running, adhoc-signed, quarantine-free, and your settings, login item and default-browser binding were all verified untouched.

| # | Action | Expected |
|---|---|---|
| 1 | Look at the menu bar, click the ActiveBrowser icon, and read the menu. Then open System Settings → General → **Login Items & Extensions**. | The menu bar item is present and the menu shows *Routing to:*, *Recent:*, **Browsers ▸**, **Fallback Browser ▸**, *Set as Default Browser*, a **ticked** *Launch at Login*, and *Quit*. Login Items lists **ActiveBrowser exactly once** — not twice, not zero times. A duplicate or missing entry would mean the delete-and-replace disturbed the login item. |
| 2 | System Settings → Desktop & Dock → **Default web browser** — open the dropdown and count the ActiveBrowser entries. **Do not change the selection** (leave it on Arc unless you are also running PR #7's steps). | **ActiveBrowser appears exactly once.** Two identical rows would mean `make release` left a second registered bundle in the repo's `build/` directory — the defect task 04 fixed and this task had to avoid re-introducing. |

> **Not testable in this PR:** the real one-liner `curl -fsSL https://raw.githubusercontent.com/hexember/active-browser/main/install.sh | sh` needs a **published GitHub Release with both assets** and a **public repository** — neither exists until task 10's workflow runs on a pushed tag, and the repo is currently private. What this PR proves is that every step *after* the download works end to end on a real zip. The one-liner is PR #12's step.

**Reset after this block**
- **Nothing to undo.** These steps only look; they change no state.
- `build/ActiveBrowser.app.zip` and `build/SHA256SUMS` are left on disk deliberately — task 10 uses them. They are **not** committed. `make clean` removes them when you no longer want them, but do not run it while testing PR #12.
- **Leave `/Applications/ActiveBrowser.app` installed and running.**

---

## PR #12 — Task 10: `.github/workflows/release.yml` (`feature/release-workflow`, base `feature/release-install-sh`)

**This workflow has never run.** The repo is private and every earlier PR is unmerged, so the file is not on `main` and no tag exists. All 10 `ai` steps pass, but they are static and local by necessity: YAML structure, the three-way asset-name agreement between `release.yml`, `install.sh` and the `Makefile`, the workflow's own verify block run verbatim against real artefacts, and the duplicate-registration invariant at T+12 s / T+45 s.

**Do these in order — each row depends on the one before it.**

| # | Action | Expected |
|---|---|---|
| 1 | **Merge the whole stack first, bottom-up: #3 → #4 → #5 → #6 → #7 → #8 → #9 → #10 → #11 → #12.** Then repo → **Actions**, and locally `git fetch && git show main:.github/workflows/release.yml \| head -5` | `main` contains the workflow, and Actions lists **release** with "This workflow has no runs yet". Until this, a pushed tag does nothing. |
| 2 | **Tag and publish.** `git checkout main && git pull && git tag v0.1.0 && git push origin v0.1.0`, then watch Actions → **release**. | The run starts within seconds and goes green in ~2–5 min. The `v0.1.0` Release page shows both assets. |
| 3 | **Verify the published bytes.** `mkdir -p /tmp/ab10-rel && cd /tmp/ab10-rel && GH_TOKEN=$(gh auth token --user hexember) gh release download v0.1.0 --repo hexember/active-browser && ls -l && shasum -a 256 -c SHA256SUMS` | Exactly `ActiveBrowser.app.zip` and `SHA256SUMS`; the check prints `ActiveBrowser.app.zip: OK`. **This is the last checkpoint that works while the repo is private.** |
| 4 | **The real one-liner.** *Also requires making the repo public* (Settings → General → Danger Zone → Change visibility). `curl -fsSL https://raw.githubusercontent.com/hexember/active-browser/main/install.sh \| sh` | Prints its `==>` progress lines, ends with the "Set as Default Browser" hint, menu bar item appears. |
| 5 | `xattr -l /Applications/ActiveBrowser.app` | **No `com.apple.quarantine`.** (`com.apple.provenance` is a different, expected attribute.) This is what lets an ad-hoc-signed bundle install without Gatekeeper prompts. |
| 6 | With the app running, re-run the same one-liner; then `pgrep -x ActiveBrowser` and `codesign --verify --strict /Applications/ActiveBrowser.app; echo verify=$?` | Exits `0`, replaces the running app, exactly one process, `verify=0`. |
| 7 | Click Brave → Terminal → `open https://example.com`; click Arc → Terminal → `open https://example.com`. *Requires ActiveBrowser set as default (PR #7's step).* | First link opens in **Brave**, second in **Arc**. |

**Not run, with reasons — not substituted or faked**
- **Phase 6 Verify 3's fresh-machine simulation** (`lsregister -kill -r -domain local -domain user`) rebuilds your entire Launch Services database and transiently clears every app's handler bindings. Refused on a machine holding a ten-PR test stack, your default-browser binding and a login item.
- **Intel install of an arm64 release** — no Intel Mac available. The failure mode is pinned anyway: `install.sh` refuses with `this release is built for arm64, this Mac is x86_64` *before* touching `/Applications`, exercised for real with a stubbed `uname` in PR #11.

**Reset after this block**
- If you do not want to keep the release: `GH_TOKEN=$(gh auth token --user hexember) gh release delete v0.1.0 --yes` and `git push --delete origin v0.1.0`.
- If you made the repo public and want it private again, change it back in Settings.
- After step 4, `/Applications/ActiveBrowser.app` is the curl-installed copy — that is the intended end state.

## PR #19 — Task 11: default-browser picker listing (`fix/default-browser-listing`, base `main`)

All 11 `ai` steps pass. ActiveBrowser was absent from the System Settings picker because the
bundle claimed `http`/`https` but no HTML content type, so Launch Services never set its
`web-browser` flag. Fixed with `CFBundleDocumentTypes` at `LSHandlerRank: Alternate`. The
earlier "needs a notarized Developer ID" theory was wrong — the bundle is still ad-hoc signed.

| # | Action | Expected |
|---|---|---|
| 1 | System Settings → Desktop & Dock → *Default web browser* → open the dropdown | **ActiveBrowser is listed.** ✅ *confirmed 2026-09-22* |
| 2 | Select `ActiveBrowser` there and accept the macOS confirmation | It becomes the default. No Dock icon, no window. |
| 3 | Focus Brave, then from Terminal `open https://example.com/u1` | Opens in **Brave**. |
| 4 | Focus Arc, then `open https://example.com/u2` | Opens in **Arc**. |
| 5 | Double-click any `.html` file in Finder | Still opens in your **normal** browser — `Alternate` rank must not hijack it. |

**Reset after this block**
- If step 2 was run: System Settings → Desktop & Dock → *Default web browser* → **Arc**.

---

## PR #23 — Task 12: icon + menu bar glyph in the bundle (`feature/bundle-icon-assets`, base `main`)

All 13 `ai` steps pass. PR #20 added the artwork but wired none of it; this PR adds
`CFBundleIconFile`, copies the resources in `make bundle` (before `codesign`, so the ad-hoc
signature seals them), and loads the bundled menu bar glyph with the `globe` SF Symbol as
fallback. Verified on the installed copy: the picker entry and the single Launch Services
record both survive the change.

| # | Action | Expected |
|---|---|---|
| 1 | Look at the menu bar | The ActiveBrowser mark, **not a globe**. Tints correctly in light/dark and while the menu is open. *On a non-Retina external display the 1x PNG is largely antialiasing and may read faint — worth a look if you have one.* |
| 2 | Finder → `/Applications` → ActiveBrowser | The custom app icon, not a generic one. |
| 3 | System Settings → Desktop & Dock → *Default web browser*, and General → Login Items | ActiveBrowser shows its icon in both lists. |
| 4 | Confirm no Dock icon and no window appear | `LSUIElement` still holds — an icon does not make it a foreground app. |

**Reset after this block**
- None. This PR changes no system state; `make clean` removes `build/`.

---

---

# PR #24 — Task 13: default-browser state in the menu row

https://github.com/hexember/active-browser/pull/24

**Preconditions**
- `make install` on the PR branch; ActiveBrowser running from `/Applications`.
- ActiveBrowser is currently the default web browser, so start with step 1.

| # | Action | Expected |
|---|---|---|
| 1 | Click the ActiveBrowser menu bar icon | The row above *Launch at Login* reads **✓ Default Browser**, greyed out; hovering does not highlight it and clicking does nothing. |
| 2 | System Settings → Desktop & Dock → *Default web browser* → **Safari**, then reopen the menu | The row reads **Set as Default Browser**, with no check, and is clickable. |
| 3 | Click *Set as Default Browser* → accept the macOS dialog → reopen the menu | The row reads **✓ Default Browser** again. |
| 4 | (optional) Switch to Safari again, click *Set as Default Browser*, **decline** the dialog, reopen the menu | The row still reads **Set as Default Browser**. |

**Reset after this block**
- System Settings → Desktop & Dock → *Default web browser* → your real browser.


---

## PR #27 — Task 16: links and install.sh point at hexember

**Preconditions:** PR merged to `main`.

| # | Action | Expected |
|---|---|---|
| 1 | Fresh Terminal: `curl -fsSL https://raw.githubusercontent.com/hexember/active-browser/main/install.sh \| sh` | Prints `Resolving the latest release of hexember/active-browser`, installs, ends with the Set-as-Default hint |
| 2 | Open the README on GitHub | ci and release badges render; links go to `hexember/active-browser` |

**Reset after this block**
- Step 1 replaces any dev build in `/Applications`; `make install` to go back.

---

## PR #29 — Task 17: README restructured for users (`chore/readme-restructure`, base `main`)

**Preconditions:** none. This is a docs-only change; read it on the PR.

| # | Action | Expected |
|---|---|---|
| 1 | On the GitHub PR, open *Files changed* → `README.md` → *Display the rendered blob*. Read top to bottom, click the platform badge and the *Why this exists* / *Build from source* / *Install* links. | The page reads as a user landing doc. The badge jumps to *Install*. All in-page links land on the right heading. The menu mock-up matches what you see when you click the menu bar icon (with the `✓ Default Browser` row if ActiveBrowser is your default). No developer-only sections remain. |
| 2 | Same for `CONTRIBUTING.md` and `SECURITY.md` (rendered) | CONTRIBUTING reads in order: build → gotchas → architecture → layout → history → testing → constraints → PRs → releasing. Nothing is said twice. SECURITY's *What the app can see* link opens README *Privacy*. |

**Reset after this block:** none.

---

## PR #30 — Task 18: README published to GitHub Pages (`chore/pages-readme`, base `chore/readme-restructure`)

**Preconditions:** merge PR #29 first, then this PR. The PR-time Jekyll build already passed (run 35777705333), so these steps check only the deploy and the live site.

| # | Action | Expected |
|---|---|---|
| 1 | Merge **PR #29 first**, confirm this PR retargets to `main`, then merge this PR. Open Actions → **pages** → the run for the merge commit. | The `pages` run is green: `build` passes, including "Published set is exactly the allowlist", and `deploy` shows the `github-pages` environment URL. Any `::warning::` about "View on GitHub" is noted. *(Note 2026-09-23: step 10b already proved the build on the PR, so this step now checks only the deploy on `main`.)* |
| 2 | About 2 minutes after the deploy (Pages CDN cache), open https://hexember.github.io/active-browser/ in a browser, hard-refreshed. | It shows the current README: the **Contents** line, and Install showing the `raw.githubusercontent.com/hexember/active-browser/main/install.sh` one-liner. The Cayman header reads "ActiveBrowser" with the tagline and has a **View on GitHub** button that opens github.com/hexember/active-browser. There are no Download .zip/.tar.gz buttons, and the H1 is not repeated under the header. |
| 3 | `curl -s https://hexember.github.io/active-browser/ \| grep -c 'activebrowser\.app'` and, for each of `install.sh`, `Project.html`, `CLAUDE.html`, `tasks/TEST-PLAN.html`, `LICENSE`, `Makefile`, `README.html`: `curl -s -o /dev/null -w '%{http_code} %{url_effective}\n' https://hexember.github.io/active-browser/<path>` | The count is `0`, so the old one-liner is gone. Every path returns `404` (a `README.html` 404 is expected, because README is served at `/`). |
| 4 | On the site, click README's links to **CONTRIBUTING.md** (Build from source → "CONTRIBUTING.md"), **SECURITY.md** (Gatekeeper paragraph), **Code of Conduct** and **CHANGELOG.md**. Then, on the SECURITY page, click **README.md → Privacy**. Also click the in-page **Contents** links. | Each opens the rendered `.html` page on hexember.github.io, not a 404 or a raw `.md` file. `#build-and-run`, `#what-youre-trusting-when-you-install-this` and `#privacy` land on the right heading. The Contents anchors scroll correctly. The LICENSE links open GitHub. |

**Reset after this block:** none. Pages is the intended end state.

---

## PR #32 — Task 19: Showcase video on the README and Pages site (`chore/showcase-video`, base `main`)

**Preconditions:** the PR-time Jekyll build already passed (run 36261053487) and its artifact contains the video, so these steps check only the deploy, the live site in real browsers, and the github.com fallback.

| # | Action | Expected |
|---|---|---|
| 1 | Merge this PR. Actions → **pages** → the run for the merge commit. | `build` passes, including "Published set is exactly the allowlist", and `deploy` succeeds with the `github-pages` URL. |
| 2 | About 2 minutes later, open https://hexember.github.io/active-browser/ (hard refresh) in **Safari**, then in **Chrome**. Also run `curl -sI https://hexember.github.io/active-browser/media/showcase.mp4 \| grep -i '^content-type'`. | In both browsers the video sits between the "That's the whole idea" paragraph and the Contents line, spans the text column at 16:9, and **starts playing by itself, silently**. It **loops** back to the start after about 31 s, the controls can pause it, and the poster shows before playback. There is no poster image duplicated below the video, and no stray `{%` text. The header is `content-type: video/mp4`. (If Safari is in Low Power Mode, autoplay may be blocked and the poster plus play button shows. That's expected; turn Low Power Mode off and retest.) |
| 3 | On github.com, open the README on branch `chore/showcase-video` (before merge) or on `main` (after merge), and click the image under "That's the whole idea." | The poster image shows in the same place, with no empty gap or raw `{% comment %}` text around it. Clicking it opens the project website. (Before merge, the site does not have the video yet.) |

**Reset after this block:** none. The Pages deploy is the intended change.

---

# Final teardown — run this only when you are finished with everything above

Order matters; doing it out of order lets the login item come back.

1. `pkill -x ActiveBrowser`
2. `rm -rf /Applications/ActiveBrowser.app`
3. **Then** System Settings → General → Login Items & Extensions → select ActiveBrowser → **−**
4. System Settings → Desktop & Dock → **Default web browser** → **Arc** (if you ever changed it)
5. `cd` to the repo and `make clean` (removes `.build/` and `build/`, and unregisters the build copy)
6. Optional: `defaults delete com.local.activebrowser` to drop the app's stored settings
