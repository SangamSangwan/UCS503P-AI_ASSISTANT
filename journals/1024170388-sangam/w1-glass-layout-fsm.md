# Week 1 : Overlay Glassmorphism Layout and Renderer State Machine Transitions

# Glassmorphism Layout and UI State Transitions

## Error:

When rendering the floating glassmorphic overlay panel over transparent backgrounds in Chromium/Electron, severe visual artifacts occurred:
1. `backdrop-filter: blur(...)` caused flickering and opaque black box rendering whenever the underlying OS desktop refreshed behind transparent `BrowserWindow`.
2. Rapid user input during asynchronous stub IPC calls triggered race conditions where the UI remained stuck in an inconsistent `listening` state without returning to `idle`.

> Observed Visual Glitch: Translucent rounded glass corners became jaggy with black halos, and state machine transitions allowed duplicate trigger submissions while a request was in flight.

## Relevant Context

In `renderer/styles.css`, glassmorphism was styled using standard web properties:

```css
/* renderer/styles.css - initial */
body {
  background: transparent;
}
.glass-panel {
  background: rgba(20, 20, 25, 0.6);
  backdrop-filter: blur(16px); /* Glitched on transparent Electron windows */
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}
```

In `renderer/renderer.js`, state flags were managed as informal loose booleans (`let isBusy = false; let isListening = false;`) without an explicit state machine, leading to illegal state combinations when rapid hotkeys or button clicks were fired.

## Key Observation

1. Hardware-accelerated `backdrop-filter` in Chromium on transparent Windows/DWM surfaces often produces compositing artifacts because the background buffer is undefined. Using multi-layered subtle translucent backgrounds (`rgba(15, 18, 25, 0.82)`) with a fine border gradient provides clean glass aesthetics across all GPU drivers without compositing glitches.
2. The UI state must be modeled as a strict Finite State Machine (`IDLE` -> `CAPTURING` -> `RESPONDING` -> `IDLE`, with an `ERROR` state that safely reverts to `IDLE`). Transitions must lock out invalid user events.

## Solution

1. In `renderer/styles.css`, refine CSS custom properties and surface layering:

```css
:root {
  --bg-surface: rgba(18, 20, 29, 0.88);
  --border-glass: rgba(255, 255, 255, 0.08);
  --text-primary: #f3f4f6;
  --accent-cyan: #38bdf8;
}

body {
  margin: 0;
  padding: 8px;
  background: transparent;
  user-select: none;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
}

.glass-pill {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 14px;
  background: var(--bg-surface);
  border: 1px solid var(--border-glass);
  border-radius: 9999px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.35);
  -webkit-app-region: drag; /* Draggable pill handle */
}
```

2. In `renderer/renderer.js`, implement a strict state machine controller:

```javascript
// renderer/renderer.js
const States = {
  IDLE: 'IDLE',
  LISTENING: 'LISTENING',
  RESPONDING: 'RESPONDING',
  ERROR: 'ERROR'
};

class UIStateMachine {
  constructor() {
    this.currentState = States.IDLE;
    this.statusIndicator = document.getElementById('status-indicator');
    this.actionInput = document.getElementById('action-input');
    this.renderState();
  }

  transition(newState, payload = {}) {
    const validTransitions = {
      [States.IDLE]: [States.LISTENING, States.RESPONDING],
      [States.LISTENING]: [States.RESPONDING, States.IDLE, States.ERROR],
      [States.RESPONDING]: [States.IDLE, States.ERROR],
      [States.ERROR]: [States.IDLE]
    };

    if (!validTransitions[this.currentState].includes(newState)) {
      console.warn(`[FSM] Rejected invalid transition: ${this.currentState} -> ${newState}`);
      return false;
    }

    console.log(`[FSM] State: ${this.currentState} -> ${newState}`);
    this.currentState = newState;
    this.renderState(payload);
    return true;
  }

  renderState(payload) {
    document.body.dataset.state = this.currentState;
    if (this.currentState === States.IDLE) {
      this.statusIndicator.textContent = 'Ready';
      this.actionInput.disabled = false;
    } else if (this.currentState === States.RESPONDING) {
      this.statusIndicator.textContent = 'Generating...';
      this.actionInput.disabled = true;
    }
  }
}

const fsm = new UIStateMachine();
```

**Because**

A formal finite state machine guarantees that user actions cannot execute concurrently during IPC dispatch or async generation cycles. Explicitly defining allowable transitions eliminates re-entrancy bugs, and avoiding buggy GPU blur shaders ensures deterministic styling across varied graphics hardware.
