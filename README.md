# Qwen3.5-0.8B ONNX Desktop

> A portable Tauri v2 desktop application that runs the **Qwen3.5-0.8B** language model fully **on-device** — no servers, no cloud, no Python runtime. Just a single `.msi` installer and your GPU.

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Windows-0078D6.svg)](https://github.com/MushroomFleet/Qwen35-08B-onnx-desktop)
[![Tauri](https://img.shields.io/badge/Tauri-v2-FFC131.svg)](https://tauri.app/)
[![ONNX Runtime](https://img.shields.io/badge/ONNX_Runtime-WebGPU-00599C.svg)](https://onnxruntime.ai/)

---

## ✨ Overview

**Qwen3.5-0.8B ONNX Desktop** is a reference implementation for shipping a local large language model inside a Windows desktop app with **zero server-side dependencies**. It packages the Q4-quantized [Qwen3.5-0.8B](https://huggingface.co/onnx-community/Qwen3.5-0.8B-ONNX) model into a lightweight (~10 MB) MSI installer that downloads weights on first launch and runs entirely on your machine thereafter.

Built on Tauri v2, React 19, and ONNX Runtime Web with WebGPU acceleration — this project demonstrates how to deliver privacy-respecting, fully offline AI to end users without asking them to install Python, Docker, or configure a model server.

---

## 🎯 Key Features

- **🚀 Fully On-Device Inference** — All generation runs locally via ONNX Runtime Web. No telemetry, no API keys, no cloud calls after the initial weight download.
- **⚡ WebGPU Acceleration** — Primary execution provider with automatic WASM fallback for systems without WebGPU support.
- **📦 Tiny Installer** — The `.msi` ships at ~10 MB. The ~600 MB Q4 model weights are downloaded once on first run and cached permanently.
- **💬 Streaming Chat Interface** — Token-by-token streaming with live tokens/sec metrics, conversation history, and stop-generation control.
- **🧠 Thinking Mode Toggle** — Surface or hide Qwen3.5's chain-of-thought reasoning via a single toggle. `<think>` blocks render in collapsible "Reasoning" boxes.
- **💾 Persistent Conversations** — Chat sessions are saved as newline-delimited JSON to your app data directory; replay past sessions from a sidebar.
- **🎛️ Configurable Generation** — Tune `max_new_tokens`, `temperature`, `top_p`, and `repetition_penalty` from a settings drawer with persistent preferences.
- **📊 System Information Panel** — Live readout of execution provider, GPU adapter, tokens/sec, and RAM usage.
- **🔒 Offline After First Run** — Download once. Use forever. Air-gap friendly.

---

## 🏗️ Architecture

| Layer | Technology |
|---|---|
| Desktop shell | Tauri v2 (Rust) |
| Installer | MSI via `tauri-bundler` |
| Frontend | React 19 + TypeScript 5 |
| Build tool | Vite 6 |
| LLM runtime | `@huggingface/transformers` ≥ 3.5 (ORT-backed) |
| Execution providers | WebGPU (primary), WASM (fallback) |
| Model | `onnx-community/Qwen3.5-0.8B-ONNX` (Q4 quantized) |
| Styling | Tailwind CSS v4 |
| State management | Zustand |
| Inference isolation | Dedicated Web Worker |

The ORT session runs inside a dedicated Web Worker so model inference never blocks the UI thread. The Rust backend handles HTTP streaming downloads (avoiding gigabytes-in-JS-heap issues) and exposes the local cache directory via Tauri's asset protocol so the model loader can read weights as if they were remote URLs.

---

## 📥 Installation

### For End Users

1. Download the latest `Qwen3508B Desktop_x.x.x_x64_en-US.msi` from the [Releases page](https://github.com/MushroomFleet/Qwen35-08B-onnx-desktop/releases).
2. Run the installer.
3. Launch the app — on first run, it will download ~600 MB of model weights. Subsequent launches skip straight to the chat interface.

**System Requirements:**
- Windows 10/11 (x64)
- ~700 MB free disk space (for model cache)
- WebGPU-capable GPU recommended (NVIDIA RTX 3060+ or equivalent); WASM fallback works on any system but is slower

---

## 🛠️ Building from Source

### Prerequisites

- **Node.js** ≥ 20
- **Rust** stable toolchain (`rustup update stable`)
- **Tauri CLI v2** (`cargo install tauri-cli --version "^2.0"`)
- **Windows SDK** (for MSI bundling)

### Development

```bash
git clone https://github.com/MushroomFleet/Qwen35-08B-onnx-desktop.git
cd Qwen35-08B-onnx-desktop
npm install
npm run tauri dev
```

### Production Build

```bash
npm run tauri:build
```

The MSI will be output to:
```
src-tauri/target/release/bundle/msi/Qwen3508B Desktop_0.1.0_x64_en-US.msi
```

> **Tip:** Code-sign the MSI with `signtool.exe` before distribution to avoid Windows SmartScreen warnings.

---

## 🚀 Performance

| Metric | Target |
|---|---|
| Model load time | ≤ 10 seconds on WebGPU (RTX 3060+) |
| First token latency | ≤ 2 seconds after send |
| Streaming throughput (WebGPU) | ≥ 20 tok/s |
| Streaming throughput (WASM) | ≥ 5 tok/s |
| MSI installer size | ≤ 10 MB (weights excluded) |

---

## 🗂️ Project Structure

```
qwen3508b-desktop/
├── src-tauri/              # Rust backend
│   ├── src/
│   │   ├── commands/       # Download, cache validation, sysinfo
│   │   ├── state.rs        # Shared app state
│   │   └── lib.rs
│   └── tauri.conf.json     # Bundle / window / CSP config
├── src/                    # React frontend
│   ├── workers/
│   │   └── model.worker.ts # ORT session lives here
│   ├── stores/             # Zustand state (model, chat, settings)
│   ├── components/         # UI components
│   ├── lib/                # Path resolution, chat templates, parsers
│   └── App.tsx             # Phase router
├── public/
│   └── ort-wasm/           # ORT WASM binaries
└── package.json
```

---

## 🔄 User Flow

**First Run:**
1. App detects no local model cache → shows download screen
2. Rust streams ~600 MB of Q4 ONNX weights to `{app_cache_dir}/qwen3508b-onnx/`
3. Manifest written → ORT session initializes in Web Worker
4. Chat interface ready

**Subsequent Runs:**
1. Manifest detected → skip directly to model loading
2. ORT session ready in seconds → chat interface ready

**Sending a Message:**
1. User input → conversation formatted via Qwen3.5 chat template
2. `model.generate()` called with `TextStreamer` → tokens stream live
3. Stop button aborts generation; partial output is preserved

---

## 🛡️ Privacy & Offline Use

- **No telemetry.** The app makes zero network requests after the initial model download.
- **No accounts.** No login, no API keys, no cloud sync.
- **Air-gap friendly.** Once weights are cached, the app runs indefinitely without an internet connection.
- **Conversations stay local.** Chat history is stored as plain JSON in your app data directory and never leaves your machine.

---

## ⚙️ Configuration

Generation parameters are user-configurable via the in-app settings drawer:

| Parameter | Range | Default |
|---|---|---|
| `max_new_tokens` | 1–2048 | 512 |
| `temperature` | 0.0–2.0 | 0.7 |
| `top_p` | 0.1–1.0 | 0.9 |
| `repetition_penalty` | 1.0–2.0 | 1.1 |

Settings persist across sessions via Zustand's `persist` middleware.

---

## 🤝 Contributing

Contributions, issues, and feature requests are welcome. Please open an issue to discuss substantial changes before submitting a PR.

Areas where help is particularly appreciated:
- macOS and Linux build targets (currently Windows-only)
- Additional quantization options (Q8, FP16)
- Resumable downloads
- Multi-model support / model picker UI

---

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

The Qwen3.5-0.8B model itself is distributed under its own license — see the [model card on Hugging Face](https://huggingface.co/onnx-community/Qwen3.5-0.8B-ONNX).

---

## 🙏 Acknowledgements

- [Tauri](https://tauri.app/) — for making lightweight native desktop apps possible
- [Hugging Face Transformers.js](https://github.com/huggingface/transformers.js) — for the ORT-backed inference layer
- [ONNX Runtime](https://onnxruntime.ai/) — for WebGPU and WASM execution providers
- [Qwen Team](https://qwenlm.github.io/) — for the Qwen3.5 model family
- [onnx-community](https://huggingface.co/onnx-community) — for the ONNX-converted weights

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{qwen3508b_onnx_desktop,
  title = {Qwen3.5-0.8B ONNX Desktop: A portable Tauri-based desktop runtime for on-device LLM inference},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/Qwen35-08B-onnx-desktop},
  version = {1.0.0}
}
```

### Donate

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
