# PR #77 regression and mergeability audit

## Review update — 2026-10-02

Reviewed PR head `fea0798` against current main `34d26f1`, then resolved the merge in an isolated worktree. The update preserves main's extracted command/session handlers, menu lifetime guards, shared text helpers, resize key table, and test helpers. Main deleted `docs/usage.md`; the responsive sidebar instructions now live in README instead of restoring that obsolete document.

### Standards

Independent review found no documented-standard violations or actionable introduced code smells. Existing TUI tests were adapted to the new controls and width semantics; no test cases were added.

### Spec

One P3 robustness defect was found and fixed: fullscreen Pi normalizes invalid terminal dimensions to one column before evaluating layout visibility. That normalized width reset Auto expansion history. The resolver now also validates the terminal's raw reported width. This affects custom Terminal implementations exposing invalid dimensions; stock Pi substitutes a fallback for zero columns.

A manual in-memory replay through Pi 0.84.0's real `TuiAltScreen`, `renderLayoutFrame`, and overlay/image composition reproduced `124 shown → raw 0 → 124 collapsed` before the fix. After the fix:

```text
124 → shown
  0 → too-narrow
124 → shown
 -1 → too-narrow
124 → shown
123 → auto-collapsed
131 → auto-collapsed
132 → shown
```

No other concrete spec deviations were found. Mode/visibility independence, width preservation, cancellation, lifecycle cleanup, TODO output, and selection/image geometry were reviewed; static review does not establish all visual behavior.

### Validation and observed terminal behavior

`npm run check` passes: TypeScript, Biome, 453 existing tests in 23 files, package contents (35 files), dependency audit (zero vulnerabilities), and packed-package installation using host-provided Pi packages. The test count reflects current main's test consolidation. Dependencies were freshly installed with `npm ci --ignore-scripts`.

Environment: macOS 26.3 arm64, Node 22.22.2 from fnm, Pi 0.84.0, 40 terminal rows. The default Homebrew Node remains unusable because its simdjson dylib is missing.

- Regular and fullscreen, Auto: 140 → 123 → 131 → 132 columns showed → hid → stayed hidden → reopened, reclaiming main-pane width.
- Fullscreen, Manual: Ctrl+Shift+R entered resize; Left grew one column; Escape restored the width and command input.
- Fullscreen: Off → Auto → widen to 150 stayed hidden.
- Regular, Manual: 91 columns hid the sidebar; 92 restored it at minimum width.
- Both ephemeral sessions exited normally with cursor restoration. Fullscreen PTY verification preceded the invalid-width guard; regular PTY verification and the fullscreen renderer replay included it.

[Selected terminal output](issue-76-terminal-review-2026-10-02.txt) records this review. The remaining manual TODO below is still pending, including native image/clipboard checks and target-laptop streaming comfort. No GitHub CI checks were attached at review time.

## Previous audit — 2026-09-26

Reviewed 2026-09-26: PR head `a8b8759`, integrated with main `371b847`.

## Findings and resolution

- GitHub reported a merge conflict. The conflicting root-menu assertion now preserves main's removal of Actions and this PR's Manual status label.
- Added the missing Unreleased changelog entry.
- Restored the existing visibility-toggle test's original purpose through the new Sidebar submenu. No new TUI tests were added.
- Independent Standards and Spec reviews found no concrete introduced TUI regression. The remaining visual checks below are not established by static review or the automated gate.

## Validation

`npm run check` passed: TypeScript, Biome formatting, 478 existing tests in 20 files, and package verification. `git diff --check` passed.

Environment: macOS 26.3 (25D125), arm64, Node 22.22.2, Pi 0.84.0. The default Homebrew Node was unusable due to a missing simdjson dylib; checks used the installed fnm Node 22 runtime. Dependencies were reused from the existing checkout after a fresh install stalled. No application source changes were needed during this audit.

Manual interaction with real Pi PTYs, always 40 rows:

| Scenario | Observed result |
| --- | --- |
| Regular and fullscreen, Auto, 140 → 123 → 131 → 132 columns | Sidebar shown → hidden → hidden → shown; main pane reclaimed the width |
| Regular, Off → Manual → On at 132 columns | Hidden state retained until explicit On |
| Regular, Manual, 91 → 92 columns | Hidden → shown at minimum sidebar width |
| Fullscreen, Auto, Ctrl+Shift+R | Warning to select Manual; no resize gesture |
| Fullscreen, Manual, Ctrl+Shift+R → Left → Escape | Resize guidance shown, width grew one column, then restored |
| Fullscreen, active Manual resize, 132 → 91 → 132 columns | Gesture cancelled; guidance removed; subsequent command input worked |
| Fullscreen, Off → Auto at 132 columns | Remained hidden |
| Exit both sessions using Ctrl+D | Process exited normally and emitted cursor restoration |

[Terminal output excerpts](issue-76-terminal-excerpts.txt) preserve selected emitted lines with ANSI removed. They are not full screenshots and do not establish native graphics or clipboard behavior.

Start an isolated temporary session from this checkout:

```sh
./node_modules/.bin/pi --no-session --no-extensions -e ./extensions/index.ts --tui-mode regular
```

Repeat with `--tui-mode fullscreen`. Only the checkout extension is loaded.

## Remaining manual TODO

- [ ] In Herdr on the target laptop, resize panes during streaming and confirm reading comfort.
- [ ] Check native images and subagent plots, transcript selection/copy, modal dialogs, cursor placement, and short terminal heights across collapse/reopen.
- [ ] Check mouse dragging/release, Enter commit, and mode changes during a live resize gesture; verify preferred width survives each transition.
- [ ] Verify live TODO/activity/contributed-panel data after reopening, complete hidden TODO output, and session switch/reload/disable/enable cleanup.
- [ ] Exercise F6 → Controls → Sidebar and startup visibility disabled through the UI.

No GitHub CI checks were attached to the reviewed head. Local validation is the automated evidence; this audit does not claim full visual coverage.
