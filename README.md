# WSGM Device Plugin — Generic PC

A [WSGM](https://github.com/NightHammer1000/WSGM) device plugin for an ordinary desktop PC: no
vendor firmware, no handheld controller, no fan or power-limit registers. What it adds is the part
of a living-room setup WSGM's core does not own — which outputs are active and what the AV
receiver is doing — so a desk-to-TV switch is one Quick Access Menu action.

The reference setup it is written for: three monitors on the desk, one HDMI 2.0 run over active
fibre to an HDMI audio extractor (with VRR fix), video from there to the TV and audio to an Onkyo
7.1 AV receiver. See [`docs/design.md`](docs/design.md).

**Status: design.** The repository layout, packaging script, CI and submodule pins follow the
[reference plugin](https://github.com/KillerPixelCrew/WSGM.Device.Msi.Claw8A2Vm); the plugin
sources are not written yet. It depends on the controller-ownership SDK extension proposed in the
[Handheld Companion plugin's design](https://github.com/KillerPixelCrew/WSGM.Device.HandheldCompanion/blob/main/docs/ipc-protocol.md#12-required-sdk-extension-controller-ownership).

## Building

```powershell
git clone --recursive https://github.com/KillerPixelCrew/WSGM.Device.Generic.Pc
dotnet build WSGM.Device.Generic.Pc.slnx
dotnet test  WSGM.Device.Generic.Pc.slnx
./eng/pack.ps1
```

`--recursive` matters: the SDK and Device Lab are commit-pinned submodules.

## Licence

MIT. See `LICENSE`.
