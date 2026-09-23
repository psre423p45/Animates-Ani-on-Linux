**TL;DR:** Animates (the Windows-only AI companion) runs on Linux with Heroic + GE-Proton, a fixed WebView2 runtime, 4 one-byte-ish binary patches, a tiny push-to-talk bridge and a custom KWin effect that makes the black background transparent. Login, 3D companion, transparency, dragging and push-to-talk all work. No downloads here, just the recipe, so you can write your own tools.


**Tested on:** two Arch machines, Animates 1.0.9, Heroic (Flatpak) 2.22.3, GE-Proton11-7, KDE Plasma 6.7.5 on **X11**, AMD RX 7800 XT (RADV), PipeWire.

Animates is a Unity 6000.3 app with WebView2 windows, installed with Velopack. Here's everything it took.

## 1. Install

- Install Heroic and GE-Proton11-7.
- `animates.ai/win` `https://downloads.animates.ai/windows/win/Animates.exe`
- Run the installer with Proton in a fresh prefix:

      STEAM_COMPAT_DATA_PATH=<prefix> STEAM_COMPAT_CLIENT_INSTALL_PATH=~/.steam/steam <GE-Proton dir>/proton run Animates.exe

- It installs to `<prefix>/pfx/drive_c/users/steamuser/AppData/Local/AnimateApp/current/Animates.exe`. When it finishes, Velopack launches the app and it hangs with no window: kill it with `WINEPREFIX=<prefix>/pfx <GE-Proton dir>/files/bin/wineserver -k`.
- Add it to Heroic as a sideloaded Windows app pointing to that `.exe`, same prefix, GE-Proton11-7.

## 2. Fixed-version WebView2

The Evergreen WebView2 runtime doesn't work, but the fixed-version runtime **109.0.1518.78** does. It's on NuGet as `WebView2.Runtime.X64`:

    https://api.nuget.org/v3-flatcontainer/webview2.runtime.x64/109.0.1518.78/webview2.runtime.x64.109.0.1518.78.nupkg

It's a zip. Copy the contents of `contentFiles/any/any/WebView2/` (the folder with `msedgewebview2.exe`) to `<prefix>/pfx/drive_c/webview2fixed/`, then set these environment variables for the game in Heroic (Settings -> Advanced):

- `WEBVIEW2_BROWSER_EXECUTABLE_FOLDER` = `C:\webview2fixed`
- `WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS` = `--no-sandbox --remote-debugging-port=9222 --remote-allow-origins=*`

The second one opens a Chrome DevTools Protocol (CDP) port, used for push-to-talk. With the game running, `curl -s 127.0.0.1:9222/json` should list its pages.

Tip: Heroic keeps its config in memory, so fully close it before editing its JSON files by hand.

## 3. Four binary patches

Files are in `.../AnimateApp/current/Animates_Data/`. Offsets are **for 1.0.9 only**; check the MD5 first and back up the files.

| # | File (MD5 of 1.0.9 original) | Offset | Change | Fixes |
|---|---|---|---|---|
| 1 | `Managed/Assembly-CSharp.dll` (`827d426bc1421dbacca467daaed1fdcf`) | 472266 | `17` -> `16` | Black UI |
| 2 | `Plugins/x86_64/WebViewHost.dll` (`9a382b1500b4a05fb4afbc8c8d6431f7`) | 55728 | `40 57 48` -> `31 c0 c3` | Clicks go through the UI |
| 3 | `Managed/NAudio.Wasapi.dll` (`06df328b05d90a91e3f353577ef8a126`) | 15350 | `02` -> `2a` | Crash after login |
| 4 | `Managed/Assembly-CSharp.dll` (same file) | 268016 | `02` -> `2a` | Companion can't be dragged |

What each one does:

1. **Black UI.** The WebView tries to use DirectComposition, and Wine doesn't implement `DCompositionCreateDevice2` (`Player.log` shows hresult `-2147467263`, E_NOTIMPL). The byte is the `ldc.i4.1` before `stfld composition`; making it `ldc.i4.0` sets `CreateOptions.composition = false` and the UI renders.
2. **Clicks go straight through the web UI.** The native export `WVH_SetClickThrough` becomes `xor eax,eax; ret`.
3. **Crash right after login.** `AudioSessionManager.UnregisterNotifications` in NAudio makes a COM call Wine doesn't implement. First IL byte -> `ret` (0x2A).
4. **Companion can't be dragged.** After onboarding, the game reads the alpha of the pixel under the mouse and makes the window click-through (`WS_EX_TRANSPARENT`) when it's transparent. Under Wine that read always says "transparent", so every click passes through. First IL byte of `WindowsAPI.SetClickThrough` -> `ret`.

**Velopack auto-updates overwrite these files, so re-apply the patches after every game update.**

**For a different game version**, find the offsets yourself:

