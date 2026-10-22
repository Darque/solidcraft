# Local Security and Data-Egress Baseline

- **Reviewed:** 2026-10-23
- **Repository:** `/Users/phil/git/darque/solidcraft`
- **Branch:** `local/solidcraft-personalizations`
- **Upstream source commit:** `030187452ab71e44e31d9ea25c3d8aa70eb760ee` (`upstream/main`)
- **Previous source base:** `6c60f1a9ed1fe61bc5cd8390e49e74b0655e72d5`
- **Reviewed source range:** `6c60f1a9..03018745`
- **Current local overlay diff SHA-256 (excluding this report):** `c55edf727956773015cf7a57b518510660f68926c79bdee5b7360b7c0195b134`
- **Build and launch (2026-10-23):** `CARGO_TARGET_DIR=/Users/phil/git/darque/solidcraft/target cargo build --locked --release -p solidcraft` succeeded in 3m16s. Started `./target/release/solidcraft --sample`; startup reported Metal graphics with 4x MSAA, and process 14838 remained active. Process inspection found this was the only running SolidCraft process. Tests were not run.

The hash is SHA-256 of the binary diff from `upstream/main` to the local overlay working tree,
excluding this report. It covers the fork workflow, trackpad navigation changes, Browser tree
persistence and selection highlighting, and related documentation. These local changes add no
communication behavior.

## Summary

The desktop CAD/modeling work runs on the user's machine. SolidCraft is not strictly offline:
release builds can check GitHub for updates, the optional AI panel can send conversation and design
context to a configured model or installed coding agent, and issue-reporting tools can open a
user-reviewed draft on GitHub. The browser build fetches and updates its static application files
from the site origin hosting it.

I found no direct analytics or telemetry SDK, automatic crash-report uploader, or unexplained
background transfer of CAD designs in the reviewed application source. The network and process
paths found have identifiable product purposes and are listed below. This is a static source review,
not proof that every transitive dependency or runtime environment is safe.

## Communication and process paths

