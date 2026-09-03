# Generic PC plugin — design

Status: **draft, design only** (2026-09-03).

## 1. What this plugin is for

WSGM already owns, per display and per Desktop/Game mode, resolution, refresh rate, DPI and HDR
(`docs/power-and-display.md` in WSGM), default audio endpoint switching and volume
(`windows-device-control`), panel brightness, keep-awake and screen-off mute. None of that belongs
here.

What a desktop-to-TV setup still lacks is **scene switching**: which outputs are active at all, and
what the receiver on the other end of the cable is set to. The reference room:

```text
 PC ─┬─ DP ── Monitor 1 ─┐
     ├─ DP ── Monitor 2 ─┤  "Desk"
     ├─ DP ── Monitor 3 ─┘
     └─ HDMI 2.0, active fibre ──► HDMI audio extractor / VRR fix ─┬─ HDMI ──► TV        "TV"
                                                                    └─ audio ──► Onkyo 7.1 AVR
```

The extractor is a dumb box and gets no control. Everything else does.

## 2. Identity and ownership

- **Detection matches any 64-bit Windows PC.** There is no exact identity to gate on; the user
  installs this package deliberately into WSGM's single plugin slot. `DeviceDefinitionId` is
  `generic-pc`. A machine that has a first-party plugin (a Claw, a Handheld Companion install)
  should run that one instead — the slot holds exactly one package, so that is a choice, not a
  conflict.
- **Controller ownership: `Unmanaged`.** Pads talk to Steam directly, as on any desktop. The plugin
  publishes no physical devices, samples or haptic sink and declares
  `ControllerOwnership.Unmanaged` from the SDK extension proposed in the Handheld Companion
  design (section 12 there). Until that extension exists, WSGM's HidHide readability step still
  runs before each cycle; on a desktop with no HidHide installed that is a no-op, which is why
  this plugin can be developed before the extension lands.
- Nothing here needs elevation. `SetDisplayConfig` and TCP to the receiver run as the user.

## 3. Capabilities

Declared as SDK API 2 sections so the Device tab is plugin-authored.

### 3.1 Section `output` — key *Display*, icon *Display*

| Capability | Role · kind | What it does |
| --- | --- | --- |
| `output.scene` | `GenericChoice` · Choice: `desk`, `tv`, `both` | Applies a captured display arrangement with `SetDisplayConfig(SDC_APPLY \| SDC_USE_SUPPLIED_DISPLAY_CONFIG \| SDC_SAVE_TO_DATABASE)`. Readback compares the active `QueryDisplayConfig` paths against the scene; `AppliedVerified` only on an exact match. |
| `output.capture-desk`, `output.capture-tv`, `output.capture-both` | `GenericAction` | Snapshot the **current** arrangement (paths, modes, primary) into the scene of that name, stored under the plugin `StateDirectory`. Scenes are captured, never authored: the user sets the arrangement up once in Windows and presses *Capture*. |
| `output.active` | `GenericReadOnly` · Text | Names of the currently active outputs, e.g. `TV` or `Monitor 1 · Monitor 2 · Monitor 3`. |

Rules: a scene whose target IDs are no longer present (TV off, cable unplugged) is published
unavailable with `PrerequisiteMissing` naming the missing output, not applied blindly. Applying a
scene never changes a display's mode beyond what the snapshot recorded, so WSGM's own display
profiles keep the last word on resolution, refresh and HDR after the switch.

### 3.2 Section `receiver` — key *Custom* "Receiver", icon *Gauge*

Onkyo (and Pioneer, post-2016) receivers speak **eISCP** over TCP port 60128 and answer a UDP
discovery broadcast. The transport is a hundred lines of plain sockets; every write is a query
round-trip away from a verified readback.

| Capability | Role · kind | eISCP |
| --- | --- | --- |
| `receiver.power` | `GenericToggle` | `PWR01` / `PWR00`, readback `PWRQSTN` |
| `receiver.volume` | `GenericRange` · Integer 0–80 (receiver units, `Step` 1) | `MVLxx` (hex), readback `MVLQSTN` |
| `receiver.mute` | `GenericToggle` | `AMT01` / `AMT00`, readback `AMTQSTN` |
| `receiver.input` | `GenericChoice` from the discovered selector list | `SLIxx`, readback `SLIQSTN` |
| `receiver.listening-mode` | `GenericChoice` (Stereo, Direct, Dolby Surround, DTS Neural:X, All Ch Stereo, …) | `LMDxx`, readback `LMDQSTN` |
| `receiver.model` | `GenericReadOnly` · Text | from discovery (`ECN`) |

Availability: the whole section is unavailable with `PrerequisiteMissing` while no receiver
answers, and every command revalidates the connection first. Bounds are read from the receiver
where it reports them (`MVL` maximum), otherwise the conservative 0–80.

### 3.3 Section `system` — key *General*, icon *Wrench*

