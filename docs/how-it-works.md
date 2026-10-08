# How it works — data path & maneuverType catalogue

## The problem
Mazda Connect (CMU150, FW 74.00.324) renders Android Auto and native navigation on the
instrument-cluster HUD, but **never CarPlay turn-by-turn**. An exhaustive search of the
firmware shows there is no Apple-iAP2 `maneuverType` → HUD-icon mapping anywhere on the
unit: the OEM `jciCARPLAY.so` "TurnByTurn" subsystem is status/arbitration only (a boolean
"who owns TBT"), carrying no geometry. The head unit was simply never built to draw CarPlay
maneuvers — it relies on phone-screen mirroring.

So the maneuver decode must live in **our** bridge.

## Data path
```
iPhone (CarPlay, actively navigating)
   │  Apple iAP2 over USB/wireless
   ▼
usr/bin/ipoddev_full_auth        (OEM iAP2 producer)
   │  viAP2NavRouteManeuverUpdate -> SysV message queue (msgsnd)
   ▼
jciCARPLAY  (OEM process)  ◄── our shim is LD_PRELOAD'ed in here
   │  the OEM receive thread calls libc msgrcv() to pull each event
   │
   ├── devmgr_shim.cpp: we interpose msgrcv() (PLT/preload). Every message the OEM
   │   receives also passes through us — we COPY-READ it and always chain the real call
   │   (fully passive, zero crash risk, never alters the data).
   │      maneuver burst = type 0x8059 (840 B)
   │      guidance       = type 0x8058 (1628 B)
   │
   ├── nav.cpp: decode maneuverType / distance / road-name, pick the first REAL maneuver
   │
   ▼
hud/hud_send.cpp -> com.jci.vbs.navi  (OEM HUD nav D-Bus service, via libjcidbus)
   │   VBS_NAVI_SetHUDDisplayMsgReq (+ SetHUD_Display_Msg2)
   ▼
Instrument-cluster HUD  → arrow + distance + road name
```

`NaviSupported=TRUE` in `/etc/devmgr_config_master.xml` is what makes Apple advertise +
stream the maneuver data over iAP2 in the first place — the installer sets it.

## Activation / gating (no DRM)
- **`main.cpp` constructor** reads `/proc/self/cmdline`; it only arms (`g_enabled=true`)
  if the process is the `jciCARPLAY` launcher (`in_carplay_launcher`). In any other process
  that happens to inherit `LD_PRELOAD`, the shim stays completely inert and chains straight
  through to the real OEM functions.
- The PLT shims for `eDevMgriPodStartCallStateUpdates` / `…CommunicationUpdates` /
  `…SendLocationInformation` call `arm_navi()` when a CarPlay session sets up; that brings up
  the HUD sender and flips `g_armed` so the passive `msgrcv` tap begins decoding maneuvers.

## iAP2 nav message model (reverse-engineered)
The OEM `ipoddev` process decodes the phone's iAP2 route-guidance messages
(`RouteGuidanceUpdate` 0x5201 and `RouteGuidanceManeuverUpdate` 0x5202) into two SysV
messages. Offsets are the ones `nav.cpp` reads:

- **MANEUVER `0x8059` (840 B)** — one message per maneuver in the phone's list (idx 0..N),
  re-sent from idx 0 on every recalculation:
  - `validInfo` = `u32 @ 0x1c` — field-presence bits (`0x10` road name, `0x400` exit angle,
    `0x800` roundabout junction)
  - `index` = `u32 @ 0x20 >> 16`
  - `maneuverDesc` = text @ `0x24` (256 B) — the instruction text; `u32 @ 0x124` is its length
  - `maneuverType` = `u32 @ 0x128` — Apple's `CPManeuverType` (table below)
  - `afterManeuverRoadName` = text @ `0x12c` (256 B)
  - `driveSide` = `u32 @ 0x33c` — 0 right-hand traffic, 1 left-hand traffic
  - `junctionType` = `u32 @ 0x340` — 0 intersection, 1 roundabout
  - `junctionElementExitAngle` = `s16 @ 0x346` — signed degrees: 0 straight, negative left,
    positive right
- **GUIDANCE `0x8058` (1628 B)** — `distToNextManeuver` (metres, counts down) @ `0x34c`;
  `maneuverList[0]`, the index of the next maneuver, `u16 @ 0x458`; list count `u16 @ 0x656`.
- **Selection:** the bridge buffers the displayable maneuvers of the current list (idx 0 and
  context types skipped) and shows the one whose index matches `maneuverList[0]`, or the next
  one when that index points at a context entry.

Google Maps fills in fewer fields than Apple Maps. In iPhone packet logs from drives in
Australia on 2026-09-28, every Google Maps route start and reroute put `StartRoute` at idx 0, so skipping
idx 0 is safe for it too. None of its maneuvers carried an exit angle, a junction-element
angle, exit info or linked lane guidance, and its final arrival maneuver reported driving
side Right even in left-hand traffic.

Wire-level field lists for 0x5200–0x5204 are documented in
[luka-dev/mib2q-carplay-rgi `docs/rgd/rgd-tlv.md`](https://github.com/luka-dev/mib2q-carplay-rgi/blob/main/docs/rgd/rgd-tlv.md).
Apple's [CarPlay Developer Guide](https://developer.apple.com/download/files/CarPlay-Developer-Guide.pdf)
("Share upcoming maneuvers with vehicle") defines each maneuver type.

## maneuverType → HUD icon
`maneuverType` is Apple's `CPManeuverType` enum. An earlier version of this document listed
11 = left and 12 = right: those were read from `0x124`, the instruction-text length, not the type.

| type | CPManeuverType | HUD code |
|-----:|----------------|----------|
| 0, 5, 11, 18 | noTurn, followRoad, startRoute, startRouteWithUTurn | context, not shown |
| 1 / 2 | leftTurn / rightTurn | 2 / 3 |
| 3 | straightAhead | 1 |
| 4, 26 | uTurn, uTurnWhenPossible | 10 in left-hand traffic, 13 in right-hand |
| 6, 7, 19, 28–46 | enter/exit roundabout, U-turn at roundabout, roundaboutExit1–19 | 37–60, by exit angle (`roundabout_icon()`) |
| 8 | offRamp | 30 or 7 from the exit-angle sign; the kerb side when there is no angle (always the case with Google Maps) |
| 9 | onRamp | 17 (merge right) in left-hand traffic, 16 (merge left) in right-hand |
| 10, 12, 27 | arrive | 8 |
| 13 / 14 | keepLeft / keepRight | 15 / 14 (fork) |
| 15–17 | enter / exit / change ferry | blank |
| 20 / 21 | leftTurnAtEnd / rightTurnAtEnd | 31 / 32 (T-junction) |
| 22 / 23 | highwayOffRampLeft / Right | 30 / 7 |
| 24 / 25 | arriveAtDestinationLeft / Right | 33 / 34 |
| 47 / 48 | sharpLeftTurn / sharpRightTurn | 11 / 9 |
| 49 / 50 | slightLeftTurn / slightRightTurn | 4 / 5 |
| 51 | changeHighway | 1 |
| 52 / 53 | changeHighwayLeft / Right | 15 / 14 |

## Why a passive `msgrcv` tap (and not the OEM vtable)
The OEM receive thread inside `jciCARPLAY` pulls every devmgr event through the libc
`msgrcv` PLT entry. Interposing `msgrcv` lets the bridge *see* every message — maneuvers
included — while always forwarding the real call unchanged. It never modifies the OEM
dispatch, vtable, or data, so it cannot crash or destabilise the OEM process: it is a pure
read-side tap.
