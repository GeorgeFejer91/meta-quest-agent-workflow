---
name: meta-quest-workflow
description: Use when working with Meta Quest headsets, Quest/Android ADB workflows, development APK install/launch/validation, logcat, screenshots, Perfetto, Camera2 or headset camera metadata, broker-style localhost probes, OpenXR tracking boundaries, Meta Horizon MCP, or hzdb.
---

# Meta Quest Workflow

Use this skill before touching a Meta Quest headset, ADB transport, APK install
or launch loops, screenshots, logcat, Perfetto, camera metadata, broker probes,
or Quest-specific MCP tooling.

The workflow is portable and project-independent. Do not assume a particular
repo, package name, headset serial, local path, broker implementation, or MCP
client. Use placeholders such as `<serial>`, `<package>`, `<activity>`,
`<apk>`, and `<out-dir>` in reusable notes.

## First Decisions

1. Classify the request as planning-only, read-only inspection, bounded
   capture, app lifecycle, file mutation, device setting, shell helper,
   network forwarding, or destructive cleanup.
2. Prefer read-only probes before side effects.
3. Use an explicit serial when more than one Android device may be connected.
4. Preserve headset power, stay-awake, and proximity state unless the task
   explicitly requires a bounded change.
5. Record evidence for device-facing work: goal, provider, fallback, serial,
   model, package/activity or endpoint, foreground before/after, permissions,
   artifact paths, headset prompts, and remaining uncertainty.
6. For controller-only or gaze/controller-driven UI work, separate command
   behavior validation from real hardware binding proof.

## Provider Order

Use the narrowest provider that answers the question:

1. App-owned or broker-style HTTP/WebSocket status endpoint for health, clocks,
   streams, and command acknowledgements.
2. Meta Horizon MCP or `hzdb` for Quest-specific docs, device status, logs,
   screenshots, Perfetto, and asset/package search when configured.
3. ADB for install, launch, logcat, screenshots, dumpsys, file push/pull, and
   port forwarding.
4. App-private diagnostics, usually pulled through `run-as` on debuggable
   builds.
5. Manual headset action for runtime permission prompts, MediaProjection
   consent, protected prompts, and real controller input.

Do not substitute one source for another without labeling it. ADB screenshots,
MediaProjection captures, casts, headset camera frames, app-private camera
frames, and `hzdb` screenshots are different witnesses.

For controller-driven UI validation, prefer an app-owned command replay or
diagnostic endpoint before asking a person to repeat a full headset flow. Manual
controller action proves only the physical input boundary unless it is paired
with app-level command and state markers.

## Safe ADB Baseline

Start with explicit discovery:

```powershell
adb devices -l
adb -s <serial> shell getprop ro.product.model
adb -s <serial> shell getprop ro.build.version.release
adb -s <serial> shell wm size
adb -s <serial> shell wm density
adb -s <serial> shell dumpsys window | findstr /i "mCurrentFocus mFocusedApp"
```

Avoid broad repeated `dumpsys activity activities` polling in tight loops unless
the extra detail is necessary.

## APK Install, Grant, Launch

Use this sequence for development APK validation:

```powershell
adb -s <serial> install -r -d -g <path-to.apk>
adb -s <serial> shell pm grant <package> android.permission.CAMERA
adb -s <serial> shell pm grant <package> horizonos.permission.HEADSET_CAMERA
adb -s <serial> shell am start -n <package>/<activity>
```

Only grant permissions declared by the APK and required by the profile. Some
permissions are install-time, signature, special, or headset-UI gated; preserve
the failed grant output rather than pretending ADB can bypass that boundary.

For a bounded launch smoke run, prefer the bundled helper:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass `
  -File <skill-root>\scripts\install-launch-watch.ps1 `
  -Serial <serial> `
  -Apk <path-to.apk> `
  -Package <package> `
  -Activity <activity> `
  -OutDir <out-dir>