| Capability | Role · kind | Source |
| --- | --- | --- |
| `system.power-plan` | `GenericChoice` of the installed schemes | `PowerEnumerate` / `PowerSetActiveScheme`, readback `PowerGetActiveScheme` |
| `system.cpu-temperature`, `system.gpu-temperature` | `Telemetry` · Integer · Celsius | **later**, only if a dependency-free source turns out to exist; LibreHardwareMonitor needs a kernel driver and is not worth it for a desktop |

## 4. Settings (WSGM-drawn, plugin-declared)

| Setting | Kind | Purpose |
| --- | --- | --- |
| `receiver.host` | Text (≤ 64) | Fixed receiver address; empty = eISCP discovery |
| `receiver.volume-ceiling` | Integer 20–100 | Upper bound the volume slider may reach, for ears and speakers |
| `output.scene-names.*` | Text (≤ 24) | Optional labels for the three scenes |

## 5. Open questions

1. **Coupling to WSGM's Desktop/Game transition** — resolved by section 8: `ApplySessionModeAsync`
   and `PrepareSessionModeAsync` in the SDK, plus the Desktop-First mode in WSGM core.
2. **Scene shape.** Whole `DISPLAYCONFIG_PATH_INFO` + `MODE_INFO` snapshots are exact but brittle
   across driver updates; a reduced form (target IDs, primary, per-target mode) is more robust and
   is the intended first implementation.
3. **Receiver zones.** Zone 2/3 and the equivalent Pioneer vocabulary are left out of v1.
4. **Naming.** `wsgm.device.generic.pc` and "Generic PC" are placeholders.

## 6. Not in scope

Audio endpoint switching, volume and mute of the PC (WSGM core), display resolution, refresh, HDR
and DPI (WSGM display profiles), brightness (core), the HDMI extractor / VRR fix (no interface),
controller emulation (Steam handles desktop pads), fans, TDP, lighting (no generic path on a
desktop).

## 7. The EDID problem, and why "wait" is a state

The HDMI audio extractor is also a switch, and **its inactive input sends no EDID**. While the
switch is on another source the TV is not "off" from Windows' point of view — the display target
does not exist at all. Any tool that applies a display profile (DisplayMagician included) fails
until the user first picks up the extractor's remote and switches the input, and only then can
the software act.

HDMI-CEC would be the usual fix and is a **no-go**: PC graphics cards do not expose CEC on their
HDMI outputs. The one exception is the Steam Machine, whose HDMI is wired for it; on anything else
the extractor and the TV can only be told what to do by infrared (section 9) or by hand.

So the plugin never treats an absent designated display as an error. It is a **waiting state**
with a reason the user can act on:

| State | Meaning | What the plugin publishes |
| --- | --- | --- |
| `present` | the designated target enumerates with a mode | `output.scene` available |
| `waiting` | the target is absent; nothing to apply yet | `output.scene` unavailable, `PrerequisiteMissing("TV")` with `Retryable = true`; `output.designated-display` reads *Waiting for TV — switch the HDMI input* |
| `arrived` | a display-change notification brought the target back | scene applied; in on-demand mode the takeover continues |

Arrival is observed, not polled: `WM_DISPLAYCHANGE` / `WM_DEVICECHANGE` on the plugin's message
window, confirmed by `QueryDisplayConfig`. Because `SetDisplayConfig` right after arrival can race
the driver's own mode set, the plugin waits for one stable enumeration (two identical reads 500 ms
apart) before applying a scene.

## 8. A new WSGM mode: Desktop First, Game Mode on Demand

Today WSGM 2.0 has one posture: the auto-start service launches `WSGM.exe --boot` at logon and the
shell takeover runs unconditionally — splash over the booting desktop, Explorer exited, Steam Big
Picture started (`docs/boot-and-shell.md` in WSGM). That is right for a handheld and wrong for a
desk PC that is a workstation most of the day.

**Desktop First** is a WSGM core feature this plugin is the first reason for. Windows boots
normally, Explorer stays the shell, WSGM sits in the background. When it is time to game on the TV:

1. **Hotkey** (keyboard chord, or a controller chord once a pad is connected) → WSGM shows its
   splash, now over a running desktop rather than a booting one.
2. **Prerequisite gate.** The splash shows the plugin's waiting text (*Waiting for TV — switch the
   HDMI input*) until the device plugin reports the designated display present. Nothing has been
   torn down yet: *Cancel* on the splash returns to the desktop with no side effects. Later, with
   the IR blaster, this step begins by *sending* the switch command instead of asking for it.
3. **Scene.** The plugin applies the `tv` scene and the receiver input; WSGM's Game-mode display
   profile then applies resolution, refresh and HDR on the now-present target.
4. **Takeover.** The existing `--boot` sequence from "posture" onward: exit Explorer, tray host,
   startup apps, Steam Big Picture.
5. **Back.** The existing *Switch to desktop* transition, extended with the reverse scene: `desk`
   scene and the receiver back to the desk input. The TV going away while in Game mode (extractor
   switched back by hand) is not an emergency: WSGM's profiles already survive display loss, and
   the scene state simply returns to `waiting`.

What this needs in **WSGM core** (not this plugin):

