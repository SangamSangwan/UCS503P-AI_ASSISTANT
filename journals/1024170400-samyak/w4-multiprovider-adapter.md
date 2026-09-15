# Week 4 : Multi-Provider Base URL Shaping and Authentication Quirk Normalization

# Multi-Provider Protocol Normalization

## Error:

When switching providers between standard OpenAI, Azure OpenAI, Google Gemini (via OpenAI-compatibility mode), and custom local endpoints (Ollama / vLLM / LM Studio), requests failed with HTTP 404 Not Found, 401 Unauthorized, or URL malformation errors.

> Examples:
> 1. Azure OpenAI: `404 Resource Not Found` due to missing `/openai/deployments/{model}/chat/completions?api-version=2024-02-15-preview` route structure.
> 2. Local endpoints (Ollama): `401 Unauthorized` or connection refusal when mandatory `Authorization: Bearer <key>` header was included with blank dummy keys.
> 3. Network hang with no user feedback when local inference servers stalled or timed out.

## Relevant Context

In `src/openai-compatible.js`, requests originally targeted a fixed standard OpenAI endpoint URL `/v1/chat/completions`:

```javascript
// Initial naive implementation
const endpoint = `${config.baseUrl}/chat/completions`;
const response = await fetch(endpoint, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${config.apiKey}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(payload)
});
```

Different providers differ significantly in route hierarchies, header conventions, and authentication schemes:
- **Azure OpenAI**: Requires `api-key` header instead of `Authorization: Bearer`, with model names parameterized directly in the path (`/openai/deployments/{deploymentName}`).
- **Custom / Local (Ollama/LM Studio)**: Base URL often supplied with or without `/v1` trailing slash; crashes if bearer token header is malformed.
- **Gemini**: Requires specific base URL `https://generativelanguage.googleapis.com/v1beta/openai/`.

## Key Observation

The client adapter must normalize user-provided configuration objects into standard HTTP dispatch parameters by detecting the provider dialect, stripping redundant slashes, constructing vendor-specific headers, and attaching an `AbortSignal.timeout` controller.

## Solution

Refactor `src/openai-compatible.js` into an adapter that normalizes URLs, headers, and error states:

```javascript
// src/openai-compatible.js
class ProviderAdapter {
  static resolveRequestConfig(providerConfig, payload) {
    const { provider, baseUrl, apiKey, model, apiVersion } = providerConfig;
    let url = '';
    const headers = { 'Content-Type': 'application/json' };

    switch (provider) {
      case 'azure': {
        const cleanBase = baseUrl.replace(/\/+$/, '');
        const version = apiVersion || '2024-02-15-preview';
        url = `${cleanBase}/openai/deployments/${model}/chat/completions?api-version=${version}`;
        headers['api-key'] = apiKey;
        break;
      }
      case 'gemini': {
        const cleanBase = (baseUrl || 'https://generativelanguage.googleapis.com/v1beta/openai').replace(/\/+$/, '');
        url = `${cleanBase}/chat/completions`;
        headers['Authorization'] = `Bearer ${apiKey}`;
        break;
      }
      case 'custom': {
        const cleanBase = baseUrl.replace(/\/+$/, '');
        url = cleanBase.endsWith('/v1') ? `${cleanBase}/chat/completions` : `${cleanBase}/v1/chat/completions`;
        if (apiKey && apiKey.trim() !== '') {
          headers['Authorization'] = `Bearer ${apiKey}`;
        }
        break;
      }
      case 'openai':
      default: {
        const cleanBase = (baseUrl || 'https://api.openai.com/v1').replace(/\/+$/, '');
        url = `${cleanBase}/chat/completions`;
        headers['Authorization'] = `Bearer ${apiKey}`;
        break;
      }
    }

    return { url, headers };
  }

  static async sendChatRequest(providerConfig, payload, { timeoutMs = 25000 } = {}) {
    const { url, headers } = ProviderAdapter.resolveRequestConfig(providerConfig, payload);
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeoutMs);

    try {
      const response = await fetch(url, {
        method: 'POST',
        headers,
        body: JSON.stringify(payload),
        signal: controller.signal
      });

      if (!response.ok) {
        const errText = await response.text();
        if (response.status === 401 || response.status === 403) {
          throw new Error(`Authentication failed (${response.status}): Please verify your API key.`);
        }
        throw new Error(`HTTP ${response.status} from ${providerConfig.provider}: ${errText}`);
      }

      return response;
    } catch (err) {
      if (err.name === 'AbortError') {
        throw new Error(`Request timed out after ${timeoutMs / 1000}s. Check your provider connection.`);
      }
      throw err;
    } finally {
      clearTimeout(timer);
    }
  }
}

module.exports = { ProviderAdapter };
```

**Because**

Different cloud AI platforms and local model runners diverge on path formatting and header names while preserving the JSON body schema of OpenAI's `/chat/completions`. Centralizing URL resolution and header injection in an adapter decouples the UI and prompt pipeline from low-level network and provider intricacies.
