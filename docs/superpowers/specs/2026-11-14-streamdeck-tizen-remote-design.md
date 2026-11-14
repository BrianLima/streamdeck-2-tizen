# Stream Deck → Samsung Tizen TV Remote — Design

Date: 2026-11-14
Status: draft, awaiting approval (reviewed by builder check)

## 1. Intent

A personal Stream Deck plugin that controls a Samsung Tizen TV on the same home network as the PC running Stream Deck. It talks to the TV directly over the LAN (no Samsung cloud, no subscription, no Home Assistant).

The headline feature is **power actions that check the TV's state before acting**, so a press never does the opposite of what was intended (the TV's own power key is a toggle).

## 2. Goals

1. **TV On** key action:
   - TV reports `on` → do nothing (show OK).
   - TV reports `standby` → send `KEY_POWER` over the remote WebSocket.
   - TV does not answer → send a Wake-on-LAN magic packet.
   - Then confirm the TV reached `on`; if it didn't within the confirmation window after `KEY_POWER`, send Wake-on-LAN as a fallback (some TVs ignore network keys in standby).
2. **TV Off** key action (dedicated, separate key):
   - TV reports `on` → send `KEY_POWER`.
   - `standby` or no answer → do nothing (show OK).
3. **Remote Key** action: send any single remote key (volume up/down, mute, source, home, back, arrows, enter, play/pause, etc.), chosen from a dropdown in the key's settings. This covers the project's broader purpose of sending remote inputs.
4. One-time **pairing** from the settings panel; the TV's token is saved and reused so the TV never asks again.
5. Clear failure feedback on the key (Stream Deck's alert triangle) plus a log line saying why.
6. Runs on **Windows** (primary). **macOS** is a nice-to-have: don't block it, don't spend effort on it in v1.

## 3. Non-goals (v1)

- Keys that continuously show live TV state (lit when on). Status is checked only on press. Can be added later; the design keeps the status checker separate so it can be polled.
- Launching TV apps (unreliable on 2023+ models).
- Stream Deck+ dials.
- Multiple TVs in one action / auto-discovery of TVs on the network.
- SmartThings / cloud control.
- Publishing to the Elgato Marketplace.

## 4. Anti-goals (failure even if it "works")

- The TV pops "Allow this device?" on every press.
- TV On turns the TV off, or TV Off turns it on (double press, stale state).
- A slow or offline TV freezes the Stream Deck app or other keys.
- Magic numbers or opaque logic in the code (see §5).

## 5. Constraints

- **TV only, LAN only:** the plugin talks directly to the TV on the local network, in the spirit of Apps2Samsung (C#, direct-to-TV). No SmartThings API, no Samsung account, no cloud services — not even as an optional fallback. *(User, 2026-11-14.)*
- **Language:** C#, using BarRaider's StreamDeck-Tools (v7+, cross-platform) as the Stream Deck wrapper. Plugin ships as a self-contained .NET build (Stream Deck launches it as an executable).
- **Target TVs:** the user's own Samsung TV(s), not every Tizen model. Primary path is the 2018+ protocol (secure WebSocket on 8002 with pairing token). 2016–2017 models use plain `ws://…:8001` with no token and often lack `PowerState`; support for that is added only if the spike shows the user's TV needs it. *(Builder's call, recorded by Claude.)* User's exact model is **not yet known** — see §9 Milestone 0.
- **TV interfaces used:**
  - Status: `GET http://<tv-ip>:8001/api/v2/` → JSON; `device.PowerState` is `"on"` or `"standby"` on newer models. No response = unreachable.
  - Keys: `wss://<tv-ip>:8002/api/v2/channels/samsung.remote.control?name=<base64 name>&token=<token>`, message `{"method":"ms.remote.control","params":{"Cmd":"Click","DataOfCmd":"KEY_…","Option":"false","TypeOfRemote":"SendRemoteKey"}}`. The TV uses a self-signed certificate; accept it only for the configured TV IP.
  - Pairing: first connect without a token → TV shows a prompt → on accept, `ms.channel.connect` event carries `data.token`. `ms.channel.unauthorized` = denied; `ms.channel.timeOut` = prompt never answered / model doesn't support pairing.
  - Wake-on-LAN: standard magic packet (6×0xFF + MAC×16) over UDP to the broadcast address, port 9. Requires the TV's "Power On with Mobile"/network standby setting to be enabled.
- **Code style (user's standing rules):** Sandi Metz's POODR / 99 Bottles principles — small objects with one responsibility, dependencies injected, decisions isolated from I/O. Self-explanatory names, no redundant comments, no unexplained numbers: every timeout/port/delay is a named constant.
- Network calls must be async with timeouts and never block the Stream Deck event loop.

## 6. Approaches considered

| Approach | Trade-offs | Decision |
|---|---|---|
| **C# + StreamDeck-Tools, own TV client** | User's preferred desktop language; we own ~200 lines of protocol code, which allows clean object design. Depends on a community wrapper; larger download (bundled runtime). | **Chosen** by user. |
| TypeScript + official Elgato SDK + `samsung-tv-remote` npm | Officially supported; less code. Not the user's preferred language; TV quirks handled inside a third-party library. | Not chosen. |
| Python script via Stream Deck's built-in "Open" action | No plugin needed. No settings UI, no key feedback, reconnects every press. | Dropped. |

## 7. Design

### 7.1 Objects (responsibilities, not final names)

- **`TvAddress`** — value object: IP, MAC, display name. Validates format.
- **`TvStatusProbe`** — asks the TV for its state; returns `TvPowerState { On, Standby, Unreachable }`. It combines two signals, because some TVs keep answering `/api/v2/` in standby or don't report `PowerState` (user chose this over a plain toggle or dropping TV Off):
  - **Signal A — device info:** `GET /api/v2/` → `device.PowerState`.
  - **Signal B — screen-on service:** the TV's UPnP/DLNA media renderer, which on many models only runs while the screen is on. Its address is found once via SSDP discovery during the spike and stored as a setting (the port is commonly reported as 9197, but must be confirmed on the user's TV).
  - Rules:
    - Signal A doesn't answer → Unreachable.
    - Signal A says `"standby"` → Standby.
    - Signal A answers (`"on"` or no field) **and** Signal B answers → On.
    - Signal A answers but Signal B doesn't → Standby.
  - Each signal sits behind its own small object so the probe only combines answers; the spike decides whether Signal B is needed at all (if `PowerState` alone proves reliable, Signal B is left out).
  - **If the spike finds neither signal reliable:** TV Off becomes a plain power toggle, and its settings panel shows a warning that it can't tell standby from on. TV On keeps Wake-on-LAN for the "unreachable" case.
  - Frame / Art Mode TVs are out of scope unless the user's TV turns out to be one.
- **`TvRemote`** — opens the secure WebSocket with the saved token, sends one key, closes. Raises a distinct error for "not paired / unauthorized".
- **`TvPairing`** — runs the pairing handshake and returns the token.
- **`WakeOnLanSender`** — builds and broadcasts the magic packet.
- **`PairingTokenStore`** — reads/writes tokens in the plugin's global settings, keyed by TV IP (so several keys pointing at the same TV share one token).
- **`PowerOnPolicy`** / **`PowerOffPolicy`** — pure decision objects: given a `TvPowerState`, return what to do (`DoNothing`, `SendPowerKey`, `SendWakeOnLan`). No I/O; fully unit-testable.
- **`PowerTransitionGuard`** — per-TV lock: after a power command is sent, further power presses for that TV are ignored until the probe reports the expected state or the guard window expires. Prevents the double-press toggle problem.
- **Stream Deck actions** (thin): `TvOnAction`, `TvOffAction`, `RemoteKeyAction` — read settings, call the objects above, show OK/alert.

### 7.2 TV On flow

1. If guard is active for this TV → ignore press.
2. Probe → state.
3. `PowerOnPolicy` decides:
   - On → show OK, stop.
   - Standby → send `KEY_POWER`, start guard.
   - Unreachable → send WoL, start guard.
4. Confirm: poll the probe every *confirmation interval* until On or *confirmation window* expires.
   - If the action was `KEY_POWER` and the window expires → send WoL once, wait one more window.
   - On success → show OK. On final failure → show alert, log "TV did not turn on".

### 7.3 TV Off flow

1. Guard check → probe → `PowerOffPolicy`.
2. On → send `KEY_POWER`, start guard, show OK. Standby/Unreachable → show OK, do nothing.

### 7.4 Remote Key flow

Probe is **not** used (keeps volume presses fast). Send the key; if the socket can't connect → alert. Holding a key does not auto-repeat in v1.

### 7.5 Settings (Property Inspector, per key)

- TV IP (required), TV MAC (required for TV On), display name (optional, defaults from the TV's reported name). The TV should have a fixed IP (DHCP reservation on the router); the plugin does not rediscover a moved TV in v1.
- Media renderer address (filled automatically by the Pair step via SSDP, editable) — only if the spike shows Signal B is needed.
- **Pair** button: runs pairing, tells the user to accept the prompt on the TV, shows result (paired / denied / no prompt appeared).
- Remote Key action only: key dropdown.

### 7.6 Named timing constants (starting values, tune in Milestone 0)

| Constant | Start value | Purpose |
|---|---|---|
| Status probe timeout | 1.5 s | Max wait for `/api/v2/` before treating TV as unreachable |
| WebSocket connect timeout | 3 s | Max wait to open the key channel |
| Pairing prompt timeout | 30 s | Time the user has to accept on the TV |
| Power confirmation interval | 1 s | How often to re-probe after a power command |
| Power confirmation window | 8 s | How long to wait for the expected state before fallback/failure |
| Power transition guard | 10 s | How long repeat power presses are ignored |

## 8. Testing

- Unit tests (xUnit) for: both policies (every state), guard behaviour, probe JSON parsing (with/without `PowerState`, malformed), magic packet bytes, token store keyed by IP. TV, network and clock injected as fakes.
- Manual checklist on the real TV: pair; On from fully off (WoL); On from standby; Off from on; double-press On/Off; volume/mute keys; TV unplugged → alert, Stream Deck stays responsive.

## 9. Milestones

0. **Feasibility spike (throwaway, C# console):** against the user's TV, confirm:
   - pairing succeeds and the token sticks across reconnects;
   - which protocol the TV speaks (8002 + token vs. 8001 without);
   - what `/api/v2/` reports for `PowerState` when on, right after switching off, and after it has been off for 10+ minutes;
   - whether the UPnP media renderer (found via SSDP) answers only while the screen is on;
   - that WoL wakes the TV from fully off.
   Outcome picks the status rule in §7.1, tunes §7.6, and confirms the TV model is supported. **Stop and re-plan if pairing fails.**
1. Core library (no Stream Deck): probe, remote, pairing, WoL, token store, policies, guard + tests.
2. Stream Deck plugin shell with StreamDeck-Tools; Remote Key action + Property Inspector + Pair button.
3. TV On / TV Off actions with confirmation and fallback.
4. Packaging (`.streamDeckPlugin`, Windows). macOS build attempted only if it costs little.

## 10. Left to the builder

- Project/solution layout, exact class names, logging library (StreamDeck-Tools ships NLog).
- Key icons and Property Inspector styling.
- Exact list of keys in the Remote Key dropdown (start with the common ~20).
- Broadcast address choice for WoL (limited broadcast vs. subnet broadcast — use both if simple).

## 11. Open items

- **User's TV model** — needed before Milestone 0.
