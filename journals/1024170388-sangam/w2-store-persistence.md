# Week 2 : Atomic JSON Store Persistence and Secure Settings IPC Roundtrip

# Settings Store and API Key Persistence

## Error:

When updating settings or API keys in the settings modal, rapid saves or app restarts occasionally resulted in empty `cue-data.json` files or parse errors:
`SyntaxError: Unexpected end of JSON input`. In addition, API keys were initially stored in plain text without proper file permission restrictions, and changes in the renderer were not immediately reflected in the main process LLM client without restarting the application.

> Error: `SyntaxError: Unexpected end of JSON input at JSON.parse (<anonymous>)` after sudden application close or concurrent read/write.

## Relevant Context

The initial store module wrote directly to disk using non-atomic `fs.writeFile`:

```javascript
// src/store.js - initial attempt
const fs = require('fs');
const path = require('path');

function saveSettings(data) {
  const filePath = path.join(app.getPath('userData'), 'cue-data.json');
  fs.writeFileSync(filePath, JSON.stringify(data, null, 2));
}
```

If the user closed the application or if an asynchronous save was interrupted midway, the file was truncated to 0 bytes before writing completed, corrupting user preferences and stored credentials.

## Key Observation

1. Writing configuration directly to the target file is dangerous because file truncation occurs before data buffer flushing completes. Writing to a temporary sibling file (`cue-data.json.tmp`) followed by an atomic filesystem rename (`fs.renameSync`) ensures the original file remains intact until the new file is completely flushed.
2. In-memory settings caching must be synchronized between Main and Renderer via two-way IPC handlers (`ipcMain.handle('get-settings')` and `ipcMain.handle('save-settings')`).

## Solution

1. Rewrite `src/store.js` using atomic file writing with safe defaults and restrictive permissions:

```javascript
// src/store.js
const fs = require('fs');
const path = require('path');
const { app } = require('electron');

const DEFAULT_SETTINGS = {
  activeProvider: 'openai',
  providers: {
    openai: { apiKey: '', model: 'gpt-4o' },
    anthropic: { apiKey: '', model: 'claude-3-5-sonnet-20240620' },
    gemini: { apiKey: '', model: 'gemini-1.5-pro-latest' }
  },
  preferences: {
    smartModel: true,
    fontSize: 13
  }
};

class SettingsStore {
  constructor() {
    this.filePath = path.join(app.getPath('userData'), 'cue-data.json');
    this.tempPath = `${this.filePath}.tmp`;
    this.cache = this.load();
  }

  load() {
    try {
      if (fs.existsSync(this.filePath)) {
        const raw = fs.readFileSync(this.filePath, 'utf-8');
        return { ...DEFAULT_SETTINGS, ...JSON.parse(raw) };
      }
    } catch (err) {
      console.error('[Store] Failed to parse existing settings file. Resetting to defaults:', err);
    }
    return { ...DEFAULT_SETTINGS };
  }

  save(newSettings) {
    this.cache = { ...this.cache, ...newSettings };
    const serialized = JSON.stringify(this.cache, null, 2);

    // Atomic write: write to temp file first, then atomically rename
    fs.writeFileSync(this.tempPath, serialized, { encoding: 'utf-8', mode: 0o600 });
    fs.renameSync(this.tempPath, this.filePath);
    return this.cache;
  }

  get(key) {
    return key ? this.cache[key] : this.cache;
  }
}

module.exports = new SettingsStore();
```

2. Wire IPC channels in `main.js`:

```javascript
// main.js
const store = require('./src/store');

ipcMain.handle('settings:get', () => store.get());
ipcMain.handle('settings:save', (event, updated) => {
  const saved = store.save(updated);
  return saved;
});
```

3. In `src/llm.js`, implement non-streaming connectivity test to verify key validity on save:

```javascript
// src/llm.js
async function testConnection(provider, apiKey) {
  const response = await fetch('https://api.openai.com/v1/models', {
    headers: { 'Authorization': `Bearer ${apiKey}` }
  });
  return response.ok;
}
```

**Because**

Atomic file swaps (`fs.rename`) are guaranteed by POSIX and Windows NTFS filesystems to be indivisible metadata pointer operations. Even if power is lost or the process terminates during a write, the file system will either show the previous valid version or the new complete version, completely preventing configuration zero-byte corruption.
