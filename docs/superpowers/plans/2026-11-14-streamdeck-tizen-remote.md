# Stream Deck → Samsung Tizen TV Remote Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A C# Stream Deck plugin with three key actions — TV On, TV Off (both state-aware) and Remote Key — that control the user's Samsung Tizen TV directly over the LAN.

**Architecture:** A plain .NET class library (`StreamDeck2Tizen.Core`) holds every TV concern — status probing, the remote WebSocket channel, pairing, Wake-on-LAN, power policies and the power sequence — behind small injected interfaces, fully unit-tested with fakes. A thin plugin executable (`StreamDeck2Tizen.Plugin`) built on BarRaider's StreamDeck-Tools wires Stream Deck key events and settings to the core. A throwaway console spike runs first against the real TV to confirm the protocol and tune timings.

**Tech Stack:** .NET 8 (C# 12), StreamDeck-Tools 7.0.0 (NuGet), xUnit, Microsoft.Extensions.TimeProvider.Testing (`FakeTimeProvider`), System.Text.Json in Core (StreamDeck-Tools itself uses Newtonsoft — keep Newtonsoft out of Core), Fluent UI System Icons (MIT) for artwork, Elgato CLI (`@elgato/cli`) for validate/pack.

**Spec:** `docs/superpowers/specs/2026-11-14-streamdeck-tizen-remote-design.md`

## Global Constraints

- TV only, LAN only: no SmartThings API, no Samsung account, no cloud services, not even as a fallback.
- Windows is the primary platform; macOS must not be blocked (no Windows-only APIs in Core; no `System.Drawing`; no `TitleParameters.FontFamily`), but no effort is spent on a macOS build in v1.
- Target TVs: the user's own Samsung Tizen TV, 2018+ protocol — `wss://<ip>:8002/api/v2/channels/samsung.remote.control?name=<base64>&token=<token>`; status at `http://<ip>:8001/api/v2/`.
- Self-signed TV certificate is accepted **only** when the remote host equals the configured TV IP.
- Code style: Sandi Metz (POODR / 99 Bottles) — small single-responsibility objects, dependencies injected, decisions separated from I/O; self-explanatory names; no redundant comments; **no magic numbers** — every port, timeout, delay and byte count is a named constant (a short comment is allowed on a constant to explain a protocol value).
- All network calls are `async`, take a `CancellationToken`, and have a timeout from `TvTimings`. Nothing blocks the Stream Deck event loop.
- Timing constants (starting values, tuned after the spike): status probe timeout 1.5 s; WebSocket connect timeout 3 s; pairing prompt timeout 30 s; power confirmation interval 1 s; power confirmation window 8 s; power transition guard 10 s.
- Small features, one commit per task (or per green step where the task says so), conventional commit messages (`feat:`, `test:`, `chore:`, `docs:`), every commit builds and passes `dotnet test`.
- Icons follow the Windows 11 (Fluent) design language.

## Review Focus

1. **Empty or invalid IP/MAC in a key's settings** → the key shows the alert and logs which setting is wrong; nothing throws out of the action. (Task 3 `TvAddress` tests; Task 12 action guard.)
2. **Saved token later rejected** (TV reset or device removed in TV settings) → `PowerOutcome.NotPaired` / `TvNotPairedException`, alert + log "re-pair this TV". (Task 8, Task 10 tests.)
3. **TV On then TV Off pressed within seconds on the same TV** → the second press is ignored; the guard is keyed by TV IP, not by action. (Task 9 and Task 10 tests.)
4. **Status endpoint answers oddly** (HTTP 5xx, empty body, non-JSON) → the TV is awake: `AnswersWithoutPowerState`; only timeout / connection refused means `NoAnswer`. (Task 4 tests.)
5. **MAC missing when Wake-on-LAN is needed** → `PowerOutcome.WakeOnLanUnavailable`, alert + log "set the TV's MAC address". MAC accepted as `AA:BB:CC:DD:EE:FF`, `aa-bb-cc-dd-ee-ff`, `AABBCCDDEEFF`. (Task 3, Task 10 tests.)

---

## File Structure

```
StreamDeck2Tizen.sln
Directory.Build.props                      net8.0, Nullable, ImplicitUsings, TreatWarningsAsErrors
.github/workflows/build.yml                windows-latest: restore, build, test
.gitignore, README.md, THIRD-PARTY-NOTICES.md
spike/TizenSpike/                          throwaway console (Task 1), not in the .sln
src/StreamDeck2Tizen.Core/
  TvPorts.cs, TvTimings.cs, RemoteKeys.cs
  Tv/TvAddress.cs, Tv/TvPowerState.cs
  Status/DeviceInfoReading.cs, Status/DeviceInfoParser.cs, Status/IDeviceInfoSignal.cs, Status/DeviceInfoSignal.cs
  Status/IScreenOnSignal.cs, Status/AlwaysOnScreenSignal.cs, Status/MediaRendererSignal.cs, Status/MediaRendererDiscovery.cs
  Status/TvStatusProbe.cs
  WakeOnLan/MagicPacket.cs, WakeOnLan/IDatagramSender.cs, WakeOnLan/UdpBroadcastSender.cs, WakeOnLan/WakeOnLanSender.cs
  Remote/RemoteChannelUri.cs, Remote/RemoteKeyMessage.cs, Remote/ChannelEvent.cs, Remote/ChannelEventParser.cs
  Remote/IRemoteChannel.cs, Remote/ClientWebSocketChannel.cs, Remote/TvRemote.cs, Remote/TvPairing.cs
  Remote/PairingResult.cs, Remote/TvNotPairedException.cs, Remote/TvUnreachableException.cs, Remote/TokenBook.cs
  Power/PowerCommand.cs, Power/IPowerPolicy.cs, Power/PowerOnPolicy.cs, Power/PowerOffPolicy.cs
  Power/PowerTransitionGuard.cs, Power/PowerOutcome.cs, Power/PowerSequence.cs
tests/StreamDeck2Tizen.Core.Tests/         mirrors Core folders; Fakes/ holds shared fakes
src/StreamDeck2Tizen.Plugin/
  Program.cs, PluginServices.cs, TvKeySettings.cs, GlobalSettingsTokenStore.cs, KeyFeedback.cs
  Actions/RemoteKeyAction.cs, Actions/TvOnAction.cs, Actions/TvOffAction.cs
  com.brianlima.streamdeck2tizen.sdPlugin/  manifest.json, PropertyInspector/*.html|js|css, Images/*.svg
```

Namespaces follow folders: `StreamDeck2Tizen.Core.Status`, etc.

---

### Task 1: Feasibility spike (throwaway, run by the user)

Spike rules: no tests, no review gate beyond "does it run", lives in `spike/`, deleted after Task 14. Not referenced by the solution.

**Files:**
- Create: `spike/TizenSpike/TizenSpike.csproj` (net8.0 console, no packages), `spike/TizenSpike/Program.cs`, `spike/README.md`

- [ ] **Step 1: Write a console app with these subcommands** (`dotnet run -- <cmd>`):
  - `info <ip>` — GET `http://<ip>:8001/api/v2/`, print raw JSON.
  - `pair <ip>` — open `wss://<ip>:8002/api/v2/channels/samsung.remote.control?name=<base64("StreamDeck2Tizen")>` (accept the TV's self-signed cert), print every message for 30 s, print the token from `ms.channel.connect`.
  - `key <ip> <token> <KEY_…>` — connect with token, send one `ms.remote.control` Click, print messages, close after 0.2 s.
  - `watch <ip> [rendererUrl]` — every 1 s print `HH:mm:ss  api=<PowerState|no-field|NO ANSWER>  renderer=<up|down|->`.
  - `ssdp` — send an SSDP M-SEARCH for `urn:schemas-upnp-org:device:MediaRenderer:1`, print each responder's IP and `LOCATION` for 3 s.
  - `wol <mac>` — broadcast the magic packet to 255.255.255.255:9.
- [ ] **Step 2: Build check** — `dotnet build spike/TizenSpike` → `Build succeeded`.
- [ ] **Step 3: Write `spike/README.md`** — the test script for the user: pair → key KEY_MUTE twice (token reused, no prompt) → `ssdp` → `watch` while turning the TV off with the physical remote; keep watching 10+ min → `wol` from fully off. Ask the user to paste outputs and the TV model.
- [ ] **Step 4: Commit** — `chore(spike): add TV protocol spike console`
- [ ] **Step 5: Hand to user, record findings** in `docs/spike-findings.md` (TV model, protocol, PowerState behaviour over time, renderer behaviour, WoL result). Then decide and note in that file: (a) is Signal B needed (Task 5)? (b) adjusted timing values for `TvTimings`. **If pairing failed: stop and re-plan.** Tasks 2–4 and 6–9 do not depend on the outcome and may proceed while waiting.

---

### Task 2: Solution scaffold + CI

**Files:**
- Create: `StreamDeck2Tizen.sln`, `Directory.Build.props`, `.gitignore` (dotnet template + `*.streamDeckPlugin`), `.github/workflows/build.yml`, `README.md` (one paragraph: what it is, LAN-only, link to spec), `src/StreamDeck2Tizen.Core/StreamDeck2Tizen.Core.csproj`, `tests/StreamDeck2Tizen.Core.Tests/StreamDeck2Tizen.Core.Tests.csproj` (xUnit, `Microsoft.Extensions.TimeProvider.Testing`, references Core), `src/StreamDeck2Tizen.Core/TvPorts.cs`, `src/StreamDeck2Tizen.Core/TvTimings.cs`, `tests/.../TvTimingsTests.cs`

**Interfaces:**
- Produces:
  - `static class TvPorts { const int DeviceInfoHttp = 8001; const int SecureRemote = 8002; const int WakeOnLan = 9; }`
  - `static class TvTimings` — `static readonly TimeSpan StatusProbeTimeout, WebSocketConnectTimeout, PairingPromptTimeout, PowerConfirmationInterval, PowerConfirmationWindow, PowerTransitionGuard, KeyDeliveryGrace` with the Global Constraints values; `KeyDeliveryGrace = 0.2 s` (wait before closing the socket so the TV processes the key — Claude's call, tune from spike).

- [ ] **Step 1: Failing test** `TvTimingsTests.Guard_outlasts_confirmation_window` — `Assert.True(TvTimings.PowerTransitionGuard > TvTimings.PowerConfirmationWindow)`; and `Probe_timeout_is_shorter_than_confirmation_interval_times_two` — `Assert.True(TvTimings.StatusProbeTimeout < TvTimings.PowerConfirmationInterval * 2)`.
- [ ] **Step 2: Run** `dotnet test` → FAIL (types missing).
- [ ] **Step 3: Implement** the files above. `build.yml`: on push/PR, `windows-latest`, `actions/setup-dotnet@v4` with `8.0.x`, `dotnet test --configuration Release`.
- [ ] **Step 4: Run** `dotnet test` → PASS.
- [ ] **Step 5: Commit** — `chore: scaffold solution, core library, tests and CI`

---

### Task 3: `TvAddress` and `TvPowerState`

**Files:**
- Create: `src/StreamDeck2Tizen.Core/Tv/TvAddress.cs`, `Tv/TvPowerState.cs`
- Test: `tests/StreamDeck2Tizen.Core.Tests/Tv/TvAddressTests.cs`

**Interfaces:**
- Produces:
  - `enum TvPowerState { On, Standby, Unreachable }`
  - `sealed record TvAddress(IPAddress Ip, PhysicalAddress? Mac, string DisplayName)`
  - `static TvAddress TvAddress.Parse(string? ip, string? mac, string? displayName)` — throws `ArgumentException` whose `ParamName` is `"ip"` or `"mac"`. Empty/whitespace MAC → `Mac = null`. Empty display name → `DisplayName = ip`.

- [ ] **Step 1: Failing tests**
  - `Parses_ipv4_and_colon_mac` — `Parse("192.168.1.20","AA:BB:CC:DD:EE:FF","Living room")` → `Ip.ToString()=="192.168.1.20"`, `Mac!.ToString()=="AABBCCDDEEFF"`.
  - `[Theory] Accepts_mac_formats("aa-bb-cc-dd-ee-ff")("AABBCCDDEEFF")("aa:bb:cc:dd:ee:ff")` → `Mac!.ToString()=="AABBCCDDEEFF"`.
  - `Blank_mac_is_allowed` — `Parse("10.0.0.5","  ",null).Mac` is null; `DisplayName=="10.0.0.5"`.
  - `[Theory] Rejects_bad_ip(null)("")("tv.local")("300.1.1.1")` → `ArgumentException` with `ParamName=="ip"`.
  - `[Theory] Rejects_bad_mac("AA:BB:CC")("ZZ:BB:CC:DD:EE:FF")` → `ParamName=="mac"`.
- [ ] **Step 2: Run** `dotnet test --filter TvAddressTests` → FAIL.
- [ ] **Step 3: Implement.** Use `IPAddress.TryParse` but require IPv4 with exactly four dotted parts (rejects `"10"`); normalise MAC by stripping `:`/`-`, uppercase, then `PhysicalAddress.Parse`, require 6 bytes.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(core): add TvAddress value object`

---

### Task 4: Device info signal (Signal A)

**Files:**
- Create: `Status/DeviceInfoReading.cs`, `Status/DeviceInfoParser.cs`, `Status/IDeviceInfoSignal.cs`, `Status/DeviceInfoSignal.cs`
- Test: `tests/.../Status/DeviceInfoParserTests.cs`, `Status/DeviceInfoSignalTests.cs`, `Fakes/StubHttpHandler.cs`

**Interfaces:**
- Consumes: `TvPorts.DeviceInfoHttp`, `TvTimings.StatusProbeTimeout`
- Produces:
  - `enum DeviceInfoReading { NoAnswer, ReportsOn, ReportsStandby, AnswersWithoutPowerState }`
  - `static DeviceInfoReading DeviceInfoParser.Parse(string body)` — reads `device.PowerState` case-insensitively.
  - `interface IDeviceInfoSignal { Task<DeviceInfoReading> ReadAsync(IPAddress tv, CancellationToken ct); }`
  - `sealed class DeviceInfoSignal(HttpClient http) : IDeviceInfoSignal`
  - `Fakes/StubHttpHandler` — `HttpMessageHandler` returning a configured status/body or throwing a configured exception; records request URIs.

- [ ] **Step 1: Failing tests**
  - Parser: `{"device":{"PowerState":"on"}}` → `ReportsOn`; `"standby"` → `ReportsStandby`; `{"device":{}}` → `AnswersWithoutPowerState`; `""` → `AnswersWithoutPowerState`; `"<html>"` → `AnswersWithoutPowerState`; `"ON"` → `ReportsOn`.
  - Signal: requests exactly `http://192.168.1.20:8001/api/v2/`; HTTP 200 + on-JSON → `ReportsOn`; HTTP 500 → `AnswersWithoutPowerState`; `HttpRequestException` → `NoAnswer`; `TaskCanceledException` from the handler (timeout) → `NoAnswer`; caller-cancelled token → throws `OperationCanceledException` (not swallowed).
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement.** Per-request timeout via linked `CancellationTokenSource` with `TvTimings.StatusProbeTimeout`; distinguish caller cancellation from timeout by checking `ct.IsCancellationRequested`.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(core): read TV power state from device info endpoint`

---

### Task 5: Screen-on signal (Signal B) and renderer discovery

**Skip condition:** if `docs/spike-findings.md` says PowerState alone is reliable, implement only `IScreenOnSignal` + `AlwaysOnScreenSignal` (Step 1 first bullet) and skip the rest; note it in the commit message.

**Files:**
- Create: `Status/IScreenOnSignal.cs`, `Status/AlwaysOnScreenSignal.cs`, `Status/MediaRendererSignal.cs`, `Status/MediaRendererDiscovery.cs`
- Test: `tests/.../Status/MediaRendererSignalTests.cs`, `Status/MediaRendererDiscoveryTests.cs`

**Interfaces:**
- Consumes: `StubHttpHandler`, `IDatagramSender` is **not** reused here (SSDP needs receive).
- Produces:
  - `interface IScreenOnSignal { Task<bool> IsScreenOnAsync(CancellationToken ct); }`
  - `sealed class AlwaysOnScreenSignal : IScreenOnSignal` — always `true` (used when no renderer URL is configured, so Signal A alone decides).
  - `sealed class MediaRendererSignal(HttpClient http, Uri descriptionUrl) : IScreenOnSignal` — `true` on any HTTP 2xx within `StatusProbeTimeout`, else `false`.
  - `static Uri? MediaRendererDiscovery.PickLocation(IEnumerable<(IPAddress From, string Response)> responses, IPAddress tv)` (pure) and `static Task<Uri?> MediaRendererDiscovery.DiscoverAsync(IPAddress tv, CancellationToken ct)` (I/O: M-SEARCH to `239.255.255.250:1900`, ST `urn:schemas-upnp-org:device:MediaRenderer:1`, MX 2, listen 3 s — named constants).

- [ ] **Step 1: Failing tests**
  - `AlwaysOnScreenSignal` returns `true`.
  - `MediaRendererSignal`: 200 → `true`; 404 → `false`; `HttpRequestException` → `false`; timeout → `false`.
  - `PickLocation`: picks the `LOCATION:` header (case-insensitive) from the response whose `From` equals the TV IP; ignores other IPs; returns `null` if none.
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement.** `DiscoverAsync` is untested I/O; keep it to socket send/receive and delegate parsing to `PickLocation`.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(core): add screen-on signal via UPnP media renderer`

---

### Task 6: `TvStatusProbe`

**Files:**
- Create: `Status/TvStatusProbe.cs`
- Test: `tests/.../Status/TvStatusProbeTests.cs`, `Fakes/FakeDeviceInfoSignal.cs`, `Fakes/FakeScreenOnSignal.cs`

**Interfaces:**
- Consumes: `IDeviceInfoSignal`, `IScreenOnSignal`, `TvPowerState`
- Produces: `interface ITvStatusProbe { Task<TvPowerState> ProbeAsync(IPAddress tv, CancellationToken ct); }` and `sealed class TvStatusProbe(IDeviceInfoSignal deviceInfo, IScreenOnSignal screenOn) : ITvStatusProbe`. Fakes expose a settable reading/bool and a call counter.

- [ ] **Step 1: Failing tests** (`[Theory]` over the rule table from spec §7.1):

| deviceInfo | screenOn | expected |
|---|---|---|
| NoAnswer | any | Unreachable (screen signal **not called**) |
| ReportsStandby | any | Standby (screen signal **not called**) |
| ReportsOn | true | On |
| ReportsOn | false | Standby |
| AnswersWithoutPowerState | true | On |
| AnswersWithoutPowerState | false | Standby |

- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(core): combine status signals into TvStatusProbe`

---

### Task 7: Wake-on-LAN

**Files:**
- Create: `WakeOnLan/MagicPacket.cs`, `WakeOnLan/IDatagramSender.cs`, `WakeOnLan/UdpBroadcastSender.cs`, `WakeOnLan/WakeOnLanSender.cs`
- Test: `tests/.../WakeOnLan/MagicPacketTests.cs`, `WakeOnLan/WakeOnLanSenderTests.cs`, `Fakes/RecordingDatagramSender.cs`

**Interfaces:**
- Produces:
  - `static byte[] MagicPacket.Build(PhysicalAddress mac)` — named constants `SyncStreamLength = 6`, `MacRepetitions = 16`, `SyncByte = 0xFF`.
  - `interface IDatagramSender { Task SendAsync(byte[] payload, IPEndPoint target, CancellationToken ct); }`
  - `sealed class UdpBroadcastSender : IDatagramSender` (UdpClient with `EnableBroadcast = true`; untested I/O).
  - `interface IWakeOnLanSender { Task WakeAsync(PhysicalAddress mac, CancellationToken ct); }`
  - `sealed class WakeOnLanSender(IDatagramSender sender) : IWakeOnLanSender` — sends to `255.255.255.255:TvPorts.WakeOnLan`.

- [ ] **Step 1: Failing tests**
  - Packet length is 102; first 6 bytes are `0xFF`; bytes `[6..12)` and `[96..102)` equal the MAC bytes.
  - `WakeAsync` sends exactly one datagram to `255.255.255.255:9` whose payload equals `MagicPacket.Build(mac)`.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(core): add Wake-on-LAN magic packet sender`

---

### Task 8: Remote channel protocol, `TvRemote`, `TvPairing`, `TokenBook`

**Files:**
- Create: `RemoteKeys.cs`, `Remote/RemoteChannelUri.cs`, `Remote/RemoteKeyMessage.cs`, `Remote/ChannelEvent.cs`, `Remote/ChannelEventParser.cs`, `Remote/IRemoteChannel.cs`, `Remote/ClientWebSocketChannel.cs`, `Remote/TvRemote.cs`, `Remote/TvPairing.cs`, `Remote/PairingResult.cs`, `Remote/TvNotPairedException.cs`, `Remote/TvUnreachableException.cs`, `Remote/TokenBook.cs`
- Test: `tests/.../Remote/RemoteChannelUriTests.cs`, `RemoteKeyMessageTests.cs`, `ChannelEventParserTests.cs`, `TvRemoteTests.cs`, `TvPairingTests.cs`, `TokenBookTests.cs`, `Fakes/ScriptedRemoteChannel.cs`

**Interfaces:**
- Consumes: `TvPorts.SecureRemote`, `TvTimings.WebSocketConnectTimeout`, `TvTimings.PairingPromptTimeout`, `TvTimings.KeyDeliveryGrace`
- Produces:
  - `static class RemoteKeys` — `const string Power = "KEY_POWER"`; `IReadOnlyList<(string Code, string Label)> Catalog` with: VolumeUp `KEY_VOLUP`, VolumeDown `KEY_VOLDOWN`, Mute `KEY_MUTE`, Power `KEY_POWER`, Home `KEY_HOME`, Back `KEY_RETURN`, Source `KEY_SOURCE`, Menu `KEY_MENU`, Up `KEY_UP`, Down `KEY_DOWN`, Left `KEY_LEFT`, Right `KEY_RIGHT`, Enter `KEY_ENTER`, Play `KEY_PLAY`, Pause `KEY_PAUSE`, Stop `KEY_STOP`, FastForward `KEY_FF`, Rewind `KEY_REWIND`, ChannelUp `KEY_CHUP`, ChannelDown `KEY_CHDOWN`, HDMI 1–4 `KEY_HDMI1`…`KEY_HDMI4`.
  - `static Uri RemoteChannelUri.Build(IPAddress tv, string clientName, string? token)` — clientName is base64-encoded (UTF-8); `&token=` appended only when token is non-empty.
  - `static string RemoteKeyMessage.Build(string keyCode)` — `{"method":"ms.remote.control","params":{"Cmd":"Click","DataOfCmd":<key>,"Option":"false","TypeOfRemote":"SendRemoteKey"}}`.
  - `abstract record ChannelEvent`; `sealed record ChannelConnected(string? Token)`, `ChannelUnauthorized`, `ChannelTimedOut`, `ChannelOther(string EventName)`.
  - `static ChannelEvent ChannelEventParser.Parse(string json)` — events `ms.channel.connect` (token from `data.token`, string **or** number), `ms.channel.unauthorized`, `ms.channel.timeOut`; anything else / malformed → `ChannelOther`.
  - `interface IRemoteChannel : IAsyncDisposable { Task ConnectAsync(Uri uri, CancellationToken ct); Task SendAsync(string text, CancellationToken ct); Task<string> ReceiveAsync(CancellationToken ct); }`
  - `sealed class ClientWebSocketChannel(IPAddress trustedTv) : IRemoteChannel` — `ClientWebSocket` with `RemoteCertificateValidationCallback` that returns `true` only when the request host equals `trustedTv` (otherwise default validation). Untested I/O.
  - `const string ClientName = "StreamDeck2Tizen"` on `TvRemote`.
  - `interface ITvRemote { Task SendKeyAsync(IPAddress tv, string? token, string keyCode, CancellationToken ct); }`
  - `sealed class TvRemote(Func<IPAddress, IRemoteChannel> openChannel, TimeProvider time) : ITvRemote` — connect; read events until `ChannelConnected` (send key, wait `KeyDeliveryGrace`, return), `ChannelUnauthorized` → `TvNotPairedException`, connect failure or no event within `WebSocketConnectTimeout` → `TvUnreachableException`.
  - `abstract record PairingResult`; `Paired(string Token)`, `Denied`, `NoPromptAnswered`, `PairingTvUnreachable`.
  - `sealed class TvPairing(Func<IPAddress, IRemoteChannel> openChannel, TimeProvider time)` with `Task<PairingResult> PairAsync(IPAddress tv, CancellationToken ct)` — connect without token, wait up to `PairingPromptTimeout`; `ChannelConnected(token)` → `Paired(token ?? "")`.
  - `sealed class TokenBook` — `string? Find(IPAddress tv)`, `TokenBook With(IPAddress tv, string token)` (immutable), `IReadOnlyDictionary<string,string> ToDictionary()`, `static TokenBook From(IReadOnlyDictionary<string,string>? entries)`.
  - `Fakes/ScriptedRemoteChannel` — replays a queue of incoming messages, records sent text and the connect URI, can be told to throw on connect.

- [ ] **Step 1: Failing tests (pure parts)**
  - Uri: `Build(192.168.1.20, "StreamDeck2Tizen", "1234")` == `wss://192.168.1.20:8002/api/v2/channels/samsung.remote.control?name=U3RyZWFtRGVjazJUaXplbg==&token=1234`; with `null` token → no `&token`.
  - Message: parse `Build("KEY_MUTE")` with `JsonDocument` and assert each field value.
  - Parser: connect with `"token":"5555"` → `ChannelConnected("5555")`; `"token":5555` → `ChannelConnected("5555")`; connect without token → `ChannelConnected(null)`; unauthorized; timeOut; `"ms.channel.ready"` → `ChannelOther("ms.channel.ready")`; `"{"` → `ChannelOther`.
  - TokenBook: `With` then `Find` round-trips; `From(null).Find(ip)` is null; `With` does not mutate the original.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement pure parts.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(core): add Samsung remote channel protocol messages`
- [ ] **Step 6: Failing tests (TvRemote / TvPairing with `ScriptedRemoteChannel`, `FakeTimeProvider`)**
  - `SendKeyAsync`: script `[connect-with-token]` → sent text equals `RemoteKeyMessage.Build("KEY_MUTE")`, connect URI carries the token.
  - Script `[ms.channel.ready, connect]` → still sends the key (ignores `ChannelOther`).
  - Script `[unauthorized]` → throws `TvNotPairedException`.
  - Channel throws on connect → `TvUnreachableException`.
  - Pairing: `[connect "777"]` → `Paired("777")`; `[unauthorized]` → `Denied`; `[timeOut]` → `NoPromptAnswered`; connect throws → `PairingTvUnreachable`.
- [ ] **Step 7: Run** → FAIL. **Step 8: Implement.** **Step 9: Run** → PASS.
- [ ] **Step 10: Commit** — `feat(core): add TvRemote and TvPairing over the remote channel`

---

### Task 9: Power policies and transition guard

**Files:**
- Create: `Power/PowerCommand.cs`, `Power/IPowerPolicy.cs`, `Power/PowerOnPolicy.cs`, `Power/PowerOffPolicy.cs`, `Power/PowerTransitionGuard.cs`
- Test: `tests/.../Power/PowerOnPolicyTests.cs`, `PowerOffPolicyTests.cs`, `PowerTransitionGuardTests.cs`

**Interfaces:**
- Produces:
  - `enum PowerCommand { DoNothing, SendPowerKey, SendWakeOnLan }`
  - `interface IPowerPolicy { PowerCommand Decide(TvPowerState state); bool IsReached(TvPowerState state); PowerCommand FallbackAfter(PowerCommand failed); }`
  - `PowerOnPolicy`: Decide On→DoNothing, Standby→SendPowerKey, Unreachable→SendWakeOnLan; IsReached only On; FallbackAfter SendPowerKey→SendWakeOnLan, else DoNothing.
  - `PowerOffPolicy`: Decide On→SendPowerKey, else DoNothing; IsReached Standby or Unreachable; FallbackAfter always DoNothing.
  - `sealed class PowerTransitionGuard(TimeProvider time)` — `bool TryEnter(IPAddress tv)`, `void Release(IPAddress tv)`; an entry expires after `TvTimings.PowerTransitionGuard`; thread-safe.

- [ ] **Step 1: Failing tests** — every `Decide`/`IsReached`/`FallbackAfter` case above as `[Theory]` rows; guard: `TryEnter` true then false for the same IP; true for a different IP; true again after `Release`; true again after advancing `FakeTimeProvider` by 10 s; still false at 9.9 s.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(core): add power policies and transition guard`

---

### Task 10: `PowerSequence`

**Files:**
- Create: `Power/PowerOutcome.cs`, `Power/PowerSequence.cs`
- Test: `tests/.../Power/PowerSequenceTests.cs`, `Fakes/ScriptedStatusProbe.cs`, `Fakes/FakeTvRemote.cs`, `Fakes/FakeWakeOnLan.cs`

**Interfaces:**
- Consumes: `ITvStatusProbe` (Task 6), `ITvRemote` (Task 8), `IWakeOnLanSender` (Task 7), `PowerTransitionGuard`, `IPowerPolicy`, `TimeProvider`, `TvTimings.PowerConfirmationInterval`, `TvTimings.PowerConfirmationWindow`, `RemoteKeys.Power`
- Produces:
  - `enum PowerOutcome { AlreadyThere, Reached, IgnoredWhileTransitioning, TimedOut, NotPaired, WakeOnLanUnavailable }`
  - `sealed class PowerSequence(ITvStatusProbe probe, ITvRemote remote, IWakeOnLanSender wakeOnLan, PowerTransitionGuard guard, TimeProvider time)` with `Task<PowerOutcome> RunAsync(TvAddress tv, string? token, IPowerPolicy policy, CancellationToken ct)`

Algorithm (spec §7.2/7.3; TV Off also waits for confirmation so the guard releases on the real state change — Claude's call, OK shows once the TV is off):
```
if !guard.TryEnter(tv.Ip): return IgnoredWhileTransitioning
try:
  state = probe; command = policy.Decide(state)
  if command == DoNothing: return AlreadyThere
  for attempt in [command, policy.FallbackAfter(command)] while attempt != DoNothing:
    execute(attempt)            # SendPowerKey: remote.SendKeyAsync(RemoteKeys.Power)
                                #   TvUnreachableException → skip to next attempt (fallback)
                                #   TvNotPairedException  → return NotPaired
                                # SendWakeOnLan: tv.Mac null → return WakeOnLanUnavailable
    if await ConfirmAsync(policy): return Reached   # poll every interval until IsReached or window elapses
  return TimedOut
finally: guard.Release(tv.Ip)
```

- [ ] **Step 1: Failing tests** (`FakeTimeProvider`; drive time with `Advance` from the test while awaiting the task)
  - On policy, probe `[On]` → `AlreadyThere`, nothing sent.
  - On policy, probe `[Standby, Standby, On]` → `Reached`, one power key, no WoL.
  - On policy, probe `[Standby]` then Standby for the whole 8 s window, then `[On]` → power key then WoL → `Reached`.
  - On policy, probe `[Unreachable, …, On]` → WoL sent once → `Reached`.
  - On policy, Unreachable and `tv.Mac == null` → `WakeOnLanUnavailable`.
  - On policy, Standby and remote throws `TvUnreachableException` → WoL sent immediately.
  - Off policy, probe `[On, On, Standby]` → `Reached`, one power key.
  - Off policy, probe `[Standby]` → `AlreadyThere`; `[Unreachable]` → `AlreadyThere`.
  - Off policy, remote throws `TvNotPairedException` → `NotPaired`.
  - Off policy, stays On for the whole window → `TimedOut`, exactly one power key (never a second toggle).
  - **Review Focus 3:** start On-run (Standby, slow to change); while it runs, start Off-run on the same IP → `IgnoredWhileTransitioning`.
  - Guard is released after every outcome (a second run afterwards is not ignored).
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** Use `Task.Delay(interval, time, ct)`. **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(core): add state-aware PowerSequence`

---

### Task 11: Plugin shell, settings, pairing and Remote Key action

**Files:**
- Create: `src/StreamDeck2Tizen.Plugin/StreamDeck2Tizen.Plugin.csproj` (OutputType Exe, AssemblyName `StreamDeck2Tizen`, `StreamDeck-Tools` 7.0.0, references Core; manifest/PI/images copied to output), `Program.cs`, `PluginServices.cs`, `TvKeySettings.cs`, `GlobalSettingsTokenStore.cs`, `KeyFeedback.cs`, `Actions/RemoteKeyAction.cs`, `com.brianlima.streamdeck2tizen.sdPlugin/manifest.json`, `com.brianlima.streamdeck2tizen.sdPlugin/PropertyInspector/remote-key.html`, `PropertyInspector/tv-connection.js` (shared IP/MAC/name fields + Pair button), vendored PI helper files from the StreamDeck-Tools samples (`sdtools.common.js`, `sdpi.css`)
- Modify: `StreamDeck2Tizen.sln` (add Plugin project)
- Test: none automated (thin adapter) — verification is the manual checklist in Step 5. Keep all logic in Core.

**Interfaces:**
- Consumes: everything in Core.
- Produces:
  - Plugin UUID `com.brianlima.streamdeck2tizen`; action UUIDs `…streamdeck2tizen.tvon`, `….tvoff`, `….remotekey`.
  - `TvKeySettings` (Newtonsoft `[JsonProperty]`): `tvIp`, `tvMac`, `displayName`, `rendererUrl`, `remoteKey` (default `KEY_MUTE`).
  - `GlobalSettingsTokenStore` — holds the current `TokenBook`; loads from global settings `{"tokens":{"<ip>":"<token>"}}` on `ReceivedGlobalSettings`, saves via `SetGlobalSettingsAsync` after pairing.
  - `PluginServices` — process-wide singletons: one `HttpClient`, one `PowerTransitionGuard`, `TimeProvider.System`, `GlobalSettingsTokenStore`; factory `TvStatusProbe ProbeFor(TvKeySettings)` (uses `MediaRendererSignal` when `rendererUrl` is set, else `AlwaysOnScreenSignal`); `PowerSequence`, `TvRemote`, `TvPairing` built from these.
  - `KeyFeedback.RunAsync(ISDConnection connection, string actionName, Func<Task<bool>> work)` — runs `work` off the event thread, `ShowOk` on true, `ShowAlert` on false **and on any exception**, logs the exception via `Logger.Instance.LogMessage(TracingLevel.ERROR, …)`; never throws.
  - PI → plugin message `{"command":"pair"}`; plugin → PI `{"pairingStatus":"<text>"}` with texts: `Paired ✓`, `Pairing denied on the TV`, `No answer — accept the prompt on the TV within 30 s`, `TV not reachable at <ip>`, `Invalid IP address`.
  - After a successful pair, if `rendererUrl` is empty, run `MediaRendererDiscovery.DiscoverAsync` (only if Task 5 implemented it) and save the found URL into the key's settings.

- [ ] **Step 1: Scaffold the plugin** from the StreamDeck-Tools 7 SamplePlugin conventions (`SDWrapper.Run(args)`, actions derive from `KeypadBase`, `[PluginActionId]`, `Tools.AutoPopulateSettings`). Verify the exact v7 API names against the StreamDeck-Tools 7.0 `SamplePlugin` / `MigrateTo7.0.md`; the names in this task are the expected ones.
- [ ] **Step 2: manifest.json** — `SDKVersion: 2`, `CodePath: "StreamDeck2Tizen.exe"`, `CodePathMac: "StreamDeck2Tizen"`, `OS: windows MinimumVersion "10"` + `mac "13"`, `Software.MinimumVersion "6.4"`, `Category: "Samsung TV"`, three actions (images can be placeholders until Task 13), `PropertyInspectorPath` per action.
- [ ] **Step 3: RemoteKeyAction** — `KeyPressed` → `KeyFeedback.RunAsync`: parse `TvAddress` (invalid → log which field, alert), look up token (missing → log "pair this TV first", alert), `TvRemote.SendKeyAsync`; `TvNotPairedException` → log "re-pair this TV", alert. PI: IP, MAC, display name, key dropdown filled from `RemoteKeys.Catalog` labels, Pair button + status line, and an editable "Screen-on check URL" field bound to `rendererUrl` (shown only if Task 5 was implemented).
- [ ] **Step 4: Build** — `dotnet build -c Release` → succeeded with 0 warnings; `dotnet test` still green.
- [ ] **Step 5: Manual check (user, Windows)** — `dotnet publish src/StreamDeck2Tizen.Plugin -c Release -r win-x64 --self-contained -o <sdPlugin folder>`, `streamdeck link <sdPlugin folder>` (Elgato CLI), add a Remote Key, enter IP, Pair, accept on TV, press → TV mutes; press again → unmutes without a prompt; restart Stream Deck → still no prompt; wrong IP → alert, other keys still respond.
- [ ] **Step 6: Commit** — `feat(plugin): add plugin shell, pairing and Remote Key action`

---

### Task 12: TV On and TV Off actions

**Files:**
- Create: `Actions/TvOnAction.cs`, `Actions/TvOffAction.cs`, `PropertyInspector/tv-power.html`
- Modify: `manifest.json` (point the two actions at `tv-power.html`)

**Interfaces:**
- Consumes: `PluginServices`, `KeyFeedback`, `PowerSequence.RunAsync`, `PowerOnPolicy`, `PowerOffPolicy`, `PowerOutcome`.
- Produces: outcome → feedback map (shared helper in `KeyFeedback` or a small `PowerOutcomeFeedback` class): `AlreadyThere`/`Reached` → OK; `IgnoredWhileTransitioning` → no feedback, log at INFO; `TimedOut` → alert, log "TV did not reach <state>"; `NotPaired` → alert, log "re-pair this TV"; `WakeOnLanUnavailable` → alert, log "set the TV's MAC address".

- [ ] **Step 1: Implement both actions** — thin: parse `TvAddress` (invalid → alert per Review Focus 1), token lookup, `RunAsync` with the policy, map outcome. TV On PI requires MAC (shows a hint if empty); TV Off PI hides MAC.
- [ ] **Step 2: Build** → 0 warnings; `dotnet test` green.
- [ ] **Step 3: Manual check (user)** — the spec §8 checklist: On from fully off (WoL); On from standby; On while on (OK, nothing happens); Off from on; Off while off (OK, nothing happens); press On then Off within 3 s (second ignored); TV unplugged → On shows alert after ~8–16 s and other keys stay responsive.
- [ ] **Step 4: Commit** — `feat(plugin): add state-aware TV On and TV Off actions`

---

### Task 13: Windows 11 (Fluent) icons

**Files:**
- Create: `com.brianlima.streamdeck2tizen.sdPlugin/Images/` — `plugin.svg`, `category.svg`, `action-tv-on.svg`, `action-tv-off.svg`, `action-remote-key.svg` (action-list icons), `key-tv-on.svg`, `key-tv-off.svg`, `key-remote.svg` (key images), `keys/<KEY_CODE>.svg` for the Remote Key catalog entries that have a Fluent glyph; `THIRD-PARTY-NOTICES.md` (Fluent UI System Icons, MIT, © Microsoft)
- Modify: `manifest.json` image paths; `RemoteKeyAction` sets the key image to `keys/<code>.svg` when the selected key has one (`SetImageAsync` with an SVG data URI), else `key-remote.svg`.

**Style (Windows 11 dark, values pinned):**
- Glyphs: Fluent UI System Icons, **Regular** weight, 24 px source grid — `tv`, `power`, `speaker_2`, `speaker_mute`, `speaker_1` (vol down), `home`, `arrow_left` (back), `arrow_up/down/left/right` (`chevron_*` if preferred), `checkmark` (enter), `play`, `pause`, `stop`, `fast_forward`, `rewind`, `video` (source), `navigation` (menu), `remote`.
- Action-list / category icons: monochrome `#FFFFFF` glyph on transparent (Elgato requirement), 20×20 (action) and 28×28 (category) viewBox.
- Key images (144×144 viewBox): rounded square, corner radius 16, fill `#2B2B2B` (Windows 11 dark layer), 1 px inner stroke `#FFFFFF` at 8 % opacity; glyph 72 px centered, `#FFFFFF`; TV On glyph uses Windows 11 dark-mode accent `#4CC2FF`.
- Plugin icon (256×256 viewBox): same tile style, `tv` glyph with `#4CC2FF`.

- [ ] **Step 1:** Fetch the listed SVGs from `github.com/microsoft/fluentui-system-icons` (`assets/<Name>/SVG/ic_fluent_<name>_24_regular.svg`). If unreachable, stop and tell the user rather than hand-drawing look-alikes.
- [ ] **Step 2:** Compose the tiles (hand-written SVG wrapping the glyph path, colours as pinned above).
- [ ] **Step 3:** `streamdeck validate src/StreamDeck2Tizen.Plugin/com.brianlima.streamdeck2tizen.sdPlugin` → no errors.
- [ ] **Step 4:** Render each key SVG to PNG (any local SVG renderer) and visually check legibility at 72×72; send the user a contact sheet.
- [ ] **Step 5: Commit** — `feat(plugin): add Windows 11 style icons`

---

### Task 14: Packaging and docs

**Files:**
- Create: `build/pack.ps1` (publish win-x64 self-contained into the `.sdPlugin` folder, then `streamdeck pack` → `dist/com.brianlima.streamdeck2tizen.streamDeckPlugin`)
- Modify: `README.md` (install, TV setup: enable "Power On with Mobile" / network standby, reserve the TV's IP on the router, pairing steps, troubleshooting by alert cause), `.github/workflows/build.yml` (add a job that runs `build/pack.ps1` and uploads the `.streamDeckPlugin` as an artifact)
- Delete: `spike/` (findings remain in `docs/spike-findings.md`)

- [ ] **Step 1:** Write `pack.ps1` with named variables for UUID, runtime and output paths.
- [ ] **Step 2:** Run it (user, Windows) → `.streamDeckPlugin` produced; double-click installs; repeat the Task 12 manual checklist on the installed build.
- [ ] **Step 3: Commit** — `chore: add packaging script, CI artifact and user docs`

---

## Execution notes

- Tasks 2–4 and 6–10 depend only on earlier tasks, not on the spike. Task 5's scope and the `TvTimings` values depend on `docs/spike-findings.md`; Tasks 11–14 need the user's Windows PC for manual checks.
- Build/test environment: requires the .NET 8 SDK and NuGet access. Every task's verification is `dotnet test` at the repo root unless stated.
