# Week 3 : Server-Sent Events (SSE) Stream Chunking and Incremental Markdown Rendering

# Real-Time LLM Token Streaming

## Error:

When converting the vision LLM request to streaming via Server-Sent Events (`stream: true`), two critical failures occurred:
1. Multi-byte UTF-8 sequences (such as emoji, smart quotes, or math symbols) and JSON payloads were randomly sliced across chunk boundaries, throwing:
   `SyntaxError: Unexpected token in JSON at position ...`
2. Incrementally updating the DOM with `innerHTML += token` broke code block syntax trees midway through generation, resulting in unclosed `pre`/`code` HTML tags that corrupted the overlay styling.

> Error: Malformed JSON chunk in SSE stream and broken DOM layout during code block streaming.

## Relevant Context

The initial stream reader assumed each `reader.read()` chunk contained exactly one complete `data: {...}` line:

```javascript
// src/llm.js - initial naive streaming reader
const reader = response.body.getReader();
const decoder = new TextDecoder('utf-8');

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  const text = decoder.decode(value);
  // Fails when a single SSE message spans multiple chunks or multiple messages arrive in one chunk
  if (text.startsWith('data: ') && !text.includes('[DONE]')) {
    const json = JSON.parse(text.replace('data: ', ''));
    onToken(json.choices[0].delta?.content || '');
  }
}
```

## Key Observation

1. TCP and HTTP chunking is stream-oriented, not message-oriented. An SSE message `data: {...}\n\n` may arrive split across two read iterations or coalesced together with 3 other messages in a single packet buffer.
2. A line-buffering accumulator must hold partial line segments across chunk boundaries until a terminating `\n` is encountered.
3. For renderer display, a lightweight streaming markdown processor (or throttled batch renderer with an unclosed markdown tag reconciler) must be used instead of raw `innerHTML` appending.

## Solution

1. Implement an SSE buffer parser in `src/llm.js`:

```javascript
// src/llm.js
async function* streamChatCompletions(response) {
  const reader = response.body.getReader();
  const decoder = new TextDecoder('utf-8');
  let buffer = '';

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      // Retain the last incomplete line in the buffer
      buffer = lines.pop() || '';

      for (const rawLine of lines) {
        const line = rawLine.trim();
        if (!line || line.startsWith(':')) continue; // Ignore empty lines and SSE comments

        if (line === 'data: [DONE]') {
          return;
        }

        if (line.startsWith('data: ')) {
          const jsonStr = line.slice(6);
          try {
            const parsed = JSON.parse(jsonStr);
            const token = parsed.choices?.[0]?.delta?.content;
            if (token) {
              yield token;
            }
          } catch (err) {
            console.warn('[SSE] Skipped unparseable partial chunk:', jsonStr);
          }
        }
      }
    }
  } finally {
    reader.releaseLock();
  }
}
```

2. In `src/prompts.js`, define structured prompt instructions for concise hints rather than full code dump:

```javascript
// src/prompts.js
const PROMPTS = {
  SOLVE_CODING: `You are an expert pair programmer viewing the student's screen.
Identify the problem, syntax error, or failing test in the image.
Provide:
1. A concise, 2-line diagnostic of why it fails.
2. The specific corrected snippet.
Keep explanations direct, actionable, and formatted in clean markdown.`
};

module.exports = { PROMPTS };
```

3. In `renderer/renderer.js`, render streaming tokens safely using text nodes and a throttled markdown formatter.

**Because**

Stream decoders must maintain state across arbitrary network packet fragmentations. Buffering unparsed string tails until full newline delimiters (`\n`) appear guarantees that every JSON string fed into `JSON.parse` is structurally whole, preventing parse crashes regardless of network latency or transmission chunk sizes.
