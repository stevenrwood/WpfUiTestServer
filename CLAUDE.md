# WpfUiTestServer — guide for Claude

You are being asked to add UI self-testing to a **WPF app** (or to work on this library).
Read `README.md` for the full HTTP route reference; this file is the fast operating guide.

## What it is

A flag-gated, **in-process** HTTP server that lets an external script drive a WPF app's UI by
`x:Uid` (via `AutomationPeer`s) and read state back — so a change can be exercised end to end and
**self-verified** instead of always being handed to a human to click. App-agnostic, **zero
external dependencies**, **loopback-only**, and it opens a socket **only when the host explicitly
enables it** (a normal run never listens).

Target: **.NET Framework 4.6.2** WPF apps. No XAML of its own; it walks the visual tree of open
windows and acts through each element's `AutomationPeer`, so it respects `IsEnabled` and fires the
real handlers.

## Consume it (in a WPF app)

Reference the NuGet package:

```xml
<PackageReference Include="WpfUiTestServer" Version="1.0.0" />
```

Start it once the UI is realized (e.g. end of your startup), gated behind a flag/env var so it
never runs in a normal session:

```csharp
if (testServerEnabled)
    WpfUiTestServer.UiTestServer.Start(
        mainWindow,                       // window whose visual tree is addressed
        port,                             // <= 0 uses DefaultPort (8760)
        new MyStatusProvider(mainWindow), // optional IUiTestStatusProvider (null => /status empty)
        msg => MyLog.Write(msg));         // optional log sink (null => no-op)
```

- **Addressing** works off `x:Uid` (WPF surfaces it at runtime as `UIElement.Uid`). Localized apps
  using LocBaml already have a unique `x:Uid` on ~every interactive control — free addressing.
- **`IUiTestStatusProvider`** is the seam for your domain state (served at `GET /status`).
- **Dialogs**: to answer message boxes unattended, route your `MessageBox.Show` calls through a
  host-side broker the server can pre-arm (the ioSender host does this via an `AppDialogs`
  shim). See the README's dialog section.

## Drive it (from a test script / another agent)

Raw HTTP on `127.0.0.1:<port>` (JSON). The essentials — full list in `README.md`:

- `GET /ping`, `GET /idle`, `GET /tree`, `GET /uids` — discovery / settle
- `GET /state/{uid}`, `POST /invoke/{uid}`, `POST /set/{uid}?value=` — read / act
- `GET /screenshot[/{uid}]` — PNG of a window/element (so an agent can *see* the UI)
- `POST /selecttab/{viewType}`, `POST /key/{key}`, `POST /menu/{uid}` — navigation / input
- `POST /dialog/arm`, `GET /dialogs`, `POST /dialog` — message-box handling
- `GET /exceptions` — surfaced app exceptions (test mode survives throws instead of crashing)

## Security

Loopback only, no authentication — anything that can reach loopback can drive the app. **Enable
only in test/dev**, never in a shipped/normal run. Keep the enabling flag off by default.

## Build / pack (working on the library itself)

```sh
dotnet pack -c Release        # → bin/Release/WpfUiTestServer.<ver>.nupkg (+ .snupkg)
```

SDK-style project targeting `net462`; the WPF/UI-automation assemblies come from the 4.6.2
targeting pack (no `UseWPF` — there's no XAML). Publishing to NuGet happens via the
`.github/workflows/publish.yml` workflow on a `v*` tag (needs a `NUGET_API_KEY` repo secret).

## Gotchas

- **Only realized elements are addressable** — content on unselected tabs is created lazily; use
  `POST /selecttab` (or otherwise realize a view) before addressing its controls.
- **Duplicate `x:Uid`** across reused instances — `x:Uid` is unique per XAML file, not globally;
  addressing may need an index/scope for repeated controls.
- **`/idle` drains the Dispatcher only** — it doesn't know about in-flight async I/O (a running
  connect/job); poll state to confirm a settle.
