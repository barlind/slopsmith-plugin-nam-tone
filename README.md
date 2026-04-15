# NAM Tone Engine Plugin

Play through Neural Amp Modeler (NAM) amp models and cabinet impulse responses directly in the browser. Plug your guitar into a USB audio interface, click AMP, and hear your guitar processed through a neural network amp model — no external software needed.

## How It Works

### Signal Chain

```
Guitar (USB interface) → getUserMedia
  → Input Gain
  → NAM AudioWorklet (WASM amp model inference)
  → ConvolverNode (cabinet IR)
  → Output Gain
  → Speakers
```

The plugin captures your guitar via the Web Audio API, processes it through a Neural Amp Modeler compiled to WebAssembly running inside an AudioWorkletProcessor, then applies a cabinet impulse response using the browser's native ConvolverNode. The processed signal is routed to your speakers while the song's guitar stem is automatically muted.

### NAM Models

NAM (Neural Amp Modeler) uses neural networks to model real guitar amplifiers. The `.nam` model files contain the trained weights for a specific amp/pedal tone. Models are loaded into a WASM module compiled from [NeuralAmpModelerCore](https://github.com/sdatkinson/NeuralAmpModelerCore) using Emscripten in single-threaded mode (no SharedArrayBuffer required).

You can find free `.nam` models at [ToneHunt](https://tonehunt.org/) and other NAM community sites.

### Cabinet IRs

Cabinet impulse responses (`.wav` files) simulate the speaker cabinet. Without an IR, the raw NAM output sounds thin and fizzy. IRs are processed using the browser's native `ConvolverNode` which is highly optimized. On upload, IRs are automatically converted to browser-compatible format (PCM float32, 48kHz mono) via ffmpeg.

### Tone Auto-Switching

Songs can have multiple tones (e.g., Clean, Distortion, Lead). You can map each tone to a different preset. During playback, the plugin polls `highway.getToneChanges()` every 100ms and automatically switches the NAM model and IR when the active tone changes.

### Guitar Stem Ducking

When playing sloppak songs with separated stems, enabling AMP automatically mutes the guitar stem so you only hear your own playing through the amp model. The stem volume is restored when AMP is disabled. This can be toggled in settings.

## Setup

1. **Upload models**: Go to the NAM config screen → upload `.nam` files
2. **Upload IRs**: Upload `.wav` cabinet impulse response files
3. **Create presets**: Combine a model + IR with gain and gate settings
4. **Select input device**: In settings, choose your USB audio interface
5. **Play**: Open a song, click the **AMP** button in player controls

## Settings

- **Input Device** — Select your USB audio interface
- **Input Channel** — Mono (mix), Left only, or Right only
- **Input Gain** — Adjust input sensitivity (1.0 = unity)
- **Output Gain** — Master volume for processed signal
- **Noise Gate** — Threshold in dBFS to cut noise when not playing
- **Latency Offset** — Compensate for audio processing delay
- **Auto-mute guitar stem** — Mute the song's guitar stem when AMP is active

## Tone Mapping

1. Go to the NAM config screen
2. Search for a song
3. Each tone in the song gets a dropdown to assign a preset
4. Mappings auto-save and are applied during playback

## Architecture

```
plugins/nam_tone/
  plugin.json              # Plugin manifest
  routes.py                # Backend: SQLite DB, file upload, WASM serving
  screen.html              # Config screen UI
  screen.js                # Signal chain, tone switching, stem ducking, UI
  settings.html            # Inline settings panel
  worklet/
    nam-processor.js       # AudioWorkletProcessor (runs WASM inference)
  wasm/
    nam-core.wasm          # NeuralAmpModelerCore compiled to WASM
    nam-core.js            # Emscripten glue code
```

### WASM Build

The WASM artifacts were built from [NeuralAmpModelerCore](https://github.com/sdatkinson/NeuralAmpModelerCore) using Emscripten:

- Single-threaded (no SharedArrayBuffer, no COOP/COEP headers needed)
- `ALLOW_MEMORY_GROWTH=1` for large model files
- `FILESYSTEM=0` — no virtual filesystem overhead
- C bridge exposes: `nam_create`, `nam_destroy`, `nam_load_model`, `nam_process`, `nam_is_loaded`

To rebuild (requires [Emscripten SDK](https://emscripten.org/)):

```bash
git clone https://github.com/sdatkinson/NeuralAmpModelerCore.git
cd NeuralAmpModelerCore
git submodule update --init --recursive

em++ -O2 -DNAM_SAMPLE_FLOAT \
  -I Dependencies/eigen -I Dependencies/nlohmann -I NAM \
  -std=c++17 \
  nam_bridge.cpp NAM/*.cpp \
  -s WASM=1 -s MODULARIZE=1 -s EXPORT_NAME="NAMCore" \
  -s EXPORTED_FUNCTIONS="['_nam_create','_nam_destroy','_nam_load_model','_nam_process','_nam_is_loaded','_malloc','_free']" \
  -s "EXPORTED_RUNTIME_METHODS=['ccall','cwrap','HEAPU8','HEAPF32']" \
  -s ALLOW_MEMORY_GROWTH=1 -s ENVIRONMENT=web \
  -s SINGLE_FILE=0 -s FILESYSTEM=0 -s DISABLE_EXCEPTION_CATCHING=0 \
  -o nam-core.js
```

Copy `nam-core.js` and `nam-core.wasm` to `plugins/nam_tone/wasm/`.