```

Read `references/apk-install-launch.md` when debugging launchers, cold starts,
runtime grants, native library loading, OpenXR lifecycle, or black screens.

## Logcat, Screenshots, And Readiness

For meaningful visual evidence, capture after an app-owned readiness marker,
not immediately after `am start`.

Useful marker classes:

```text
source-sampling contract
projection-coordinate contract
OpenXR frame submitted
nonzero XR cadence
visible camera projection ready
first camera frame timestamp/source metadata
```

Recommended sequence:

```text
clear or start a bounded logcat window
launch app
wait for readiness marker or fixed warmup
settle briefly
capture screenshot or sequence
save logcat window and readiness summary
```

Reject or bracket captures if logs show sleep, standby, display power-off,
OpenXR session exit, fatal exception, or a protected system prompt that was not
part of the test.

Read `references/quest-signal-patterns.md`,
`references/artifact-and-evidence-discipline.md`, and
`references/troubleshooting.md` for evidence quality checks and failure triage.

## Controller-Driven UI Validation

When validating a UI that must be navigable through controller commands alone,
build and test an app-level command contract. Treat real controller input and
replayed validation input as two sources feeding the same command queue or
navigation service.

Use project-specific command names when they already exist. Otherwise use a
small explicit command set such as:

```text
NavigateUp
NavigateDown
NavigateLeft
NavigateRight
Activate
Back
TriggerSelect
```

Each generated or runtime-created control that participates in controller
navigation should have a stable focus id. Good focus ids are semantic and
screen-scoped, for example `<screen>.submit`, `<screen>.choice.<id>`,
`<screen>.field.<id>`, or `<screen>.slider.<id>`.

Require runtime markers or status snapshots after each command. Capture at
least:

```text
command name
source: replay, controller, hand, keyboard, or synthetic Android input
active screen id
focused control id
selected value or text when relevant
submit/next enabled state
export or completion status when relevant
canvas pose, distance, scale, or legibility metrics for visual UI work
```

Use this validation hierarchy and do not skip a lower gate while claiming a
higher one passed:

1. Static contract checks: config, assets, screen definitions, focus ids,
   controller-map colors, and at least one reachable submit or next action per
   screen.
2. Pure navigation unit tests: graph movement, wrapping or clamping, disabled
   control skipping, slider/value stepping, repeat timing, delete/back
   behavior, and safe `Back` semantics without rendering.
3. Unity/EditMode or equivalent UI replay: instantiate real UI controls, force
   layout, replay commands, and assert every active selectable is reachable and
   non-overlapping.
4. PlayMode/runtime full-flow replay: run complete scripted sessions through the
   same command queue used by real controller input; verify state, saved data,
   end conditions, and command markers.
5. Quest device replay without human input: place an app-owned replay marker or
   call an app-owned command endpoint, launch the APK, replay the commands on
   device, pull outputs, and inspect the summary.
6. Quest visual and legibility gate: capture after app readiness with `hzdb` or
   ADB screenshots, and pair images with foreground/logcat evidence and app
   pose/legibility metrics.
7. Minimal human hardware gate: after input-binding, SDK, OpenXR, or controller
   mapping changes, ask the headset operator to press each physical control
   once while logcat or status markers confirm those controls reached the same
   command queue. Do not re-run the whole questionnaire or app flow manually
   unless the command replay gate is missing or failing.

ADB synthetic input can help test Android routing and app focus. It is not
accepted as proof that Meta Touch/OpenXR controller actions are bound correctly.
If synthetic input drives the command queue successfully but real controller
markers are absent, record the command UI as validated and leave only the
hardware boundary gate open.

## Camera Metadata

Start with pure ADB camera metadata:

```powershell
adb -s <serial> shell dumpsys media.camera > <out-dir>\dumpsys-media-camera.txt
adb -s <serial> shell cmd media.camera dump > <out-dir>\cmd-media-camera-dump.txt 2>&1
```

For repeatable collection, use:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass `
  -File <skill-root>\scripts\collect-camera-metadata.ps1 `
  -Serial <serial> `
  -OutDir <out-dir>
