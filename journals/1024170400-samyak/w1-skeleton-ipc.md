# Week 1 : Click-Through Transparency and Mouse Event Forwarding in Frameless Overlay

# Click-Through Mouse Events

## Error:

When the overlay window was set to transparent and click-through using Electron's `win.setIgnoreMouseEvents(true)`, interaction with interactive UI elements (such as the top draggable pill and control buttons) was completely disabled. The mouse clicks passed through the entire window to underlying desktop applications, making the overlay unclickable and immovable.

> Error/Behavior: Interactive elements in renderer failed to register `mouseenter`, `mousedown`, or click events; window drag (`-webkit-app-region: drag`) became unresponsive when `setIgnoreMouseEvents(true)` was active globally.

## Relevant Context

In `main.js`, the transparent frameless window was created with mouse ignore flags enabled so background applications remain usable:

```javascript
// main.js - initial attempt
const mainWindow = new BrowserWindow({
  width: 480,
  height: 600,
  frame: false,
  transparent: true,
  alwaysOnTop: true,
  hasShadow: false,
  webPreferences: {
    preload: path.join(__dirname, 'preload.js'),
    contextIsolation: true,
    nodeIntegration: false
  }
});

mainWindow.setIgnoreMouseEvents(true);
```

In `renderer/renderer.js`, interactive regions attempted to capture events, but were never reached because the OS window layer dropped mouse hit testing before dispatching to Chromium.

## Key Observation

Electron's `BrowserWindow.setIgnoreMouseEvents` accepts an optional options object `{ forward: true }`. When `forward: true` is specified, OS-level mouse-move events are forwarded to the underlying web page. 

This allows the renderer process to track the mouse position using `mousemove` listeners over DOM elements. When the cursor enters an interactive element (e.g. the pill or buttons), the renderer can notify `main` via IPC to disable click-through (`setIgnoreMouseEvents(false)`). When the cursor leaves interactive elements, it signals `main` to re-enable click-through with `{ forward: true }`.

## Solution

1. In `main.js`, register IPC handlers to toggle mouse ignore state dynamically and ensure `{ forward: true }` is always passed when ignoring:

```javascript
// main.js - corrected implementation
ipcMain.on('set-ignore-mouse-events', (event, ignore, options) => {
  const win = BrowserWindow.fromWebContents(event.sender);
  if (!win) return;
  if (ignore) {
    win.setIgnoreMouseEvents(true, { forward: true });
  } else {
    win.setIgnoreMouseEvents(false);
  }
});
```

2. In `preload.js`, expose the method safely through `contextBridge`:

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('electronAPI', {
  setIgnoreMouseEvents: (ignore) => ipcRenderer.send('set-ignore-mouse-events', ignore)
});
```

3. In `renderer/renderer.js`, attach event listeners to interactive containers:

```javascript
// renderer/renderer.js
const interactiveArea = document.getElementById('interactive-container');

interactiveArea.addEventListener('mouseenter', () => {
  window.electronAPI.setIgnoreMouseEvents(false);
});

interactiveArea.addEventListener('mouseleave', () => {
  window.electronAPI.setIgnoreMouseEvents(true);
});
```

**Because**

The operating system window manager intercepts pointer events at the window hierarchy level before Chromium processes DOM hit testing. Calling `setIgnoreMouseEvents(true, { forward: true })` maintains OS mouse-move message forwarding into Chromium without capturing clicks, enabling dynamic hit-testing that toggles interactive focus solely over designated DOM bounding boxes.
