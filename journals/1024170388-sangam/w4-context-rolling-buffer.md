# Week 4 : Assist Context Pipeline Rolling Buffer and Token Pruning Strategy

# Assist Flow and Rolling Context Management

## Error:

During repeated Assist flow interactions (`Ctrl+Enter` with user follow-up questions), the combined context window grew uncontrollably. Because each turn attached a full desktop screenshot alongside past prompt history, the model context limit was rapidly exceeded after only 3–4 interactions, throwing:
`400 InvalidRequestError: This model's maximum context length is 128000 tokens. However, your messages resulted in ... tokens.`
Furthermore, user follow-up questions took 15+ seconds to process due to accumulating redundant high-resolution images in previous turns.

> Error: `400 ContextWindowExceeded` and excessive latency caused by multiple historical images retained in the conversational message buffer.

## Relevant Context

In `src/context.js`, conversation turns were originally pushed into an unbounded array:

```javascript
// src/context.js - initial attempt
class ContextManager {
  constructor() {
    this.messages = [];
  }

  addInteraction(userPrompt, imageBase64, assistantResponse) {
    this.messages.push({
      role: 'user',
      content: [
        { type: 'text', text: userPrompt },
        { type: 'image_url', image_url: { url: `data:image/jpeg;base64,${imageBase64}` } }
      ]
    });
    this.messages.push({
      role: 'assistant',
      content: assistantResponse
    });
  }

  getPayload() {
    return this.messages;
  }
}
```

## Key Observation

1. Historical screenshots are rarely needed for follow-up text dialogue. Only the *latest* screenshot represents the student's current active editor screen state.
2. Earlier conversation turns only require the text transcript of what was previously asked and answered. Retaining image data in older message turns wastes hundreds of thousands of vision tokens and ballooning API costs.
3. The conversation buffer must enforce:
   - A maximum rolling memory limit (e.g. last 6 text turns).
   - An image-stripping step that strips base64 payloads from historical turns, keeping only the latest capture attached to the active prompt.
4. UI toggle for **Smart (e.g. GPT-4o / Claude 3.5 Sonnet)** vs. **Fast (e.g. GPT-4o-mini / Gemini 1.5 Flash)** lets the user choose between detailed diagnostic reasoning and near-instant answers.

## Solution

1. Refactor `src/context.js` to enforce image pruning and rolling limits:

```javascript
// src/context.js
class RollingContextManager {
  constructor(maxTurns = 6) {
    this.maxTurns = maxTurns;
    this.history = []; // Stores { role, textContent }
  }

  addTurn(userText, assistantText) {
    this.history.push({ role: 'user', content: userText });
    this.history.push({ role: 'assistant', content: assistantText });

    // Enforce rolling buffer limit (keeping last N messages)
    while (this.history.length > this.maxTurns * 2) {
      this.history.shift();
    }
  }

  buildMessages(currentPrompt, latestImageBase64 = null) {
    const messages = [
      {
        role: 'system',
        content: 'You are an AI desktop coding companion. Provide direct, succinct assistance without conversational filler.'
      }
    ];

    // Include historical dialogue as pure text turns
    for (const msg of this.history) {
      messages.push({
        role: msg.role,
        content: msg.content
      });
    }

    // Build the latest active user turn with optional current screenshot
    if (latestImageBase64) {
      messages.push({
        role: 'user',
        content: [
          { type: 'text', text: currentPrompt },
          {
            type: 'image_url',
            image_url: { url: `data:image/jpeg;base64,${latestImageBase64}` }
          }
        ]
      });
    } else {
      messages.push({
        role: 'user',
        content: currentPrompt
      });
    }

    return messages;
  }

  clear() {
    this.history = [];
  }
}

module.exports = new RollingContextManager();
```

2. Add Smart/Fast model selection handling in renderer and IPC:

```javascript
// renderer/renderer.js
const modelToggle = document.getElementById('smart-fast-toggle');
modelToggle.addEventListener('change', (e) => {
  const isSmart = e.target.checked;
  window.electronAPI.setModelTier(isSmart ? 'smart' : 'fast');
});
```

**Because**

Vision tokens account for over 80% of both cost and processing latency in multimodal LLM requests. Pruning historical base64 image data while preserving text turn history maintains contextual conversational coherence while keeping prompt payloads lightweight, predictable, and strictly within API token quotas.
