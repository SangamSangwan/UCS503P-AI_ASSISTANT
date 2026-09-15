# Week 2 : Window Content Protection and Capture Exclusion via Win32 API in Electron

# Screen Share Invisibility

## Error:

When sharing the screen during testing over Discord, Zoom, or Google Meet, Electron's built-in `win.setContentProtection(true)` successfully blacked out the window on some platforms, but on Windows 10/11 with modern Desktop Duplication API (DXGI / Windows Graphics Capture), the overlay remained fully visible or rendered a solid black rectangle rather than being completely transparent/invisible in the capture stream.

> Observed Behavior: The overlay contents or an opaque black boundary box were visible to other participants in screen sharing tools, defeating the requirement for a subtle, non-intrusive personal overlay assistant.

## Relevant Context

Electron provides a cross-platform API `mainWindow.setContentProtection(true)`:

```javascript
// main.js - initial attempt
mainWindow.setContentProtection(true);
```

While on macOS `setContentProtection(true)` sets `NSWindowSharingNone`, on Windows versions prior to build 2004 it used `WDA_MONITOR` which produces a blacked-out window. In Windows 10 version 2004 (build 19041) and Windows 11, Microsoft introduced `WDA_EXCLUDEFROMCAPTURE` (`0x00000011`) for `SetWindowDisplayAffinity`, which removes the window completely from capture instead of showing a black rectangle.

## Key Observation

Electron's internal implementation of `setContentProtection` does not consistently apply `WDA_EXCLUDEFROMCAPTURE` across all Electron versions without native platform invocation or newer flags. Furthermore, during development and debugging, developers need an escape hatch to inspect and record the window behavior for integration testing.

## Solution

1. Integrate native Windows user32 call using `koffi` (or `ffi-napi`) to set `SetWindowDisplayAffinity(hwnd, WDA_EXCLUDEFROMCAPTURE)` on Windows, falling back to `setContentProtection(true)` on macOS / Linux.
2. Add a debug escape hatch `CUE_NO_PROTECT=1` to allow developer screencasting and automated screenshot tests when needed.

```javascript
// src/protection.js
const { BrowserWindow } = require('electron');
const os = require('os');

function applyContentProtection(window) {
  if (process.env.CUE_NO_PROTECT === '1') {
    console.log('[Protection] Debug bypass enabled via CUE_NO_PROTECT=1');
    return;
  }

  // Cross-platform base protection
  window.setContentProtection(true);

  if (process.platform === 'win32') {
    try {
      const koffi = require('koffi');
      const user32 = koffi.load('user32.dll');
      const SetWindowDisplayAffinity = user32.func(
        'int __stdcall SetWindowDisplayAffinity(void *hWnd, uint32 dwAffinity)'
      );

      const hwnd = window.getNativeWindowHandle();
      const WDA_EXCLUDEFROMCAPTURE = 0x00000011; // Windows 10 2004+ exclusion flag
      
      const result = SetWindowDisplayAffinity(hwnd, WDA_EXCLUDEFROMCAPTURE);
      console.log(`[Protection] SetWindowDisplayAffinity WDA_EXCLUDEFROMCAPTURE result: ${result}`);
    } catch (err) {
      console.warn('[Protection] Native Win32 affinity fallback failed:', err.message);
    }
  }
}

module.exports = { applyContentProtection };
```

3. Register global hotkeys safely in `src/shortcuts.js` with registration error checks:

```javascript
// src/shortcuts.js
const { globalShortcut } = require('electron');

function registerShortcuts(handlers) {
  const isMac = process.platform === 'darwin';
  const solveKey = isMac ? 'Command+Alt+S' : 'Control+Alt+S';
  const assistKey = isMac ? 'Command+Enter' : 'Control+Enter';

  const ret = globalShortcut.register(solveKey, () => {
    handlers.onSolveCoding();
  });

  if (!ret) {
    console.error(`[Shortcuts] Registration failed for ${solveKey}`);
  }
}

module.exports = { registerShortcuts };
```

**Because**

The Windows Desktop Window Manager (DWM) composition engine checks the window display affinity attribute during capture session enumeration. Supplying `WDA_EXCLUDEFROMCAPTURE` instructs the compositor to omit the window's visual surface entirely from screen capture streams (DirectX GI Output Duplication and Windows.Graphics.Capture) while keeping it rendered on the physical display monitor for the local user.