| Path and source | Trigger and destination | Data and purpose | Assessment |
| --- | --- | --- | --- |
| Native update check — `crates/ui-egui/src/update.rs` (`LATEST_API`, `start`); `apps/solidcraft/src/main.rs` (`http_get`, `http_call`) | Automatic at startup and about once per day for release builds when **Check for updates automatically** is enabled; a user can also choose **Check now**. Request goes to `https://api.github.com/repos/bherbruck/solidcraft/releases/latest`. | HTTPS GET for release metadata. The request has a generic `SolidCraft-update-check` user agent and GitHub API headers; it does not include a design, user account identifier, or usage history. GitHub will see ordinary request metadata such as the client's IP and time. | Expected update feature. Development builds do not check. The API URL still names the previous `bherbruck` owner; the source documents that GitHub redirects after the project transfer and accepts release links under both owners. |
| Native update download/install — `crates/ui-egui/src/update.rs`; `crates/ui-egui/src/update/install.rs`; `apps/solidcraft/src/main.rs` (`http_download`) | Only after the user chooses **Update now** or opens a release download. GitHub release assets may redirect to GitHub's download host. | Downloads the selected package and its checksum/signature files. It limits size, restricts release URLs to the SolidCraft GitHub repository, verifies the checksum list with the pinned minisign release key, checks the package SHA-256, and only then applies the update. | Expected, user-initiated update. The signature protects against an altered GitHub response/package without the release signing key. No silent package install was found. |
| OpenAI-compatible model API — `crates/ui-egui/src/agent_chat/api_client.rs`; `crates/ui-egui/src/agent_chat/agents.rs` (`key_safe_for`); `apps/solidcraft/src/main.rs` (`agent_http`) | Only when the user adds/tests/connects an API agent or sends a message. Destination is the endpoint the user configured, such as local Ollama/LM Studio or a chosen hosted service. | Sends model-list requests during endpoint tests and chat-completions requests containing conversation history, tool definitions, model replies, and tool results. User text, attached files/pictures, and optional view imagery can therefore leave the machine; tool results may contain design information. An API key is kept in the OS keychain and may be sent only over HTTPS or to loopback. Redirects are disabled. | Expected but privacy-sensitive and user-configured. Remote endpoints should use HTTPS. The code permits a no-key custom endpoint over plain HTTP, so users should not send private content to a remote `http://` endpoint. |
| Installed/custom ACP agent process — `crates/ui-egui/src/agent_chat/acp_client.rs`, `process.rs`, and `discover.rs`; documented in `docs/agent-chat.md` | On first opening the AI panel/settings, SolidCraft may run discovered programs with `--version`. It starts an agent session only after **Connect** or sending a message. The process communicates with SolidCraft over local stdin/stdout. | The agent receives prompts, attached content, and SolidCraft tool results. SolidCraft removes `SOLIDCRAFT_CONTROL_TOKEN` from the child environment, but the process otherwise runs as the user and inherits the remaining environment. The external agent can have its own network, shell, and filesystem behavior. | This is an optional third-party executable selected or installed by the user, not a SolidCraft upload service. SolidCraft's MCP permission gate applies to SolidCraft tools; it is not an OS sandbox for the external process. Review and trust the agent software/account separately. |
| App control and MCP bridge — `apps/solidcraft/src/control_server.rs`; `crates/engine/src/control_auth.rs`; `crates/mcp/src/backend.rs`; `apps/solidcraft-cli/src/main.rs` | The desktop control listener starts only when requested with `--control`/`SOLIDCRAFT_CONTROL_PORT`; an on-demand loopback listener is also used for AI-agent tools. CLI `mcp --connect HOST:PORT` connects to the supplied address. | Local JSON-line requests can inspect or change the open design and return tool results. The listener binds to `127.0.0.1`, requires a fresh random token, and stores that token in a user-only file (Unix `0700`/`0600`; Windows user ACL). | Local IPC by default, not Internet traffic. **Caution:** the CLI accepts a non-loopback host too; its TCP stream is not TLS-encrypted and the token is sent in the request. Use loopback unless a remote connection is deliberate and protected by a trusted network. |
| Crash and issue reports — `crates/engine/src/crash.rs`; `crates/ui-egui/src/crash_ui.rs`; `crates/ui-egui/src/issue_report.rs` (`draft_for`, `crash_draft`, `draft_url`, `show`) | Crash details are saved locally. Nothing is uploaded automatically. A user can review/edit a report and choose **Open GitHub draft**; the browser then opens `https://github.com/tangency-labs/solidcraft/issues/new`. Submission is a separate GitHub action. | The local crash folder may contain a design snapshot, replay commands, command parameters, backtrace, app/system details, and a minidump. The draft can include selected items, feature/command parameters, diagnostics, camera and crash context; it does not serialize the entire design for a normal modeling issue. A short draft body is placed in the GitHub URL; for a long one the report is copied to the clipboard for the user to paste. Screenshots are attached manually. | Expected, explicit sharing flow. Review for private names, paths, parameters, or geometry before opening/submitting. Opening the draft sends its URL (including a short encoded body) to GitHub; the app does not directly upload the design or screenshot. |
| Web app assets and updates — `apps/solidcraft-web/sw.js`; `apps/solidcraft-web/src/main.rs` (`fetch_font_bytes`) | On first visit and when a new build is available, the browser fetches the app's `build.json` and static build files. The Japanese font is fetched from `./fonts/...` when needed. | Same-origin page, JavaScript/WASM, font, and build metadata only. The worker intercepts same-origin GET requests, caches files, and checks each build file against hashes in the origin's build manifest. | Normal web-app delivery/update. The request goes to the origin from which the web app is served. No design data is included in these asset requests. A compromised hosting origin could still publish a new app and matching manifest; the hashes ensure build consistency, not independent publisher authentication. |
| Help, release, and issue links — `crates/ui-egui/src/help.rs`, `home.rs`, and `update.rs` | Only when a user clicks a help, release, or issue link. Destinations include `docs.solidcraft.io` and GitHub. | Browser navigation to the requested page. | Expected user-directed browsing; not background telemetry. |
| 3D mouse support — `apps/solidcraft/src/space_mouse.rs`, `crates/threedx-client` | When a supported device/driver is available. | Reads motion/buttons from local HID, the OS 3Dconnexion driver, or `spacenavd`. | Local device/driver IPC; no Internet destination found in this path. |

## Previous source-update review (b168750f..fc3dedfd)

Within `b168750f..fc3dedfd`, I found no new general-purpose network client, telemetry endpoint, or
new remote host. The direct manifest/lock changes add JSON schema/error-path support (`schemars`,
`serde_path_to_error`) and test-only syntax parsing (`syn`); they do not add a network client. The
new browser disk-file support uses browser file-picker and IndexedDB APIs to retain file handles;
it is local file access, not an upload feature. The updated issue-report presentation still requires
user review and browser navigation before any report is sent.

Direct communication mechanisms found in the application source are native `ureq` HTTP for update
checks and desktop-configured AI APIs, browser `fetch` for same-origin assets, and TCP for the
loopback control/MCP bridge (plus the explicitly configurable CLI host). I found no direct `opentelemetry`, Sentry,
PostHog, Segment, Amplitude, Mixpanel, Bugsnag, Rollbar, or New Relic package in `Cargo.lock`.

## Incremental source review (fc3dedfd..9b801634, 2026-10-21)

### Feature changes

