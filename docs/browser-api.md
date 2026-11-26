# Browser command API

After the browser app starts, `window.solidcraft` controls the design open in that tab.
Calls use the same command ids, parameters, control handlers and undo history as the desktop
control channel. No separate engine or server is started.

| Call | Result |
|---|---|
| `solidcraft.request(method, params?)` | Any [control method](control-protocol.md), such as `ui.inspect`, `document.inspect` or `engine.batch` |
| `solidcraft.execute(command, params?)` | Run an engine command without opening its interactive dialog |
| `solidcraft.commands()` | Discover commands, their parameters and current enablement |
| `solidcraft.inspect(params?)` | Inspect the active document, optionally with `{measure: true}` |
| `solidcraft.readFile(path)` | The bytes of a file in the browser's virtual file system, as a `Uint8Array` |
| `solidcraft.writeFile(path, bytes)` | Put a `Uint8Array`, which may come from another window, into the browser's virtual file system at `path` |
| `solidcraft.buttons()` | The mouse buttons the app holds now, as a pointer event's `buttons` bitmask (1 primary, 2 secondary, 4 middle); returns at once, not a promise |

Every call but `buttons()` returns a promise. It resolves to the control reply's `result`, or rejects with an
`Error`. Omitted or `null` parameters become `{}`. Parameters must be JSON serializable and
at most 16 MiB of JSON. At most 256 requests may be pending; excess requests reject without
running. Requests run on the UI thread between frames. A hidden or suspended tab can delay
completion; keep it active while driving the app. A reload ends the session and its pending work.

In the browser's developer console, on an empty design:

```js
await solidcraft.commands();
await solidcraft.request("ui.home", {open: false});
await solidcraft.execute("solid.box", {
  length: 30, width: 20, height: 10, body_name: "Browser API box"
});
await solidcraft.inspect({measure: true});
await solidcraft.execute("edit.undo");
await solidcraft.execute("edit.redo");
await solidcraft.request("ui.inspect");
```

A page that embeds the app passes files in and out with `readFile` and `writeFile`: write a
file, open it with `doc.open` or `file.insert_step`, and read back what `file.save` or
`file.export` wrote to a path. Both take memory paths only, such as `/host/part.step`. Paths
under `/opfs/` and `/downloads/`, and disk paths of the form `/disk/<handle>/<name>`, are
refused, because the app stores or downloads only what its commands write there.
Memory files last until the page reloads.

```js
await solidcraft.writeFile("/host/part.step", new Uint8Array(await response.arrayBuffer()));
await solidcraft.execute("doc.open", {path: "/host/part.step"});
await solidcraft.execute("file.export", {path: "/host/part-edited.step"});
const bytes = await solidcraft.readFile("/host/part-edited.step");
```

An agent with JavaScript evaluation access to this tab can make the same calls. The API does
not register WebMCP tools or expose a network listener. Scripts executing in the page can use
the API with the page's browser permissions. File paths refer to the browser's virtual file
system, not arbitrary files on the user's computer. Native-only operations remain subject to
the browser host's limitations; this API does not grant desktop filesystem or window access.

Call `file.save` and `file.save_as` with standalone `solidcraft.execute` requests. For a file in
browser storage or on the computer, the promise settles after the host finishes writing.
Explicit virtual paths such as `/upload/` stay in memory for the current page.

## A host page that keeps the designs

`?embed=editor` runs the app as an editing embed: the ribbon, browser, timeline and properties,
without the application bar or start page, and with nothing kept in the browser. A page that
frames it from the same origin reaches this API as `frame.contentWindow.solidcraft`, keeps the
designs itself and moves them with three [control methods](control-protocol.md):

```js
const app = frame.contentWindow.solidcraft;
const kept = await app.request("document.export"); // {name, revision, dirty, format: "json", text}
// ...the page stores kept.text, and says so:
await app.request("document.saved", {revision: kept.revision});
// ...and later puts it back, timeline and parameters as saved:
await app.request("document.import", {text: kept.text, name: "Bracket"});
```

The 16 MiB limit on parameters applies to `document.import`'s `text`. A design that links other
design files keeps those links as paths; the files themselves are not in `text`.

## Upstream source and remaining differences

