# Week 3 : High-DPI Desktop Capture Resolution and Bounding Box Extraction

# Desktop Screen Capture

## Error:

When invoking `desktopCapturer.getSources({ types: ['screen'] })` to capture the display for the vision LLM prompt, the resulting NativeImage thumbnail had severely degraded resolution (low pixel density, blurry code text) or exceeded Vision API payload size limits (causing HTTP 413 Payload Too Large / API timeout). On multi-monitor setups with scaling (e.g., 125% or 150% scaling on Windows), coordinates were distorted.

> Error from upstream vision API: `413 Request Entity Too Large` or OCR failure due to downsampled 150x150 default thumbnail size returned by Electron's desktop capturer.

## Relevant Context

In `src/screen.js`, the default `desktopCapturer` call was made without explicitly requesting high-resolution thumbnail dimensions:

```javascript
// src/screen.js - initial attempt
const { desktopCapturer, screen } = require('electron');

async function captureScreen() {
  const sources = await desktopCapturer.getSources({ types: ['screen'] });
  const primarySource = sources[0];
  // Default thumbnail size in Electron is 150x150, making text completely unreadable
  return primarySource.thumbnail.toDataURL();
}
```

When code text was sent at default thumbnail resolution, the LLM returned: *"The text in the image is too blurry to read."* Conversely, sending uncompressed 4K full raw PNG files caused payloads of 12MB+, exceeding the 4MB/10MB limits of OpenAI / Gemini vision endpoints.

## Key Observation

1. `desktopCapturer.getSources` requires `thumbnailSize` parameter configured to the screen's native display dimensions multiplied by `scaleFactor` to prevent downsampling.
2. The captured full-resolution bitmap must be resized maintaining aspect ratio (e.g., maximum width/height of 1920x1080) and converted to JPEG format with 80-85% compression quality. This reduces payload size by ~85% (down to ~300KB) with zero perceptible loss in code font legibility.
3. Multi-monitor setups require capturing the active screen where the user's cursor currently resides rather than blindly selecting `sources[0]`.

## Solution

In `src/screen.js`, determine the active display via `screen.getDisplayNearestPoint(screen.getCursorScreenPoint())`, request matching thumbnail dimensions, and compress using Electron's `nativeImage.resize()` and `toJPEG()`:

```javascript
// src/screen.js - corrected implementation
const { desktopCapturer, screen } = require('electron');

async function captureActiveScreen() {
  const cursorPoint = screen.getCursorScreenPoint();
  const currentDisplay = screen.getDisplayNearestPoint(cursorPoint);
  const { width, height } = currentDisplay.size;
  const scale = currentDisplay.scaleFactor;

  const sources = await desktopCapturer.getSources({
    types: ['screen'],
    thumbnailSize: {
      width: Math.floor(width * scale),
      height: Math.floor(height * scale)
    }
  });

  // Find source matching current display ID or default to first
  const targetSource = sources.find(s => s.display_id === `${currentDisplay.id}`) || sources[0];
  if (!targetSource || !targetSource.thumbnail) {
    throw new Error('Failed to capture active desktop display');
  }

  const rawImage = targetSource.thumbnail;
  const originalSize = rawImage.getSize();

  // Scale down proportionally if larger than 1920px max dimension
  const maxDim = 1920;
  let targetWidth = originalSize.width;
  let targetHeight = originalSize.height;

  if (targetWidth > maxDim || targetHeight > maxDim) {
    if (targetWidth > targetHeight) {
      targetHeight = Math.round((targetHeight * maxDim) / targetWidth);
      targetWidth = maxDim;
    } else {
      targetWidth = Math.round((targetWidth * maxDim) / targetHeight);
      targetHeight = maxDim;
    }
  }

  const resized = rawImage.resize({
    width: targetWidth,
    height: targetHeight,
    quality: 'better'
  });

  // Encode to JPEG at 82% quality for optimal size/clarity trade-off
  const jpegBuffer = resized.toJPEG(82);
  const base64Data = jpegBuffer.toString('base64');

  return {
    mimeType: 'image/jpeg',
    base64: base64Data,
    width: targetWidth,
    height: targetHeight
  };
}

module.exports = { captureActiveScreen };
```

**Because**

Vision LLMs (such as GPT-4o, Claude 3.5 Sonnet, and Gemini 1.5 Flash) process images through vision tokens calculated by tile bounding boxes. Supplying downsampled thumbnails destroys high-frequency edge details essential for reading monospace code syntax. Resizing to 1080p equivalent JPEG keeps transmission sizes well under 500KB while preserving crisp character edges for 100% OCR fidelity.
