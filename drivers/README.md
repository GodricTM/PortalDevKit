# Portal Windows USB driver notes

`android_winusb.inf` is a patched Google USB Driver INF with Meta Portal ADB IDs added:

- `VID_2EC6&PID_1800`
- `VID_2EC6&PID_1801`

The INF is provided so Portal owners can bind the device to WinUSB/ADB on Windows. It is not a full
driver package by itself; use it with Google's USB Driver package from the Android SDK, or install
WinUSB through Device Manager / Zadig as described in [03 - Build & sideload](../03-build-and-sideload.md).

Editing an INF invalidates the original catalog signature. If Windows blocks installation, either use
the temporary "Disable driver signature enforcement" startup option or use Zadig to bind WinUSB.