This fork integrates upstream `main` at commit
[`e4811010f558ebaaf2bfdc7de497bfc4d1fea8ef`](https://github.com/tangency-labs/solidcraft/commit/e4811010f558ebaaf2bfdc7de497bfc4d1fea8ef),
retrieved on 2026-11-21. This preserves the fork's policy of tracking upstream commits.
The latest stable release on that date is
[v0.45.0](https://github.com/tangency-labs/solidcraft/releases/tag/v0.45.0), commit
`e8ce12ef78fe76281a04b7242a9ce059ff7d5067` (`v0.45.0-nightly.20261121.151.1` is a prerelease);
the selected `main` commit includes later, unreleased changes (workspace version `0.46.0`).
Upstream supplies `?embed=editor` and `document.export`, `document.import` and `document.saved`.
Those [document handlers](https://github.com/tangency-labs/solidcraft/blob/e4811010f558ebaaf2bfdc7de497bfc4d1fea8ef/crates/ui-egui/src/documents.rs)
exchange native design JSON; they do not transfer the bytes of STEP imports or exports.

| Fork difference | Required behavior | Selected upstream evidence | Action |
|---|---|---|---|
| [`apps/solidcraft-web/src/api.rs`](../apps/solidcraft-web/src/api.rs) | Transfer native and exchange-format bytes through memory paths, including arrays from an embedding window | Upstream exports command methods but no byte-transfer methods | Keep `readFile` and `writeFile` unchanged |
| [`crates/ui-kit/src/window_chrome.rs`](../crates/ui-kit/src/window_chrome.rs) | Keep a held mouse button when focus moves between the canvas and a text input | Native focus recovery also runs on WebAssembly and releases the held button | Read `WindowFocused` events only on native targets; keep explicit picker cleanup and lost-press recovery |
| [`xtask/src/web_offline_test.rs`](../xtask/src/web_offline_test.rs) | Check STEP transfer, reopening and invalid requests in the real browser app | Upstream covers native JSON exchange without the host byte bridge | Keep the existing tests unchanged |
| This document | Record the API contract, selected source and qualification limits | Upstream does not document the fork's byte bridge | Keep and update the evidence |

Upstream's [browser API](https://github.com/tangency-labs/solidcraft/blob/e4811010f558ebaaf2bfdc7de497bfc4d1fea8ef/apps/solidcraft-web/src/api.rs)
does not expose the byte bridge. Remove it when a supported upstream API transfers exchange-format
file bytes, embedding hosts use that API, and the browser test demonstrates exported bytes,
arrays from another window, reopening the transferred file and rejection of invalid requests.
Remove the focus correction when upstream's unchanged browser gate preserves held buttons
across page-element focus changes while still releasing them on actual window blur.

### Native design compatibility

At this revision, newly saved native designs use `solidcraft/64`: `crates/doc/src/format/`
includes folded formats through `v64` and no pending `.rs` changes.
[`Formats::current`](https://github.com/tangency-labs/solidcraft/blob/e4811010f558ebaaf2bfdc7de497bfc4d1fea8ef/crates/format/src/lib.rs)
therefore returns 64. `document.export`'s outer `format: "json"` describes the encoding;
the design's own format number is inside `text`.

The previous published `host-files-2` revision, `9d55e7894d6862e45d9fef99db001ef0107c4404`, writes
`solidcraft/48` and rejects higher format numbers. Keep the original native files when
qualifying an upgrade: restoring that executable alone cannot reopen a design saved in format
64. The newer-file preservation code in v0.42.0+ does not change the older format-48 reader.

### Qualification of the selected main revision

On 2026-11-21, `cargo xtask ci` with Rust 1.99.1 on source merge
`7e66eb4006b04f9d463bc8631ac2f7f9b93e0320` passed formatting (263 files), render assets,
asset attribution (1,173 files), icon lint/status/tests, licence checks, list checks, skill
checks, layering checks, Clippy (`-D warnings`), WebAssembly checks for 34 crates and doc
tests. All 55 `solidcraft-web` tests and all 465 `solidcraft-ui-kit` tests passed. Out of
122 native test binaries, 119 passed (including crates whose only failures are recorded in
`tests/KNOWN_FAILING_TESTS`), and three binaries unchanged from upstream reported 19
unlisted failures under high concurrency (123 concurrent commands):

- `solidcraft-ui-egui` (6,600 passed, 6 failed): two `agent_chat::discover::tests` fixtures
  requiring `/bin/cat` or `cat` in a minimal `PATH` (absent on NixOS), two
  `apply_off_thread_tests` cases, `history_off_thread_tests::design_notifications_run_on_the_command_worker_and_return_panel_data`,
  and `prefs::tests::heal_tolerance_defects_is_on_by_default_and_reaches_the_kernel`.
- `solidcraft-doc` (1,182 passed, 3 failed):
  `eval::cost_tests::a_sketch_on_a_fixed_plane_costs_the_same_in_a_large_placed_design`,
  `tests::a_sketch_line_added_and_deleted_restores_the_extrude`, and
  `tests::results_kept_ahead_outlive_the_cache`.
- `solidcraft-engine` (6,878 passed, 19 failed: 9 recorded in `tests/KNOWN_FAILING_TESTS`
  and 10 unlisted): `cmd::component_link::tests::recovered_relative_links_keep_their_targets_after_save_as`,
  `cmd::drawing::tests::save_as_keeps_drawing_sources_through_undo_redo_and_checkpoints`,
  `cmd::file::tests::a_reused_import_has_a_folder_exactly_when_asked`,
  `cmd::render::job_tests::native_camera_queue_preparation::native_batch_retained_plan_and_completed_images_allow_a_new_ram_epoch_before_next_capture`
  (842,055,261 vs 4,678 bytes), two `plugin::tests::native::*` tests, and four
  `recovery::tests::*` tests.

Re-running all 19 unlisted failures with `--test-threads=1` on both the fork and a clean
checkout of unmodified upstream `e4811010f558ebaaf2bfdc7de497bfc4d1fea8ef` produced
identical results: 17 of 19 passed on both trees (`solidcraft-doc` 3/3, `solidcraft-engine`
10/10, `solidcraft-ui-egui` 4/6), and only the two `/bin/cat` `agent_chat::discover::tests`
fixtures failed deterministically on both trees due to the NixOS host environment. No test
assertion, input, fixture or known-failure list was changed.

`packaging/web/package.sh` built the release browser package from committed source
`7e66eb4006b04f9d463bc8631ac2f7f9b93e0320` with Rust 1.99.1, Trunk 0.22.2 and wasm-bindgen
0.2.129 (removing Trunk 0.22's leftover `.trunk-lock`, running `cargo xtask web-offline dist/web`,
and re-packing with `--skip-build`). The resulting `solidcraft-web-0.46.0.zip` (82 MB) has
SHA-256 `294172eacbf7fbfccc2e00d39fce434d93da8d7536dfbd1bdfb08cacf6a258be`, service-worker
version `0.46.0-466e231fe0e39979` (10 files), and serves `solidcraft-web-9fe12be78e7deb72_bg.wasm`.

`cargo xtask web-offline-test` passed all 20 scenarios. `cargo xtask web-api-test --site dist/web`
passed the complete mouse-button and focus-recovery section (confirming the WebAssembly
`WindowFocused` guard remains effective) and then stopped at the pre-existing upstream
`linked source opens in another tab` assertion before reaching the appended `readFile`/`writeFile`
checks; the full browser gate therefore remains red.

Separate headless Chrome checks against the extracted `solidcraft-web-0.46.0.zip` archive in
`?embed=editor` under a CSP blocking external connections (zero external requests, zero
console or page errors) passed:

- `ui.prefs { theme_import }` with a `solidcraft-theme/1` host theme.
- A 20 × 10 × 5 mm box (1,000 mm³, 700 mm², bounds `[0, 0, 0]` to `[20, 10, 5]`), native
  `document.export` (`solidcraft/64`), revision tracking, stale `document.saved` rejection,
  invalid `document.import` rejection preserving both existing bodies, and dirty-flag clearing
  on current revision acknowledgement.
- Native `.sc` (10,472 bytes) and STEP (9,607 bytes, `ISO-10303-21;`) round trips via
  `readFile` and cross-window `Uint8Array.subarray` with non-zero `byteOffset` in `writeFile`,
  reopening and re-exporting with identical measured geometry.
- Refusals for missing files, `/downloads/`, `/opfs/` and non-`Uint8Array` arguments, plus
  disposal unloading the iframe and rejecting subsequent calls.
- Compatibility with published `host-files-2`: the new bundle opens `host-files-2`'s
  format-48 `.sc` and STEP files with identical measured geometry; `host-files-2` opens the
  new bundle's STEP file with identical geometry and rejects the new format-64 `.sc` file
  (`file format 64; this version reads formats up to 48`) while preserving its open design.

These synthetic checks do not qualify a consuming application's authenticated storage or
permissions.

### Previous qualification on the 2026-11-17 main revision

The following results apply to upstream `df1537606dcb477d89f9e5ae95b2be9c9e3cb6f7`
(native format `solidcraft/58`) and do not qualify the newer source above.

On 2026-11-17, before the focus correction, `cargo xtask ci` with Rust 1.99.1
passed formatting, source checks, all 16 isolated shared-crate checks, Clippy,
the WebAssembly checks for 30 crates
and doc tests. Its 112 native test binaries completed with 16 distinct failures
across six binaries. Thirteen failures are recorded in the selected source's
[`tests/KNOWN_FAILING_TESTS`](https://github.com/tangency-labs/solidcraft/blob/df1537606dcb477d89f9e5ae95b2be9c9e3cb6f7/tests/KNOWN_FAILING_TESTS).

Two agent-discovery fixtures failed because they restrict `PATH` to `/usr/bin:/bin`
or invoke `/bin/cat` directly. Neither location supplies `cat` on the NixOS test
host. Their failing calls are in `crates/ui-egui/src/agent_chat/discover.rs` at
lines 1007 and 602. Both unchanged tests pass when their existing test binary runs
inside a temporary Nix `buildFHSEnv` with Bash and coreutils. That check isolates
the missing filesystem paths; it is not another full-suite result.
The render-memory assertion at
`crates/engine/src/cmd/render/job_tests.rs:3329` expected 4,678 admitted bytes and
observed 228,935,237. The native failure stops `cargo xtask ci` before its browser
offline stage. No test assertion, input, fixture or selection was changed.

In a clean checkout of the selected upstream commit, `cargo xtask test` also
exited with status 1. It ran all 112 native test binaries and doc tests inside
the same temporary Nix environment, with two concurrent commands and eight
test threads per binary. Its 14 distinct failures across five binaries match
the fork's remaining failures exactly: the 13 recorded failures and the same
render-memory assertion, including both byte counts. The full UI library
passed 5,844 tests, with its 22 ignored tests unchanged.

After the focus correction, a full `cargo xtask ci` run in the temporary Nix
environment completed all 112 native binaries and both side jobs, then exited
with status 1. Formatting, source checks, all 16 shared-crate checks, Clippy,
the 30-crate WebAssembly check and doc tests passed. The native suite reported
15 distinct failures across six binaries: all 14 pristine-baseline failures and
one additional display-quality failure. The 379 UI-kit tests passed, including
the existing native focus and picker-cleanup cases.

The additional assertion at `crates/ui-egui/src/apply_off_thread_tests.rs:541`
expected the dialog to be closed, but it remained open. The corrected UI library
passed 5,843 tests, failed that one test and retained its 22 ignored tests. The
current pristine baseline passed that test. The same failure was recorded for
the older pristine upstream revision below, and this test file is unchanged
between those upstream revisions. Three isolated runs of the corrected suite's
existing test binary passed; those runs do not resolve the full-suite failure.
No test assertion, input, fixture or selection was changed.

An earlier corrected CI attempt ended with status 137 during the shared-crate
checks, without a reported cause. The completed retry used eight Cargo jobs,
two concurrent test commands and eight test threads per binary. The native
failure still prevents CI from reaching its browser offline stage.

Before the focus correction, `bash packaging/web/package.sh` built the fork and a clean
checkout of the selected upstream commit successfully. Both real browser builds failed
`cargo xtask web-api-test --site dist/web` at `xtask/src/web_offline_test.rs:194`:
moving focus to the hidden text input left no button held, although the browser still
held the middle button. The fork failed in all three runs; pristine upstream failed in
its one run. The failure occurs before the gate reaches its file-transfer checks.

The shared native focus-recovery helper treated eframe's page-element focus changes
as window focus loss. The correction limits those `WindowFocused` scans to native
targets. The browser's pointer tracker still handles actual window blur, cancellation
and lost releases. Explicit picker cleanup and the native focus-recovery path remain
in place. No existing browser or native test was changed.

With that correction, all three browser-gate runs passed the complete mouse section,
including focus moving within the page and actual window blur. Each run then failed
the unchanged linked-source assertion at `xtask/src/web_offline_test.rs:316`.
The unchanged `API_TEST` body also failed there when run separately against the
pristine upstream browser build. At the assertion, `ui.documents` returned one tab
and `ui.inspect` reported `file_busy: true`; after loading completed, they returned
two tabs, the source active at index 1, and `file_busy: false`. This diagnostic does
not replace the full gate, which remains red.

The release package built with Rust 1.99.1, wasm-bindgen 0.2.129 and Trunk 0.22.1.
Its 15 Rust warnings also occur in the pristine upstream browser build; they are
in unchanged source. Trunk 0.22.1 leaves an empty `.trunk-lock` in the final site
after the post-build hook, so the offline gate initially rejected the generated
manifest. The selected upstream package had the same extra file. The release workflow
pins Trunk 0.21.14. Running `cargo xtask web-offline dist/web` on the completed site,
then `bash packaging/web/package.sh --skip-build`, regenerated the metadata through
the supported tools. No generated file was edited by hand. The real-site
`cargo xtask web-offline-test --site dist/web` then passed all its scenarios.

The resulting `solidcraft-web-0.44.0.zip` has SHA-256
`5781cfbcc13175bf65cb975a5e808e82623540e8450a28a2c25bba412e1940b3` and service-worker
build id `0.44.0-01c52b9a6cea4a53`. It was built from the production source at merge
`bdf398fdef4c14fc496c08fc5dd01661ba439803` plus the focus correction above:
`crates/ui-kit/src/window_chrome.rs` has Git blob
`92608e8e0ddc4e92298fd7c2cdc7d4cfb6ab38c1`. Its embedded About stamp says
`bdf398fde`, the base commit during the build. The archive hash and source blob
identify the corrected artifact.
Archive integrity and the packaged licence notices were checked. The artifact has
not been published or deployed.

Separate checks against that artifact in `?embed=editor` passed native SC and STEP
round trips through arrays created by the embedding window, including a subarray
with a nonzero byte offset. A 20 by 10 by 5 mm box retained its volume of 1,000 cubic
millimetres, surface area of 700 square millimetres and bounds. Native SC retained
the editable box feature and its dimensions. A stale save acknowledgement and an
invalid import were rejected; the invalid import preserved the open design. Missing
files, reserved storage paths and invalid byte arguments were rejected.
Native save/reopen and STEP export also passed in a separate same-origin iframe
whose CSP restricted network and asset requests to that origin. WebAssembly,
inline scripts and styles, and blob workers were allowed.

The current reader opened `host-files-2`'s format-48 native file and its STEP file.
The published older reader rejected the new format-58 native file and preserved
its open design. STEP exchange passed in both directions with the same measured
geometry. STEP reopened as an imported base feature, so it does not preserve the
editable native feature timeline. These synthetic checks do not qualify a consuming
application's authenticated storage or permissions.

### Historical qualification of the previous main revision

The previous integration selected upstream
[`7e0c36cf83e44a913dd7e5d3dba295bb78d52de6`](https://github.com/tangency-labs/solidcraft/commit/7e0c36cf83e44a913dd7e5d3dba295bb78d52de6),
after v0.42.0, and wrote native format 57. Its results below do not qualify the current source.
A browser check confirmed that `host-files-2` rejected a format-57 native file and
preserved its open design. STEP exchange passed in both directions for the
synthetic box described below.

With Rust 1.99.1, `cargo xtask ci` passed formatting, source and shared-crate
checks, Clippy, the WebAssembly checks for 30 crates and doc tests. Its native
suite reported 18 distinct failures across six binaries. Thirteen were in the
selected revision's `tests/KNOWN_FAILING_TESTS`.

Four additional failures also occurred when `cargo xtask test` ran the unmodified
upstream revision: two agent-discovery fixtures cannot find `cat` (one uses
`/bin/cat`), a display-quality test leaves an unexpected dialog open, and a
render-cache test reports retained bytes above its expected count. That baseline
reported the same 17 failures. A third agent-discovery test, which expected a
synthetic executable's version and received none, failed only in the fork run;
its cause remains unresolved. Tests and the known-failure list are unchanged.

The release browser package built, and `cargo xtask web-offline-test` passed.
The real-app `cargo xtask web-api-test --site dist/web` failed in all three runs
at `xtask/src/web_offline_test.rs:194`: moving focus to the hidden text input
leaves no button held, although the test expects the middle button. The mouse
handler and this assertion are unchanged from the selected upstream source.
An independent upstream browser build was not tested, and the full browser
gate remained red. No assertion was changed or skipped.

Separate checks in `?embed=editor` passed native SC and STEP round trips through
the host byte bridge. A 20 by 10 by 5 mm box retained its volume of 1,000 cubic
millimetres, surface area of 700 square millimetres, bounds and native feature
timeline. That reader opened the published bundle's native file, and
STEP files passed in both directions. A stale save acknowledgement was rejected;
an invalid import preserved the open design. Missing files, reserved storage
paths and invalid byte arguments were rejected. These synthetic checks do not
qualify authenticated storage or permissions in a consuming application.

### Historical qualification of the v0.42.0 integration

The earlier integration based on upstream commit
[`1b1e940cc48e4d1d5d01971a4be20e2e55647066`](https://github.com/tangency-labs/solidcraft/commit/1b1e940cc48e4d1d5d01971a4be20e2e55647066)
was checked with Rust 1.99.1. In that run, `cargo xtask ci` passed formatting,
source checks, Clippy, the WebAssembly checks for 30 crates and doc tests.
The native suite reported 19 failures across six test binaries. Fifteen were recorded in
[`tests/KNOWN_FAILING_TESTS` at that revision](https://github.com/tangency-labs/solidcraft/blob/1b1e940cc48e4d1d5d01971a4be20e2e55647066/tests/KNOWN_FAILING_TESTS).
Two agent discovery tests failed because their runtime fixtures could not find `cat` on
NixOS, including one that invoked `/bin/cat` directly. A display-quality test unexpectedly
left a dialog open, and a render-cache test reported a different retained byte count.
The assertions and known-failure list were unchanged in that run. The browser bundle and
native and STEP round trips were not qualified. These historical results do not qualify
the newer upstream revision selected above.

## Verify a browser build

Build the app with `trunk build` in `apps/solidcraft-web`, then run from the repository root:

```sh
cargo xtask web-api-test --site dist/web
```

The test starts a local server and headless Chrome, discovers commands, creates a box in the
live document, checks undo and redo, and verifies error handling and the pending-request limit.
It also checks that Ctrl+Tab and Ctrl+Shift+Tab are left to the browser, which switches tabs with them.
It checks that the app's mouse buttons follow the browser's after chords (two buttons held
at once), a drag let go outside the window, a release the page never heard of and the window losing focus,
and that the focus moving inside the page (to a text field's hidden input and back) lets go of none.
Last, in a tab of its own, it makes the drawing milestone's test plate and its drawing by commands
(`examples/acceptance/drawing_m2.json`) and exports the sheet as SVG and PNG. It then starts the app
again with the design put back (`document.export`, `document.import`): the drawing reads and exports the same.
Set `CHROME` to a Chrome or Chromium executable if it is not installed in a usual location.

For a short startup check, use `cargo xtask web-api-test --smoke --site dist/web`.
It starts a separate headless Chrome with a fresh profile and reports early JavaScript or wasm errors.
It creates a synthetic 30 × 20 × 10 mm box, saves it, opens the saved design and checks its
6000 mm³ volume. It then changes the camera three times and requires different frames through
`ui.screenshot`, whose request tokens identify frames painted after each camera command.
Each run uses a free local server port; it never controls an existing browser.
The smoke runs in the WebAssembly job in the merge queue, for a group that touches the web app or
what it builds from, and nightly; pull requests skip it so their CI stays fast. The full API and offline tests still run in Pages.
