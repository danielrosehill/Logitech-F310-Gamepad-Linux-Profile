# Logitech F310 Gamepad: Linux identification and ASR-recorder mapping

Identifies the Logitech F310 connected to `daniellaptop` on 2026-11-13 and records the button mapping
Daniel wants for the ASR-Training recorder (`audio-dictation/ASR-Training/recorder`).
Machine-readable copy: [`device.json`](device.json).

## Identification (confirmed live, 2026-11-13)

| Field | Value |
|---|---|
| Product | Logitech Gamepad F310, rear switch in **X** (XInput) mode |
| USB ID | `046d:c21d` (`lsusb`: "F310 Gamepad [XInput Mode]") |
| udev `ID_SERIAL` | `Logitech_Gamepad_F310_<serial>` (each pad has its own 8-hex-digit serial; omitted) |
| Kernel driver | `xpad` |
| Nodes | `/dev/input/by-id/usb-Logitech_Gamepad_F310_<serial>-event-joystick` (event18 at capture time), `js0` |
| Force feedback | `FF_RUMBLE`, `FF_PERIODIC` |

In **D** (DirectInput) mode the switch changes the USB ID to `046d:c216` and the button layout; everything below
assumes X mode. Leave the switch on X.

### evdev codes (`python3 -m evdev`, `evtest`)

| Physical | evdev | Browser `Gamepad.buttons[i]` (standard mapping) |
|---|---|---|
| A (green, bottom) | `BTN_SOUTH` 304 | 0 |
| B (red, right) | `BTN_EAST` 305 | 1 |
| X (blue, left) | `BTN_X` 307 (kernel alias `BTN_NORTH`, misleading) | 2 |
| Y (yellow, top) | `BTN_Y` 308 (alias `BTN_WEST`) | 3 |
| LB / RB | `BTN_TL` 310 / `BTN_TR` 311 | 4 / 5 |
| LT / RT | `ABS_Z` / `ABS_RZ` 0..255 (analog) | 6 / 7 |
| Back / Start | `BTN_SELECT` 314 / `BTN_START` 315 | 8 / 9 |
| L / R stick click | `BTN_THUMBL` 317 / `BTN_THUMBR` 318 | 10 / 11 |
| D-pad | `ABS_HAT0X` / `ABS_HAT0Y` (-1..1) | 12-15 |
| Logitech (Guide) | `BTN_MODE` 316 | 16 |

All 17 rows were press-verified on 2026-11-13, one button at a time, with the public `input-device-mapper` plugin
(`idmap.py press 046d:c21d --label <button>`). The triggers are analog axes (`ABS_Z+`, `ABS_RZ+`), not keys. Stick
movement (left `ABS_X`/`ABS_Y`, right `ABS_RX`/`ABS_RY`) was not mapped, since only buttons were needed; the stick
clicks were. The
browser index column is the W3C standard-mapping prediction; it has not been read from inside the browser.

## Target mapping for the ASR recorder (Daniel, 2026-11-13)

| Button | Recorder action (`shared/keys.js` id) | pad index |
|---|---|---|
| X | Record / stop (`record`) | 2 |
| B | Save · next (`save`) | 1 |
| Y | Retake / restart (`retake`) | 3 |
| A | Skip, which is Skip all while the recorder's `skipDefault` is "all" (`skip`) | 0 |
| Logitech (Guide, round) | Focus mode on/off: full screen, keys held (`focus`), added in release 448 | 16 |

The button printed **MODE** is handled inside the pad (it swaps the left stick and the D-pad and lights its LED) and
sends no event, so it cannot be bound. A gamepad press is not a browser user gesture, so the page's own full-screen request
is refused for it; desktop shell 1.10 makes the window full screen itself (ASR-Training commit `4589cb0`).

The rest are unassigned for now. The recorder already reads gamepads natively (Gamepad API polling, release 195), and
pad bindings are stored separately from keyboard bindings, so the keyboard shortcuts are unaffected by plugging or
unplugging the pad. What it lacked was a **per-device** pad profile (the F310 and the 8BitDo Micro want different
layouts). Shipped in recorder release 446 (v1.445, ASR-Training commit `506035a`, 2026-11-13): pads keyed by
`vendor:product` from `Gamepad.id`; each pad uses its own learned map, else the built-in F310 preset (`046d:c21d`,
the table above), else the global default. Release 447 (v1.446, commit `d38f6f7`) syncs pad maps per user to R2
`pad-profiles/<user>.json` via `api/pad-profiles.js`; a device without a map for a connected pad offers the server
map ("Use it" / "No", No remembered per pad per device). Keyboard bindings stay device-local. Not yet tried on the
physical pad.

## Alternative considered: OS-level remapper

`input-remapper` or a small evdev→uinput script could turn F310 buttons into the recorder's keyboard keys (X→Space,
B→Enter, Y→R, A→S), with the profile auto-applied by device name. Not chosen as the primary route: the app already
has native pad support, and a remapper adds a daemon and only works where it runs. Kept as a fallback for apps
without pad support.

## Reproduce on your own pad

Public plugin [`input-device-mapper`](https://github.com/danielrosehill/Input-Device-Mapper-Plugin): `idmap.py info 046d:c21d`,
then `idmap.py press 046d:c21d --label <button>` once per button.

## SDL GameController mapping (Linux)

```
030000006d0400001dc2000014400000,Logitech F310 Gamepad (XInput),platform:Linux,a:b0,b:b1,x:b2,y:b3,leftshoulder:b4,rightshoulder:b5,back:b6,start:b7,guide:b8,leftstick:b9,rightstick:b10,lefttrigger:a2,righttrigger:a5,leftx:a0,lefty:a1,rightx:a3,righty:a4,dpup:h0.1,dpright:h0.2,dpdown:h0.4,dpleft:h0.8,
```

Derived, not read from SDL: the GUID is SDL's Linux layout (bus `0003`, vendor `046d`, product `c21d`, version `4014`,
each little-endian), and `bN`/`aN` follow SDL's evdev enumeration (buttons in ascending code order 304…318, axes
`ABS_X, ABS_Y, ABS_Z, ABS_RX, ABS_RY, ABS_RZ`). Buttons, triggers and D-pad match the press-verified table; the stick
axes were not press-tested. SDL already ships this pad in its built-in database, so this line is for reference.