- Geometry and file interchange: crates/kernel/src/whole_face.rs adds whole-face checks with focused tests; crates/kernel/src/step_in and iges_in improve STEP/IGES imports, including cone-to-point and small-knot cases, nested imported parts, assembly structure and meaningful names. crates/io/src/threemf.rs expands 3MF production-extension handling. crates/doc/src/eval.rs and sheet.rs expand whole-face and sheet-metal evaluation. Regression coverage was added across kernel, engine, document and IO crates.
- Modeling and assembly commands: crates/engine/src/cmd/features.rs, face.rs, browser.rs, sketch.rs and related tests improve operations on placed components, split-body naming, extrude/join/cut behavior, joint deletion and selection-aware movement. The document schema was reorganized into per-command feature contracts and shared field types under crates/engine/src/schema/; validation and inspect tests were added.
- Sketch and viewport workflow: sketch creation, constraints, symmetry, 2-tangent circles, lofts, shortcuts, sheet-metal tabs, tooltips, section analysis and previews received focused behavior fixes and regression scenarios in crates/ui-egui/src/ and crates/sketch/src/lib.rs. viewport.rs now includes the visible world origin in pick-cache invalidation and allows picking that origin for inputs which accept it.
- Documents and recovery: crates/engine/src/recovery.rs, crates/ui-egui/src/recovery_ui.rs, documents.rs, browser.rs, and crates/io/src/design.rs improve autosave/recovery across open tabs, browser-local recent files and large STEP design storage. Browser file handles are retained locally by browser APIs.
- MCP and AI panel: MCP batch rollback now reports whether restoration succeeded. In crates/ui-egui/src/agent_chat/api_client.rs, a specific HTTP 400 refusal for tool calls with reasoning effort retries once with reasoning_effort: "none" and remembers that setting for later requests. The AI settings dialog can also be opened from Home.
- Tooling and localization: the release version advances to 0.18.2, English (UK) strings and help text are added, and CI moves normal tests onto one larger runner while using a pinned Blacksmith sticky-disk action for compiler caching. This is CI infrastructure; it does not run in the desktop application.

### Local navigation overlay

The upstream ranges through `cbac4eab` do not change `crates/ui-egui/src/navigation.rs` or add Fusion-style two-finger trackpad navigation. The local overlay still provides pinch zoom, two-finger pan and Shift+two-finger orbit for mouse navigation presets. I carried it forward to `cbac4eab` while retaining the upstream `viewport.rs` world-origin picking/cache change. The overlay adds no communication behavior.

### Incremental security and data-egress findings

No new runtime destination, telemetry SDK, automatic crash-report uploader, shell launch path, or direct dependency was introduced in this source range. Cargo.toml and Cargo.lock change the workspace version to 0.18.2 without adding a direct package dependency.

The only changed application HTTP behavior is the configured AI API path in Worker::request/Worker::post (crates/ui-egui/src/agent_chat/api_client.rs). A user-initiated chat request that receives the specific 400 response may send the same conversation/tool request once more, with reasoning_effort: "none", to the already configured endpoint. Existing endpoint selection, HTTPS/loopback policy and optional bearer-key handling remain in force; no new destination or automatic conversation upload was added.

The CI workflow adds the pinned useblacksmith/stickydisk action for a shared sccache directory. It is remote CI infrastructure, not application runtime traffic; the workflow configures main to save the cache while pull requests and forks mount it without writing changes back. This is an expected build-cache path and is separate from CAD data handling.

No unexplained egress was identified in this incremental diff. The conclusions remain limited to static review; they do not establish transitive dependency or runtime safety.

## Incremental source review (9b801634..cbac4eab, 2026-10-21)

### Feature changes

- Face-attached sketches: `state_in_face_anchor_frame` in `crates/doc/src/anchor_frame.rs` now reuses the model's own `ModelBody` when the anchor transform is identity, preserving its display-mesh cache. Transformed bodies still use the existing transform-and-rebuild path. This avoids meshing a throwaway copy while projecting face edges and then meshing the same body again for the next feature. `crates/engine/src/cmd/sketch_face_tests.rs` adds `a_sketch_on_a_face_meshes_its_body_once` to check the cache is populated after sketch projection.
- Off-thread feature dialogs: `Applying` in `crates/ui-egui/src/dialogs.rs` now tracks whether the job has been visible for a frame; `ready()` requires both that frame and a finished result. The dialog therefore presents its computing state before applying a result, including when the worker finishes quickly but the window thread was delayed. `crates/ui-egui/src/apply_off_thread_tests.rs` adds a regression test for the frame in which the job starts and updates fixtures for the new state.

### Incremental security and data-egress findings

This range changes four Rust source/test files only. It adds no runtime destination, network client, telemetry, subprocess, credential handling, upload path, or direct dependency. The changed execution paths are model-body cache reuse and off-thread feature-dialog scheduling; neither performs communication. The static scan found no new unexplained egress.