```

If an app or broker can run a Camera2 probe, collect app-context metadata too:
camera id, lens facing, physical/logical relationships, output sizes, FPS
ranges, active array, pixel array, crop region, sensor orientation, intrinsic
calibration, distortion when exposed, lens pose translation/rotation/reference,
and whether a short open/capture probe succeeded.

Treat model names, camera IDs, lens poses, and homographies as runtime metadata
or calibration, not portable constants.

Read `references/camera-metadata-collection.md` and
`references/capture-source-taxonomy.md` for camera/source distinctions.

- Native passthrough compositor: user-facing MR background controlled by the
  runtime. It is not an app-sampleable texture in normal public app code.
- Raw camera / Passthrough Camera API / Camera2: app-visible frames and camera
  metadata. Use this for custom CV and camera projection.
- Environment depth: runtime depth texture and metadata. It is not raw RGB
  camera and not final-display capture.
- MediaProjection: flattened display or app-window pixels after user consent.
  Use it to inspect what was displayed, not to obtain raw camera frames.
- ADB or hzdb screenshot: still image witness with its own timing and capture
  policy.
- Casting or screenrecord: operator inspection of the presented display, not
  raw camera data.
- Direct stream frame/status endpoints: app, broker, VNC, or MJPEG evidence
  pulled without relying on host foreground. This proves that the stream route
  produced content, not that the Quest panel presented it.

## Broker-Style Localhost Probes

A Quest app or sidecar broker can expose local status and command endpoints:

```text
http://127.0.0.1:<port>/status
http://127.0.0.1:<port>/clock/now
ws://127.0.0.1:<port>/<events-path>
```

Use ADB forwarding from the host:

```powershell
adb -s <serial> forward tcp:<host-port> tcp:<device-port>
curl.exe http://127.0.0.1:<host-port>/status
```

For repeatable status capture, use:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass `
  -File <skill-root>\scripts\broker-status-probe.ps1 `
  -Serial <serial> `
  -HostPort <host-port> `
  -DevicePort <device-port> `
  -OutDir <out-dir>
```

A broker can report metadata, clocks, launch state, and diagnostic probes. It
does not own another foreground XR app's OpenXR frame loop, layer submission,
raw camera texture import, or controller action spaces.

Read `references/broker-style-localhost-probes.md` when the task involves app
status endpoints, WebSocket event streams, ADB forwards, or broker evidence.

## Tracking And Shell Helper Boundaries

Keep fused HMD and controller tracking inside the active XR app's OpenXR
session. ADB, shell helpers, 2D brokers, and background services are useful for
launch, logs, status, and transport; they are not a supported public backdoor
to another app's fused tracking stream.

If another process needs tracking, add a thin adapter to the foreground XR app:

```text
OpenXR frame loop
  -> locate views and action spaces at the selected XrTime
  -> record validity/tracked flags and timestamps
  -> publish sanitized snapshots to an app-owned UDP/TCP/WebSocket route
```

ADB synthetic input can test Android input routing. It does not prove Meta
Touch/OpenXR controller action bindings.

For long workflows, use host-side watchdogs or ADB-launched shell helpers only
as developer tooling with explicit bounds and restore notes. If adapting the
template helper launcher, start with `-DryRun`:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass `
  -File <skill-root>\scripts\start-device-watchdog-template.ps1 `
  -HelperJar <helper.jar> `
  -MainClass <main.class.Name> `
  -Serial <serial> `
  -DryRun
