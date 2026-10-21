# Local Security and Data-Egress Baseline

**Reviewed:** 2026-10-21  
**Repository:** `/Users/phil/git/darque/solidcraft`  
**Branch:** `local/solidcraft-personalizations`  
**Upstream source commit:** `fc3dedfd5c0b2971a98e25df9edd0711d415587f` (`upstream/main`)  
**Previous source base:** `b168750fb3f4ecc68419525450406bcb54108ba6`  
**Reviewed source range:** `b168750f..fc3dedfd`  
**Local tracked worktree diff SHA-256 at baseline:** `67ac80fb3c67e6a13055cfc84db6e1b6f7578a7d02f86a738447395b50b12a69`

The local diff hash records the tracked working-tree changes after the security-review instructions
were added to `AGENTS.md` and before this report was created. The local changes at review time were
`AGENTS.md`, the trackpad navigation patch in `crates/ui-egui/src/navigation.rs` and
`crates/ui-egui/src/viewport.rs`, and related documentation. The trackpad patch does not add
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

## Source-update diff and dependencies

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

## Review method and limits

- Searched Rust, JavaScript, TypeScript, and manifests for HTTP clients, browser fetch, sockets,
  hard-coded URLs, telemetry/reporting services, child-process launches, and credential use.
- Traced the native update client, browser service worker/font fetch, AI API and ACP-agent paths,
  MCP/control listeners, issue-report flow, local crash storage, and 3D mouse integration to their
  callers and destinations.
- Compared the source update from `b168750f` to `fc3dedfd`, checked the changed dependency
  manifests/lockfile, and inspected the local trackpad/documentation overlay for communication
  changes.
- This was not a full line-by-line audit of all vendored/transitive dependency source, a software
  composition vulnerability scan, a packet capture, or a sandboxed runtime test. No claim is made
  that such unreviewed code or the user's installed ACP agents are safe. A future review should
  compare only the source and overlay diffs since this baseline, while tracing any changed call site
  into the existing sinks listed above.

## Conclusion

No unexplained or apparently malicious outbound transfer was identified in the reviewed source.
SolidCraft's ordinary CAD work is local, but AI use, issue sharing, updater checks/installs, web-app
delivery, and explicitly configured remote MCP/API endpoints cross the machine boundary as described
above. The largest data-sharing choice is the optional AI integration; the largest local trust
boundary is the selected ACP agent executable.