## Incremental source review (cbac4eab..971b2232, 2026-10-21)

### Feature changes

- STEP import and display planning: `step_in::spanning_one`, `curve_pieces`, `pin_end` and `snap_limit` keep awkward spline parameter ranges and small vertex-to-curve gaps from dropping imported edges, with caps on control points and snap distance. `Body::boundary_tol` preserves the source file's declared uncertainty through the native model cache. `tess::plan_face`, `split_bands`, periodic-surface evaluation and the new fallback planner recover faces that the coarse mesher misses, including closed spline bands, helical boundaries and sliver faces. Added tests cover gaps, seams, periodic surfaces and cached accuracy.
- Modeling and persistent references: `thread_on` and new `thread_turn` let a modeled external thread turn an oversized shaft down to its major diameter first, while refusing surrounding geometry that blocks the cut. `solid.align` and `FeatureKind::Align` now accept the origin and named or explicit planes; plane references resolve during evaluation so bodies follow moving planes. `Names::qualify_at` and `naming::along_spread` make split-face numbering stable across evaluations. `mass` measures about the body's centre to avoid placement-sensitive volume results. Sketch-face holes now target their host body, and extrusion to an edge or corner ends on the requested plane.
- Input and modeling safeguards: `transform_matrix_keeping` rejects perspective and singular transforms. `lex_spans` accepts a leading decimal comma such as `,5` where an operand is expected while retaining commas as separators in function arguments. Shell dialog selection now shows tangent face chains and explains when shelling removes the whole body.
- UI and control responsiveness: `view_key`/`step_key_hold` add steady turning while an arrow is held without repeated competing tweens. New `control_job::offload_control` evaluates eligible agent design changes on a copy while the window remains responsive; changes serialize, re-evaluate if the user edits meanwhile, and can be stopped before application. Exports run on a copy. The preference is off by default. `control.running` and the MCP `running` tool let an authorized client inspect active/queued request metadata across connections.
- CLI and test maintenance: the CLI's new `--debug`/`RUST_LOG` logger writes selected log messages to stderr. 3MF/DXF budget tests no longer depend on wall-clock timing. New regression tests cover the behaviors above; they were not run in this local review.

### Local navigation overlay

Upstream's `navigation.rs` change adds held-arrow turning; it does not add Shift+two-finger orbit. The existing local patch in `immediate_wheel` and `navigation::handle` still supplies that orbit gesture in mouse navigation presets. Pinch zoom and two-finger pan continue through the upstream input paths. The merge retains both the local gesture and upstream held-arrow changes; the local overlay adds no network behavior.

### Incremental security and data-egress findings

No new application HTTP(S) destination, telemetry client, automatic upload, subprocess launch, or external runtime dependency was added in this range. The CLI logger reads `RUST_LOG` only when initializing and writes to stderr; it does not transmit logs. Off-thread control work changes scheduling of existing commands and uses the request's existing folder scope for file operations.

`control.running` is a new read-only MCP/control method. It returns each active or queued request's connection and short request id, method, known command id or batch step count, elapsed time and queue status; it deliberately omits parameters and file paths. It is available to authenticated clients on the existing loopback listener and to the configured MCP agent when invoked. This is intentional local IPC, but it exposes limited activity metadata across clients; if the MCP result is included in a conversation with a configured hosted model, that metadata follows the existing user-directed AI conversation path. The existing CLI's explicitly configured non-loopback TCP use has the same transport limitations already described above.

The new `.github/workflows/kernel-paused.yml` is the only new remote communication path in this diff. On new issues, pull requests and discussions it uses pinned GitHub Actions to call GitHub's REST/GraphQL APIs, check contributor permission or changed PR paths, and create/apply the `kernel-paused` label. Its token permissions are limited to contents read, pull-request read, issue write and discussion write. The pull-request workflow uses trusted base-branch script content and does not check out or execute the fork's code. The calls send repository issue/PR/discussion metadata to GitHub; no CAD designs or local files are involved. This is expected repository-maintenance automation, not desktop application traffic.

The manifest change enables the existing workspace `log` package for `solidcraft-cli`; it adds no new package to the lockfile. No unexplained outbound behavior was found in the changed runtime paths or the local overlay.

## Incremental source review (971b2232..034aa60d, 2026-10-21)

### Feature changes

- Release 0.18.3 advances the workspace and local crate versions. The lockfile only changes those
  workspace package versions; no direct dependency was added or changed.
- Format migration notes move from `crates/doc/src/format/pending/` to `v29/`. The migration
  `face-pieces-by-place::step` marks existing features to retain their historical kernel-order
  face-piece names while newly created features use stable spatial ordering. The
  `align-to-planes` and `sheet-perforation-from-right` entries document new saved fields; older
  designs need no transformation because the absent fields retain their prior behavior.