```

Read `references/openxr-tracking-boundary.md`,
`references/shell-helper-boundary.md`, and
`references/long-running-watchdogs.md` before touching tracking, synthetic
input, proximity, stay-awake, or shell helpers.

## Meta Horizon MCP And hzdb

Treat Meta Horizon Debug Bridge (`hzdb`) as an optional Quest-specific provider
beside ADB.

Read-only setup checks:

```powershell
node --version
npx --version
npx -y @meta-quest/hzdb --version
npx -y @meta-quest/hzdb --help
```

The MCP config template is in `assets/mcp-config-example.json`. Choose one MCP
registration route per agent or IDE. Do not register multiple Meta Horizon MCP
servers for the same agent unless the tool explicitly supports that.

Use MCP/docs search to verify Quest-specific assumptions before editing
passthrough, camera, environment depth, input, performance capture, or Horizon
OS behavior. Gate device-changing MCP tools the same way as equivalent ADB
commands.

Read `references/meta-horizon-mcp-and-hzdb.md` before using `hzdb`, MCP
registration, `hzdb` screenshots, Perfetto helpers, app helpers, or proximity
commands.

## Termux And Linux Sidecars

Termux can be a useful Quest lab sidecar for CLI diagnostics, local dashboards,
headless command/status services, Termux:X11 panels, Proot tools, and bounded
localhost VNC evidence. Treat it as a normal Android app, not Android `shell`,
not HOME, not a kiosk policy engine, not an XR runtime authority, and not a
hidden watchdog.

Start with small visible tests:

```text
Termux CLI
Headless localhost command/status service
Termux:X11 with one small X11 client
Proot CLI
Proot GUI client only after X11 is visible and stoppable
localhost dashboard or VNC only through an explicit forward
```

For XR workflows that need Linux tools but not a visible desktop, prefer a
headless localhost command/status sidecar. Initial validation showed a
Termux-owned JSON service can keep answering allowlisted command requests while
another headset app is foregrounded and no X11 desktop is visible. Before
product use, add a local capability token or equivalent auth, argument schemas,
timeouts, cancellation, structured stdout/stderr/status, and audit records.

For VNC, bind to localhost when possible, use ADB forwarding, capture the
needed evidence, stop the server, remove the forward, and verify cleanup. If a
VNC server fails on Android shared-memory permissions, retry with an explicit
no-shared-memory mode and record the exact flags.

For live VNC or MJPEG observation, prefer direct pulls from stream endpoints
such as `/status.json` and `/frame.jpg` over screenshots of a visible browser
window. Browser and cast windows are useful human-visible witnesses, but window
capture depends on host foreground and should be secondary evidence.

Keep X-root visibility, Termux headset-panel visibility, and separate
viewer-panel visibility distinct. Native phone-like Termux:X11 geometry can
render in the Quest 2D panel. Exact landscape preferences can fix X-root/VNC
left-slice output, but Termux's Android activity can still stay constrained or
letterboxed. Do not rely on shell-level task resize for correction; it can
mismatch bounds and crop the desktop. For headset-visible full-desktop
observation, use a separate landscape viewer panel over a localhost-only
VNC/MJPEG bridge. Initial validation shows this route can present a full
1280x720 desktop stream in a Quest 2D panel.

Reduce large desktop or Droid-style setup ideas to data-first session recipes:
preflight, start, status, stop, cleanup, evidence, risk, and authority
boundary. Do not copy third-party setup source without checking license and
attribution obligations.

## Side Effects And Gates

Use these default gates:

| Operation class | Default handling |
| --- | --- |
| Read-only status, docs, health, package info | Allowed after identifying target. |
| Bounded capture, screenshots, logcat, Perfetto | Allowed with artifact path and run label. |
| App lifecycle: install, launch, force-stop, clear | Require explicit operator intent. |
| File read or pull | Require explicit source and destination. |
| File write, push, delete | Require explicit operator intent and dry run when practical. |
| Device settings: stay-awake, proximity, sensor-lock, testing modes | Require explicit intent, bounded duration, and restore notes. |
| Shell helper, network forward, root-like commands | Require explicit intent and audit trail. |

## Reference Guide

Load reference files selectively:

- `references/adb-basics.md`: ADB discovery, focus, logcat, screenshots, files,
  port forwarding, and synthetic input boundaries.
- `references/apk-install-launch.md`: Install, grants, launch, crash watch,
  native library mismatches, and OpenXR lifecycle.
- `references/artifact-and-evidence-discipline.md`: Run manifests, artifact
  labels, source-control hygiene, and public sharing.
- `references/broker-style-localhost-probes.md`: Broker endpoints, ADB
  forwarding, command routes, and status evidence.
- `references/camera-metadata-collection.md`: Camera2/headset camera metadata,
  broker-assisted probes, and calibration cautions.
- `references/capture-source-taxonomy.md`: Passthrough, raw camera,
  environment depth, MediaProjection, screenshots, casting, and screenrecord.
- `references/long-running-watchdogs.md`: Host and device watchdog patterns.
- `references/meta-horizon-mcp-and-hzdb.md`: `hzdb` CLI/MCP setup and provider
  selection.
- `references/openxr-tracking-boundary.md`: HMD/controller tracking ownership
  and app-owned adapters.
- `references/permissions-and-distribution-boundary.md`: Manifest/runtime
  grants, headset consent, distribution, and platform policy.
- `references/quest-signal-patterns.md`: Readiness, visual liveness, sleep,
  focus, and evidence rejection signals.
- `references/shell-helper-boundary.md`: ADB shell helper limits and safe
  app-process helper use.
- `references/troubleshooting.md`: Common Quest APK, camera, screenshot, and
  `hzdb` failures.
- `references/source-notice.md`: Source lineage and sanitization notes.
