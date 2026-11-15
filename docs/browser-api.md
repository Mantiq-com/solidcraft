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
under `/opfs/`, `/disk/` and `/downloads/` are refused, because the app stores or downloads only
what its commands write there. Memory files last until the page reloads.

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
[`7e0c36cf83e44a913dd7e5d3dba295bb78d52de6`](https://github.com/tangency-labs/solidcraft/commit/7e0c36cf83e44a913dd7e5d3dba295bb78d52de6),
after [v0.42.0](https://github.com/tangency-labs/solidcraft/releases/tag/v0.42.0).
Upstream supplies `?embed=editor` and `document.export`, `document.import` and `document.saved`.
Those [document handlers](https://github.com/tangency-labs/solidcraft/blob/7e0c36cf83e44a913dd7e5d3dba295bb78d52de6/crates/ui-egui/src/documents.rs)
exchange native design JSON; they do not transfer the bytes of STEP imports or exports.

The remaining fork change is the `readFile` and `writeFile` bridge in
[`apps/solidcraft-web/src/api.rs`](../apps/solidcraft-web/src/api.rs), with its documentation
here and browser coverage in [`xtask/src/web_offline_test.rs`](../xtask/src/web_offline_test.rs).
Upstream's [browser API](https://github.com/tangency-labs/solidcraft/blob/7e0c36cf83e44a913dd7e5d3dba295bb78d52de6/apps/solidcraft-web/src/api.rs)
does not expose that bridge. Remove it when a supported upstream API transfers exchange-format
file bytes, embedding hosts use that API, and the browser test demonstrates exported bytes,
arrays from another window, reopening the transferred file and rejection of invalid requests.

### Native design compatibility

At this revision, newly saved native designs use `solidcraft/57`: the
[format generator](https://github.com/tangency-labs/solidcraft/blob/7e0c36cf83e44a913dd7e5d3dba295bb78d52de6/crates/format/src/build.rs) includes folded formats through `v56` and
five pending changes, and [`Formats::current`](https://github.com/tangency-labs/solidcraft/blob/7e0c36cf83e44a913dd7e5d3dba295bb78d52de6/crates/format/src/lib.rs) advances the number
when pending changes exist. `document.export`'s outer `format: "json"` describes the encoding;
the design's own format number is inside `text`.

The previous `host-files-2` revision, `9d55e7894d6862e45d9fef99db001ef0107c4404`, writes
`solidcraft/48` and rejects higher format numbers. Keep the original native files when
qualifying an upgrade: restoring that executable alone cannot reopen a design saved in format
57. The newer-file preservation code in v0.42.0 does not change the older reader.
A browser check confirms that `host-files-2` rejects a format-57 native file and
preserves its open design. STEP exchange passes in both directions for the
synthetic box described below.

### Qualification on the selected main revision

With Rust 1.99.1, `cargo xtask ci` passes formatting, source and shared-crate
checks, Clippy, the WebAssembly checks for 30 crates and doc tests. Its native
suite reports 18 distinct failures across six binaries. Thirteen are in the
selected revision's `tests/KNOWN_FAILING_TESTS`.

Four additional failures also occur when `cargo xtask test` runs the unmodified
upstream revision: two agent-discovery fixtures cannot find `cat` (one uses
`/bin/cat`), a display-quality test leaves an unexpected dialog open, and a
render-cache test reports retained bytes above its expected count. That baseline
reports the same 17 failures. A third agent-discovery test, which expected a
synthetic executable's version and received none, failed only in the fork run;
its cause remains unresolved. Tests and the known-failure list are unchanged.

The release browser package builds, and `cargo xtask web-offline-test` passes.
The real-app `cargo xtask web-api-test --site dist/web` fails in all three runs
at `xtask/src/web_offline_test.rs:194`: moving focus to the hidden text input
leaves no button held, although the test expects the middle button. The mouse
handler and this assertion are unchanged from the selected upstream source.
An independent upstream browser build has not been tested, and the full browser
gate remains red. No assertion was changed or skipped.

Separate checks in `?embed=editor` pass native SC and STEP round trips through
the host byte bridge. A 20 by 10 by 5 mm box retains its volume of 1,000 cubic
millimetres, surface area of 700 square millimetres, bounds and native feature
timeline. The current reader opens the published bundle's native file, and
STEP files pass in both directions. A stale save acknowledgement is rejected;
an invalid import preserves the open design. Missing files, reserved storage
paths and invalid byte arguments are rejected. These synthetic checks do not
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
Last, it checks that the app's mouse buttons follow the browser's after chords (two buttons held
at once), a drag let go outside the window, a release the page never heard of and the window losing focus,
and that the focus moving inside the page (to a text field's hidden input and back) lets go of none.
Set `CHROME` to a Chrome or Chromium executable if it is not installed in a usual location.