- Decompile the managed DLLs with `ilspycmd` (needs .NET 8, can be installed to a temp folder without root) and grep for `composition`, `SetClickThrough`, `UnregisterNotifications`.
- Get the file offset of a method's IL body with Python + `dnfile`: find the method in `TypeDef.MethodList`, convert its RVA with `pe.get_offset_from_rva()`. If the header's low 2 bits are `3` it's a fat header and the code starts at `offset + (byte[offset+1] >> 4) * 4`; otherwise it's tiny and the code starts at `offset + 1`. Writing `0x2A` there turns a `void` method into a no-op.
- For `WebViewHost.dll` (native), look up the RVA of the `WVH_SetClickThrough` export, convert it to a file offset, write `31 c0 c3`.

## 4. Push-to-talk

Under Wine the game's windows never get keyboard focus, so Unity's `GetAsyncKeyState` never sees the PTT key (Alt/AltGr). The fix is a small daemon (I wrote mine in Python with `python-xlib` + `websocket-client`):

- Listen for Alt_L / Alt_R / ISO_Level3_Shift system-wide via **XInput2 raw key events** on the root window. It's not a grab, so Alt+Tab keeps working.
- On press (after ~180 ms held, and cancelled if another key is pressed so Alt+Tab doesn't trigger it), fetch `http://127.0.0.1:9222/json`, pick the page titled `Context Menu` or `Agent Bridge`, open its `webSocketDebuggerUrl` and send `Runtime.evaluate` with `window.animateHost.requestOpenInput('listening')`.
- On release, do the same on `Input` / `Context Menu` / `Agent Bridge` with `window.animateHost.closeInput()`.
- Run it from autostart. During onboarding those pages don't exist yet, which is expected.

## 5. The black background (the hard part)

On Windows the companion floats on your desktop thanks to DWM: Unity uses `DwmExtendFrameIntoClientArea` with -1 margins and the WebView uses DirectComposition. Wine composites neither, so every window of the game has a solid black background.

**What didn't work:**

- Patching the game to use a color key (`LWA_COLORKEY`): no effect, because Unity renders through DXVK/Vulkan, not GDI.
- Cutting the window with XShape from outside: X stops updating the cut-out areas, so the character vanishes when it moves into them.
- Matching windows by PID: Wine reports the PID of the umu container, not the real process.

**What worked:** a custom KWin effect, a C++ subclass of `KWin::OffscreenEffect`:

- On `windowAdded` (and for existing windows in `stackingOrder()` at startup), match X11 windows whose class contains `steam_app_default` and whose caption is `animation` (the Unity window) or empty (the WebViews).
- For those, call `redirect(w)` + `setShader(w, shader)`, and in `prePaintWindow` call `data.setTranslucent()`.
- The shader is loaded with `ShaderManager::instance()-> generateShaderFromFile(ShaderTrait::MapTexture, {}, path)` and does:

      vec4 tex = texture(sampler, texcoord0);
      float m = max(tex.r, max(tex.g, tex.b));
      tex.a = clamp(m / 0.05, 0.0, 1.0);
      tex = sourceEncodingToNitsInDestinationColorspace(tex);
      tex *= modulation;
      fragColor = nitsToDestinationEncoding(tex);

  (`#include "colormanagement.glsl"` at the top.) What rendered over black is effectively premultiplied already, so setting alpha is enough.

Why it looks good:

- The game draws its background as exact black (0,0,0), so that becomes fully transparent.
- Near-black edge pixels become semi-transparent, so edges stay smooth instead of jagged.
- The character has almost no pure-black pixels, so dark clothing doesn't get holes.

Build notes (Plasma 6, X11):

- Link against `KWinX11::kwin` (`find_package(KWinX11)`), use `kcoreaddons_add_plugin(... INSTALL_NAMESPACE "kwin-x11/effects/plugins")`, C++23, and set `QT_MAJOR_VERSION 6` before ECM. Qt Widgets/DBus/Qml/Quick are needed too.
- Ship two shader variants, `name.frag` (GLSL 1.10 style) and `name_core.frag` (`#version 140`): KWin in core profile looks for the `_core` one.
- Put `"EnabledByDefault": true` in `metadata.json` so it loads at login. Install the `.so` to `/usr/lib/qt6/plugins/kwin-x11/effects/plugins/`.

**Warnings:**

- Only works on `kwin_x11`; recompile after KWin/Plasma updates.
- After reinstalling the `.so` you need `kwin_x11 --replace`; unloading/loading the effect over D-Bus doesn't reload the library.
- Other non-Steam games launched through umu/Heroic share that window class, so pure black in their untitled windows turns transparent too.

## Tips

- Always launch the game from Heroic. Double-clicking the `.exe` leaves duplicate instances hanging.
- If the game closes right after login, launch it again; the session is already saved.
- During onboarding the companion is click-through on purpose, same as on Windows.
- Useful logs, inside the prefix: `AppData/LocalLow/Animation Inc_/animation/Player.log` (Unity) and `AppData/Local/Temp/WebViewHost.console.log` (WebView).

