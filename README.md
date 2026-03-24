# WebCodecs VideoDecoder Analysis

[![Netlify Status](https://api.netlify.com/api/v1/badges/YOUR_BADGE_ID/deploy-status)](https://app.netlify.com/sites/YOUR_SITE/deploys)

A security research tool demonstrating vulnerability patterns in WebKit's WebCodecs implementation through controlled fuzzing of the VideoDecoder API.

**🔴 WARNING: This tool intentionally triggers browser crashes for security research purposes. Use only in controlled environments.**

---

## 🚀 Quick Deploy

[![Deploy to Netlify](https://www.netlify.com/img/deploy/button.svg)](https://app.netlify.com/start/deploy?repository=https://github.com/yourusername/webcodecs-decoder-poc)

Or try it instantly: **[Live Demo](https://webcodecs-decoder-poc.netlify.app)** (replace with your URL)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Attack Techniques](#attack-techniques)
- [Browser Support](#browser-support)
- [Deployment](#deployment)
- [Technical Details](#technical-details)
- [Troubleshooting](#troubleshooting)

---

## 🔍 Overview

This tool demonstrates how type confusion and malformed input handling in WebKit's WebCodecs implementation can lead to:

- **Null pointer dereferences** in the media decoder pipeline
- **Use-after-free** conditions in frame reference management
- **Out-of-bounds reads** in bitstream parsing

### Why This Matters

WebCodecs provides low-level media access to web applications, running in privileged contexts (often separate GPU/media processes). Understanding its vulnerabilities helps:

- Security researchers analyze browser attack surfaces
- Developers understand secure coding patterns
- Users appreciate the complexity of modern web security

---

## 🏃 Quick Start

### Local Development

```bash
# Clone the repository
git clone https://github.com/yourusername/webcodecs-decoder-poc.git
cd webcodecs-decoder-poc

# Serve locally (any static server works)
npx serve . -p 3000

# Or with Python
python3 -m http.server 3000
```

Then open `http://localhost:3000`

### One-Line Test

```bash
npx serve https://github.com/yourusername/webcodecs-decoder-poc
```

---

## 💥 Attack Techniques

The tool provides 4 test scenarios:

### 1. Empty Data Injection
Sends zero-byte `EncodedVideoChunk` payloads to trigger null pointer dereferences in VP8/WebM parsing code.

```javascript
new EncodedVideoChunk({
    type: "key",
    timestamp: 1000000,
    data: new Uint8Array(0)  // Empty!
})
```

### 2. Type Confusion Attack
Mislabels delta frames (P-frames) as keyframes (I-frames), causing the decoder to access null reference frame pointers.

```javascript
// Delta frame data labeled as keyframe
new EncodedVideoChunk({
    type: "key",  // Wrong! Should be "delta"
    timestamp: chunk.timestamp,
    data: deltaFrameData
})
```

### 3. Truncated Data Attack
Sends partial frame data to trigger out-of-bounds reads during header parsing.

```javascript
const truncated = fullData.slice(0, 10);  // Only 10 bytes
```

### 4. Timestamp Manipulation
Uses negative or non-monotonic timestamps to confuse frame reordering logic.

```javascript
timestamp: -999999999  // Negative timestamp
```

---

## 🌐 Browser Support

| Browser | Version | Support | Notes |
|---------|---------|---------|-------|
| Chrome | 94+ | ✅ Full | Best compatibility |
| Edge | 94+ | ✅ Full | Chromium-based |
| Safari | 16.4+ | ✅ Full | WebCodecs enabled by default |
| Opera | 80+ | ✅ Full | Chromium-based |
| Firefox | - | ❌ None | Not yet implemented |

### Check Your Browser

Open the [live demo](https://your-site.netlify.app) - it will show a "Ready" or "Not Supported" badge based on your browser's capabilities.

---

## 🚀 Deployment

### Netlify (Recommended)

#### Option 1: One-Click Deploy
Click the "Deploy to Netlify" button above.

#### Option 2: Git Integration

```bash
# Push to GitHub
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/yourusername/webcodecs-decoder-poc.git
git push -u origin main

# Then connect repo at https://app.netlify.com
```

#### Option 3: Netlify CLI

```bash
# Install CLI
npm install -g netlify-cli

# Deploy
netlify deploy --prod --dir=.
```

### Vercel

```bash
npm i -g vercel
vercel --prod
```

### GitHub Pages

```bash
# Push to gh-pages branch
git checkout -b gh-pages
git push origin gh-pages

# Enable Pages in repo settings
```

---

## 📁 Project Structure

```
webcodecs-decoder-poc/
├── index.html                    # Main entry point (Netlify-ready)
├── webcodecs_decoder_poc.html    # Original detailed PoC
├── WEBCODECS_DECODER_ANALYSIS.md # Deep technical analysis
├── _redirects                    # Netlify SPA routing
├── netlify.toml                  # Netlify config + security headers
└── package.json                  # Metadata + dev scripts
```

### Configuration Files

- **`netlify.toml`** - Security headers and build settings
- **`_redirects`** - SPA routing for clean URLs
- **`package.json`** - Dev dependencies and scripts

---

## 🔬 Technical Details

### How It Works

1. **Encoder Setup** - Creates a `VideoEncoder` and generates VP8 keyframes
2. **Chunk Manipulation** - Corrupts the encoded chunks' metadata
3. **Decoder Trigger** - Feeds malformed chunks to `VideoDecoder`
4. **Crash Analysis** - Monitors for null dereferences or async failures

### Architecture

```
JavaScript (Main Thread)
    ↓ WebCodecs API
Media Process / GPU Process
    ↓ IPC
Hardware Decoder (VideoToolbox/VA-API)
    ↓
CRASH 💥 (null pointer dereference)
```

### Related Research

This tool is inspired by the [Coruna](https://github.com/yourusername/coruna) iOS exploit chain, which uses similar type confusion techniques in:
- WASM JIT compilation (CVE-2024-23222)
- JavaScriptCore object handling
- PAC (Pointer Authentication) bypass

Read [WEBCODECS_DECODER_ANALYSIS.md](./WEBCODECS_DECODER_ANALYSIS.md) for the full comparison.

---

## 🐛 Troubleshooting

### "WebCodecs API not supported"

Your browser doesn't support WebCodecs. Try Chrome 94+ or Safari 16.4+.

### No crash observed

Some crashes happen asynchronously in the media process. Check:
- Browser console for errors
- System crash logs (`Console.app` on macOS)
- Try running tests multiple times

### Page freezes

This is expected - the decoder may hang on certain malformed inputs. Refresh the page.

### Netlify deployment fails

Ensure these files are committed:
- `index.html`
- `netlify.toml`
- `_redirects`

---

## ⚠️ Disclaimer

This tool is for **security research and educational purposes only**. It demonstrates vulnerability patterns to help:

- Browser vendors improve security
- Security researchers understand attack surfaces
- Developers write safer code

**Do not use this tool to attack systems you don't own.**

---

## 📄 License

MIT License - See LICENSE file for details.

Research use only. The authors are not responsible for misuse of this tool.

---

## 🙏 Acknowledgments

- WebKit team for the open-source browser engine
- W3C WebCodecs working group for the specification
- Security researchers advancing browser security

---

<div align="center">

**[Live Demo](https://your-site.netlify.app)** • **[Report Bug](../../issues)** • **[Request Feature](../../issues)**

</div>