- The local Browser patch changes disclosure defaults in `component_rows`, `occurrence_rows`,
  `folder_rows_at`, `analysis_rows`, `canvas_rows`, feature rows, pattern rows and the joints folder
  so the tree opens fully collapsed. `SolveApp::prefs` and `load_prefs` store the user's expanded
  and collapsed keys in the existing app preference JSON; a one-time state-version migration clears
  old open-by-default Browser overrides while retaining timeline group folds. This is user-level
  application preference state, not data written into each `.solidcraft` design.
- `Lit::of` now resolves selected body placement paths and highlights the body row when visible, or
  the deepest visible component/occurrence parent when a disclosure branch, folder or group hides
  it. `visible_parent_of_occurrence`, `body_row_visible` and `closest_component_row` implement that
  mapping. A viewport selection does not expand the tree. `features::reveal` and the Browser
  `reveal` flow explicitly open only the path needed for a user-requested Find in Browser action.
- Upstream issue [#3005](https://github.com/tangency-labs/solidcraft/issues/3005) tracks viewport
  selection not highlighting its Browser row. The open [PR #3081](https://github.com/tangency-labs/solidcraft/pull/3081)
  expands to the selected body and scrolls it into view; that differs from the requested local
  collapsed-ancestor highlight behavior, so the local behavior remains useful while the upstream
  PR is unmerged. No separate issue for remembering the collapsed tree state was found during the
  issue search.

### Incremental security and data-egress findings

The upstream source range changes only workspace version strings and three versioned format
migration files. The one migration function edits in-memory JSON fields; it does not access files,
network clients, processes or credentials. No new destination, HTTP(S) request, browser fetch,
socket, subprocess, telemetry, upload behavior or direct dependency was introduced.

The local Browser edits only read selection/document state, update the UI disclosure sets, and
serialize those sets through the existing local preference mechanism. They add no network, file
upload, subprocess, credential or telemetry path. The full overlay remains the pre-existing
trackpad customization plus this UI behavior and its workflow/report documentation. No unexplained
egress or concerning behavior was identified in the reviewed source or overlay diffs. This is a
static diff review and does not establish transitive-dependency or runtime safety.

## Incremental source review (034aa60d..2fdc105d, 2026-10-22)

### Feature and function changes

- Import and modeling: `iges_import`, `translate` and the connected-shell handling in
  `crates/kernel/src/iges_in.rs` now keep disconnected lumps of a single IGES part as separate
  solids, with importer regressions. `mirror` in `crates/engine/src/cmd/features.rs` now places
  mirrored copies in the source component and follows the saved placement. The fuzzer accepts
  chamfer material addition as it already did for fillets.
- Sketch edits: command handlers in `crates/engine/src/cmd/sketch.rs` improve midpoint and snapped
  center-circle selection; `trim_path`, `extend`, and offset-end handling in
  `crates/engine/src/cmd/sketch_modify.rs` reject invalid offset endpoints instead of silently
  undoing the operation. Regression cases were added for these paths.
- Camera and input: `navigate`/`continuous` in `crates/engine/src/cmd/view_navigation.rs` and
  camera methods in `crates/render/src/camera.rs` now apply view changes without the old camera
  animation. The 3D mouse reader in `apps/solidcraft/src/space_mouse.rs` adds per-axis source and
  speed mapping; `spacenavd.rs` and `hidraw.rs` read local daemon/device input, including
  nonblocking polling and near-rest filtering.
- Startup and desktop UI: `home::start_jobs`/`poll` move sample thumbnails and recent-file checks
  off the window thread; `fonts::load_system_fonts_in_background` avoids blocking the first frame
  on system CJK-font lookup; `window_fit::fit` handles restored Windows windows outside the
  current monitor work area. The release also restyles `timeline_left` and adds the tokenized
  history-strip widgets in `ui_kit/history.rs`, remembers the last sketch-tool variant, and
  improves parameter suggestion insertion, Tab navigation, and tapped-hole custom sizing.
- Issue reporting: `issue_report::Draft`, `draft_url`, and `show` replace the old report handoff
  with editable summary, reproduction, expected-result, and technical-context fields. The app
  opens a GitHub issue draft only on the user's **Continue on GitHub** click; short text is encoded
  in the URL, and a long body is copied to the local clipboard while the URL carries only the
  title. The user still reviews and submits the draft in GitHub.
- Release and CLI: workspace version advances to 0.18.4. CLI help now writes usage to stdout and
  exits successfully; the added Windows frame code handles monitor work areas. Dependency changes
  add a Windows GDI API feature to the existing `windows-sys` package; no new package was added.

### Local feature comparison

- Upstream now supplies standard two-finger pan and pinch zoom. It does not add the local
  Shift+two-finger orbit handler in `navigation.rs`/`viewport.rs`, so that trackpad overlay remains.
  It also removed the old camera-animation `orbit_snap` state; the local handler now uses the
  upstream immediate `view.navigate` path without referring to that removed field.
- Upstream still opens Browser branches by default and does not persist this fork's new
  fully-collapsed first-open preference or nearest-visible-component selection highlight. The
  local changes in `browser.rs`, `browser/features.rs`, `dialogs_assembly.rs`, `lib.rs`, and
  `context_menu.rs` remain necessary. The preference migration clears old Browser disclosure
  overrides once while preserving timeline group folds; subsequent tree disclosure state remains
  in application preferences. User-requested Find in Browser still opens the path it needs.
- Group creation now writes the scoped keys that `folder_rows_at` actually reads, so a new group
  opens in the selected folder or occurrence after the default-collapse change.

### Incremental security and data-egress findings

The source range changes 121 files and advances release/workspace versions, but adds no new
dependency package, telemetry service, automatic crash upload, or unexplained remote destination.
The changed 3D mouse support reads `/sys/class/hidraw`, `/dev/hidrawN`, and the local Unix socket
`/run/spnav.sock` or `/var/run/spnav.sock`; it sends no data to an Internet host. Startup thumbnail
and recent-file work is local. CI's pinned build/cache actions run on GitHub-hosted infrastructure,
not in the desktop app. The issue-report browser handoff is explicit and may disclose the user-edited
report title/body to GitHub when the draft URL is opened; for long reports, the body is copied to the
clipboard and stays local until the user pastes/submits it. Existing updater, configured AI/ACP,
browser asset, and optional MCP/control communication paths remain as documented above.

The added-line review found only test subprocess launches, local spacenavd socket setup, and the
user-triggered `ctx.open_url` report handoff among new communication/process sinks. No suspicious
credential collection or unexplained transfer was identified. This is static source review; it does
not audit every transitive dependency or prove runtime behavior.

## Incremental source review (2fdc105d..6c60f1a9, 2026-10-22)

Upstream advanced 11 commits to release 0.19.0. Across 182 changed files, the update contains
several user-visible behavior changes, a large UI-kit/layout refactor, 14 additional official theme
files and localization/generated updates. The behavior changes reviewed include:

- Navigation in `crates/ui-egui/src/navigation.rs` adds the opt-in `free_orbit` preference to
  `view.navigate` and shares the setting across mouse, trackpad and 3D-mouse camera paths.
  `view.normal_to` aligns the camera with a selected planar face or datum plane. The view-change
  glide is restored for view changes while orbit, pan and zoom remain immediate. In the underlying
  `immediate_wheel` input path, two-finger movement still pans and pinch zooms; upstream does not
  route Shift+two-finger movement to orbit.
- Chamfer validation in `crates/kernel/src/blend.rs::chamfer_tool_sides` now checks whether the
  far side of an expanded cutter reaches nearby material. It rejects chamfers that would bite a
  neighboring wall; upstream adds a regression case for a pin in a socket.
- Current-view DXF export now includes profile curves from shown sketches. The new
  `view_dxf_sketch.rs` functions `exportable`, `profile_curve`, `curve_parts` and `parts` collect
  visible sketch runs, while `view_dxf::export` adds the `SKETCH` layer and reports its entity
  count. Hidden curves use the hidden layer when dashed export is enabled; construction and
  reference geometry is excluded.
- `sketch.dimension` in `crates/engine/src/cmd/sketch.rs::dimension` accepts `name = expression`
  to define a user parameter and use it for the dimension. The expression definition parser and
  `Document::param_name_problem` checks reject malformed, reserved or duplicate names, and the
  dimension response reports the defined parameter.
- Empty-root cuts are made consistent between preview and execution through
  `Session::cut_takes_every_body` in `crates/engine/src/cmd/features.rs`.
- The workspace version advances to 0.19.0. The compact workspace layout and shared UI container
  components change app-shell and panel rendering; a new static `xtask ui-containers` checker and
  CI invocation enforce UI structure. Fourteen official theme palettes and related locale, docs
  and asset updates are mostly presentation/catalog changes rather than new runtime services.

### Local feature comparison

The local Shift+two-finger orbit remains necessary: upstream supports pan and pinch zoom but has no
Shift gesture routing in `immediate_wheel`. Upstream's new free-orbit preference is related camera
behavior and does not replace the requested Fusion 360 trackpad binding.

The local Browser changes also remain necessary. Upstream's `browser.rs` still calls `is_open(...,
true)` for component, occurrence, folder, group and pattern branches; its saved UI preferences do
not migrate the initial tree to fully collapsed. The fork continues to default the tree to
collapsed, remember disclosure choices across runs, highlight the selected component or occurrence,
and resolve a selection hidden by a collapsed branch to the nearest visible parent. The local group
creation scope adjustment remains paired with collapsed-by-default state. No redundant local
implementation was found in these reviewed areas.

### Incremental security and data-egress findings

The source range adds no general-purpose network client, telemetry service, automatic crash upload,
or unexplained remote destination. The manifest/lock changes add `proc-macro2` and `syn` support to
the `xtask` static UI-container checker; they do not add a runtime network dependency. The new
`xtask/src/web_offline_test.rs` TCP listener binds to `127.0.0.1` for its opt-in offline browser
harness, serves local build files and coordinates a local headless Chrome test. It is a development
tool, not a desktop-app listener or outbound transfer. The changed agent-chat UI connects only after
the user activates the control and uses the already configured API endpoint or installed ACP agent;
this is the existing optional integration, not a new destination.

The added-line scan found no unexpected credential reads, telemetry endpoints, or automatic data
uploads. Existing updater, configured AI/ACP, browser asset, issue-sharing and optional MCP/control
paths retain the purposes and destinations documented in the baseline above. No concerning or
possibly malicious change was identified, so the rebuild may proceed. This is a static source review;
it does not prove the safety of transitive dependencies or runtime behavior.

## Incremental source review (6c60f1a9..03018745, 2026-10-23)

Upstream advanced 19 commits (17 non-merge commits) to release 0.20.0. The range changes 172 files
(13,732 insertions and 2,864 deletions), with substantial File-view, Preferences, rendering and
performance work. The user-visible and functional changes reviewed include:

- The Start page becomes a File view. `home::show` and its action dispatch now provide navigation
  for new/open/recover/save/export, design properties, settings, shortcuts and help. Shared
  `ui_kit::FileView` components replace much of the bespoke start-page layout while retaining the
  open design and its history.
- Preferences are reorganized into General, Interface, Design, Navigation, Graphics, Sketch,
  Assistant and Plugins sections using new `ui_kit::preferences` components. The split pages retain
  controls for the existing navigation preset, two-finger pan/zoom and free-orbit options. They do
  not add Shift+two-finger orbit routing.
- `Document::next_item_id` and `Document::allocate_id`, with the `doc::format::read` migration,
  establish a single monotonic identity counter across features, browser/timeline groups, canvases,
  joints, contact sets, studies and exploded views. Browser, timeline and assembly command handlers
  now use it so deleting an auxiliary item does not allow its ID to be reused after saving/reopening.
- Construction axes can now be saved as references for circular patterns and revolve/surface
  revolve operations. `engine::cmd::features::axis_line` resolves the named axis geometry and
  records `axis_feature`; the v31 document migration preserves older construction-axis references.
  Editing an offset plane so that it depends on itself is rejected by `edit::redefine`, and sketch
  fillet/chamfer commands reject offset corners that would otherwise leave invalid circular edges.
- Joint driving avoids reevaluating unchanged model geometry; contact-sweep meshes are cached by
  geometry key. Imported mesh vertex/triangle arrays use shared `Arc` storage to avoid duplicating
  large scans when copying documents.
- Rendering now culls out-of-view placements of instanced meshes, shares their GPU triangle buffers,
  and selects display-only levels of detail using `geom::simplify::levels` and a pixel-error bound.
  The separate GPU picking buffer was removed in favor of per-body BVH picking. New
  `gpu::view::oit` passes improve order-independent transparency for glass/see-through surfaces.
  Large-mesh vertex welding is faster and starts before the first frame; face hover highlighting
  persists as the pointer moves over the face. Ribbon tab lists are cached rather than rebuilt on
  each frame.
- On Windows, `space_mouse::windows_hid::multi_axis_controllers` asks Plug and Play for HID usage
  metadata and opens only interfaces identified as 3D mice. This avoids repeatedly querying
  unrelated keyboards, mice and touchscreens for device strings. The changes in `space_mouse.rs`
  keep macOS discovery on hidapi and use the new targeted path on Windows.
- Release/workspace versions advance to 0.20.0. UI-kit labels, the redesigned preferences and File
  view receive catalog, icon and documentation updates; these are primarily presentation and
  localization changes.

### Local feature comparison

Upstream still routes two-finger trackpad movement to pan (or zoom when that preference is off) and
pinch to zoom. The local Shift+two-finger orbit handler is absent from upstream's
`navigation::immediate_wheel`, so it remains necessary. The local Browser tree still opens collapsed
by default, persists its disclosure state and highlights viewer selections at the nearest visible
ancestor; upstream's component, occurrence, folder, group and pattern rows still default to open.
No duplicate local behavior was found. The Preferences redesign keeps the existing trackpad-pan
setting and does not replace the local gesture or Browser changes.

### Incremental security and data-egress findings

No new network client, remote destination, telemetry integration, automatic upload or credential
path was found in the source diff. The changed manifests enable Windows Configuration Manager/HID
API features for the new device scan and add `proc-macro2` as a UI-kit development dependency; these
packages/APIs do not introduce a runtime network service. The added-line sink scan found no new
socket, HTTP client, browser fetch, process launch or URL-open path. Its matches were local process
IDs used in test fixtures, removal of the former Start-page issue/release hyperlinks, and GPU buffer
"uploads" (local device-memory transfers).

The Windows HID scan reads local Plug and Play interface properties, selects the Generic Desktop
Multi-Axis Controller usage, and opens only those candidate device paths. It sends no Internet
request and reads no unrelated HID device strings. The agent-chat change in `lib.rs` calls
`agent_chat::poll` while the File view is open so a detection task already started from the optional
Assistant panel/settings can finish and update its UI. It does not start a new scan there; the
existing discovery path can run a local configured agent's `--version` probe, and an agent session
starts only after the existing user action. The existing optional AI/API, updater, issue-sharing,
web-asset and MCP/control paths remain as documented above and their destinations are unchanged in
this range.

No concerning or possibly malicious change was identified. This is a scoped static review; it is not
a full audit of transitive dependency source or a runtime packet-capture assessment.

## Review method and limits

- Searched Rust, JavaScript, TypeScript, and manifests for HTTP clients, browser fetch, sockets,
  hard-coded URLs, telemetry/reporting services, child-process launches, and credential use.
- Traced the native update client, browser service worker/font fetch, AI API and ACP-agent paths,
  MCP/control listeners, issue-report flow, local crash storage, and 3D mouse integration to their
  callers and destinations.
- Compared the initial source update from `b168750f` to `fc3dedfd`, checked the changed dependency
  manifests/lockfile, and inspected the local trackpad/documentation overlay for communication
  changes.
- Compared `2fdc105d..6c60f1a9`: reviewed navigation/free-orbit and normal-to-selection, chamfer
  cutter validation, sketch-parameter definitions, empty-root cuts, sketch-aware DXF export, the
  compact UI/theme changes, changed dependencies and CI, plus the local navigation and Browser
  overlay. Traced the new loopback test harness and user-triggered agent-chat connection.
- Compared `6c60f1a9..03018745`: reviewed the File view and Preferences redesign, monotonic document
  IDs and v31 migration, construction-axis features, joint/contact evaluation, mesh sharing/LOD/
  culling/transparency/picking changes and Windows HID discovery. Checked changed manifests and
  dependencies, scanned added sink calls, traced local HID access and existing agent discovery, and
  rechecked that local trackpad and Browser behaviors remain absent upstream.
- Compared `9b801634..cbac4eab`: inspected `anchor_frame.rs`, `dialogs.rs`, their regression tests,
  and scanned changed source, manifests, and CI files for network, subprocess, credential, telemetry,
  or upload changes.
- Compared `cbac4eab..971b2232`: inspected the changed control server/MCP, off-thread job, logger,
  navigation, STEP/IGES import, tessellation, and GitHub Actions paths; checked added manifests and
  scanned added lines for network clients, destinations, subprocesses, credentials and telemetry.
- Compared `971b2232..034aa60d`: inspected all five changed paths, the three v29 migration entries
  and manifest/lockfile changes; searched the range for network clients, URLs, sockets, subprocesses,
  credentials, telemetry and uploads. Reviewed the local Browser selection and preference diff for
  external communication paths.
- Compared `034aa60d..2fdc105d`: reviewed the IGES import, sketch command, mirror, camera/navigation,
  3D mouse, startup/font, report-handoff and Windows window-fit changes; checked dependency diffs and
  searched added Rust/TOML/YAML lines for network clients, URL opens, sockets, subprocesses,
  credentials, telemetry and upload paths.
- Compared the local overlay to `971b2232` after merging and confirmed its only behavior patch is
  the Shift+two-finger orbit addition alongside upstream's held-arrow controls.
- This was not a full line-by-line audit of all vendored/transitive dependency source, a software
  composition vulnerability scan, a packet capture, or a sandboxed runtime test. No claim is made
  that such unreviewed code or the user's installed ACP agents are safe. A future review should
  compare only the source and overlay diffs since this baseline, while tracing any changed call site
  into the existing sinks listed above.

## Conclusion

No unexplained or apparently malicious outbound transfer was identified in the reviewed source
through `03018745` or in the local overlay. This remains a static, scoped review rather than a
complete audit of all dependencies or runtime behavior.
SolidCraft's ordinary CAD work is local, but AI use, issue sharing, updater checks/installs, web-app
delivery, and explicitly configured remote MCP/API endpoints cross the machine boundary as described
above. The largest data-sharing choice is the optional AI integration; the largest local trust
boundary is the selected ACP agent executable.
