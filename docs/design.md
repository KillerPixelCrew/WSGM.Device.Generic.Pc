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

1. **Coupling to WSGM's Desktop/Game transition.** The natural use is "entering Game Mode applies
   the *TV* scene and switches the receiver to the PC input". The SDK has no lifecycle event for
   that transition today; the plugin can only offer the controls. A small SDK addition (an
   `ApplyModeAsync(Desktop|Game)` call, or a mode value in the descriptor state) would make the
   coupling a plugin-side rule instead of two QAM presses.
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