- A boot-manifest / config value `SessionStart: BootToGameMode | DesktopFirst`. In `DesktopFirst`
  the service still launches WSGM at logon (so elevation through the linked token keeps working),
  but `--boot` runs an **agent** posture: tray icon, hotkey registration, device plugin cycle
  started, no takeover.
- The splash gains a *prerequisite* line and a cancel that is safe before Explorer exit. The
  existing *Switch to desktop* recovery already has the "skip every game-mode side effect" branch;
  this reuses it.
- A global hotkey registration (`RegisterHotKey`) in agent posture; the chord itself is a setting.

What this needs in the **Device SDK**, alongside the controller-ownership extension from the
Handheld Companion design:

| Addition | Why |
| --- | --- |
| `CapabilityRole.SessionPrerequisite` — a read-only boolean capability with a bounded waiting label | Lets the host gate Game-mode entry on a plugin fact without knowing capability ids; the splash renders the label. A plugin may declare several (display present, receiver reachable). |
| `IDevicePlugin.ApplySessionModeAsync(SessionMode mode, deadline)` with `SessionMode { Desktop, Game }` | Called by the host before takeover and after return, so scene and receiver input follow the mode as one plugin-side rule instead of two QAM presses. Default implementation does nothing. |
| `IDevicePlugin.PrepareSessionModeAsync(SessionMode, deadline)` | Runs *before* the prerequisite gate: this is where the IR blaster sends "switch to PC" so the wait resolves itself. Default no-op. |

## 9. Planned: network IR blaster (ESP32-S3)

Because CEC is unavailable, the devices around the cable — the extractor's input select, the TV's
power and input, the receiver where eISCP is not enough — will be driven by a **custom ESP32-S3
network IR blaster that learns codes**. It is a separate project (a prototype is being built as
of 2026-09-03); the plugin only speaks to it.

- Section `remote` (key *Custom* "Remote", icon *Wrench*): every learned code is one
  `GenericAction` (`remote.tv-power`, `remote.extractor-input-pc`, …) plus a
  `remote.reachable` read-only row. Codes are learned on the blaster's own UI and listed by the
  plugin from the blaster's inventory endpoint; the plugin never stores IR data.
- **Sequences** bind to session modes: entering Game mode runs `extractor-input-pc`, `tv-power-on`;
  returning runs the desk equivalents. These are the bodies of `PrepareSessionModeAsync`.
- Transport is undecided until the firmware exists: plain HTTP/JSON on the LAN with a fixed host
  setting is the least surprising; mDNS discovery is a nicety. No cloud, no broker.
- A blaster that does not answer degrades the `remote` section only; the manual path of section 7
  keeps working.

Steam Machine note: on Valve's hardware HDMI-CEC would make the IR path unnecessary for the TV and
possibly the extractor. This design keeps CEC out entirely rather than half-supporting it; a
future Steam-Machine-specific plugin would own it.

## 10. Planned: Home Assistant

Smart TVs, network-capable AV receivers and soundbars already have Home Assistant integrations,
and HA is where a living room's automations live anyway. The plugin therefore gets an HA backend
next to eISCP and the IR blaster, and all three plug into the same two places: the `remote`-style
action rows, and the session-mode sequences of section 8.

- **Transport.** HA's REST API (`/api/services/<domain>/<service>`, `/api/states`) over the LAN
  with a long-lived access token; the WebSocket API only if state pushes turn out to matter. Both
  are a few hundred lines of `HttpClient`, no client library. Settings: `homeassistant.url`,
  `homeassistant.token` (text, stored by WSGM like every other setting), and an entity allow-list
  so the Device tab does not fill with the whole house.
- **Section `home`** (key *Custom* "Home Assistant", icon *Wrench*). Entities are projected onto
  generic roles by domain, nothing more clever than that:

  | HA domain | Capability |
  | --- | --- |
  | `scene`, `script`, `button` | `GenericAction` |
  | `switch`, `input_boolean`, `media_player` power | `GenericToggle` |
  | `media_player` volume | `GenericRange` 0–100 |
  | `media_player` source, sound mode; `select`, `input_select` | `GenericChoice` from the entity's option list |
  | any entity's state | `GenericReadOnly` · Text |

  Readback is the entity state after the service call, polled once with a short deadline, so a
  `media_player.select_source` that HA accepted but the TV ignored is `AppliedUnverified`, not
  `AppliedVerified`.
- **Sequences.** A session-mode step may be an HA service call (`scene.turn_on scene.gaming_tv`),
  so the same "entering Game mode" rule can run one HA scene instead of three IR codes. The IR
  blaster of section 9 can itself live behind HA: an ESP32-S3 running ESPHome with its
  `remote_transmitter` / `remote_receiver` components integrates natively and would let the
  learned codes be HA services too, which keeps the plugin's own IR transport optional.
- **Degrading.** An unreachable HA, a bad token, or a removed entity degrades only the affected
  rows; the local paths (display scenes, eISCP) never depend on it.
- **Not a controller for HA.** The plugin calls services and reads states. It does not host
  automations, does not expose WSGM to HA (a later "WSGM as an HA device" is a different feature),
  and never stores anything but the URL, the token and the allow-list.
