# Qwen3.5-0.8B ONNX Desktop

> **A fully local, offline-capable LLM desktop app — no Python, no server, no cloud.**

Built with **Tauri v2 · React · TypeScript · Vite** and powered by **ONNX Runtime Web (ORT)** with **WebGPU acceleration**, this app runs the [Qwen3.5-0.8B](https://huggingface.co/onnx-community/Qwen3.5-0.8B-ONNX) language model entirely inside a native Windows desktop window — inference happens on your own GPU or CPU, your data never leaves your machine.

The model weights (~600 MB, Q4 quantized) are downloaded once on first launch and cached permanently. Every subsequent run is fully offline.

---

## What It Does

| Capability | Detail |
|---|---|
| 🧠 **Local LLM inference** | Qwen3.5-0.8B Q4 ONNX, streamed token-by-token |
| ⚡ **WebGPU acceleration** | Runs on the GPU via the browser's WebGPU API; auto-falls back to WASM |
| 📥 **First-run model download** | Streams weights from Hugging Face to local cache with live progress bar |
| 💬 **Streaming chat UI** | Multi-turn conversation with live token streaming and a Stop button |
| 🤔 **Thinking mode** | Toggle Qwen3.5's chain-of-thought reasoning on or off; reasoning shown in a collapsible block |
| ⚙️ **Configurable generation** | `max_new_tokens`, `temperature`, `top_p`, `repetition_penalty` — all user-adjustable |
| 💾 **Conversation history** | Sessions saved locally; reload any past conversation |
| 📦 **Portable MSI installer** | Single `.msi` file — no runtime dependencies on the end-user machine |

---

## Why This Project Exists

Most local LLM tooling requires Python, a separate server process, Ollama, or a dedicated GPU runtime stack. This project proves a different path: **ship an LLM app the same way you ship any other desktop app** — as a self-contained installer that any Windows user can double-click.

The key insight is that modern browsers (embedded inside Tauri's WebView) already have:
- **ONNX Runtime Web** — a full ML inference engine compiled to WASM and WebGPU
- **`@huggingface/transformers`** — a JavaScript library that wraps ORT with a familiar `AutoModelForCausalLM` API

Tauri provides the native shell (file system, HTTP, MSI packaging) while React provides the UI. The model runs in a Web Worker so the interface stays responsive during inference. The result is a genuinely portable app with no external dependencies.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Tauri v2 (Rust shell)                                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Tauri WebView (Chromium)                             │  │
│  │                                                       │  │
│  │  React UI Thread          Web Worker                  │  │
│  │  ┌──────────────┐         ┌──────────────────────┐   │  │
│  │  │ ChatScreen   │ ◄─────► │ model.worker.ts      │   │  │
│  │  │ DownloadUI   │ tokens  │ @hf/transformers      │   │  │
│  │  │ StatusBar    │         │ ORT WebGPU / WASM     │   │  │
│  │  └──────────────┘         └──────────────────────┘   │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  Rust Commands                                              │
│  ├── download_model()  — HTTP stream → disk                 │
│  ├── check_model_cache() — manifest validation              │
│  └── fs / path plugins — cache dir resolution               │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
   {app_cache_dir}/qwen3508b-onnx/
   ├── model_q4.onnx
   ├── model_q4.onnx_data
   ├── tokenizer.json
   ├── config.json
   └── manifest.json
```

- **Rust layer** handles all disk I/O, HTTP streaming, and cache validation — keeping large binary operations out of the JavaScript heap.
- **Web Worker** runs ORT inference in isolation, preventing UI thread blocking during token generation.
- **Zustand stores** manage model lifecycle, chat history, and generation settings, with settings persisted across restarts.

---

## Requirements

### To Run the App
- Windows 10/11 x64
- ~1 GB free disk space (for model cache)
- Internet connection on first launch only (for model download)
- A WebGPU-capable GPU is recommended (any modern Nvidia/AMD/Intel discrete GPU); the app will fall back to CPU WASM if WebGPU is unavailable

### To Build From Source
- [Node.js](https://nodejs.org/) ≥ 20
- [Rust](https://rustup.rs/) stable toolchain
- [Tauri CLI v2](https://tauri.app/start/): `cargo install tauri-cli --version "^2.0"`
- Windows SDK (included with Visual Studio Build Tools)

---

## Getting Started

### Option A — Install the MSI (end users)

1. Download the latest `.msi` from the [Releases](https://github.com/MushroomFleet/Qwen35-08B-onnx-desktop/releases) page.
2. Run the installer.
3. Launch **Qwen3.5-0.8B Desktop** from the Start Menu.
4. On first launch, click **Download Model** and wait for the ~600 MB Q4 weights to download.
5. Start chatting — fully offline from this point on.

---

## First-Run Download Flow

On first launch the app detects that no model cache exists and presents the download screen:

```
┌──────────────────────────────────────────┐
│         Qwen3.5-0.8B                     │
│                                          │
│  The model weights (~600 MB, Q4          │
│  quantized) will be downloaded once      │
│  and cached locally for offline use.     │
│                                          │
│           [ Download Model ]             │
│                                          │
│  File 3 of 7: model_q4.onnx_data        │
│  ████████████░░░░░░░░░░  61%             │
└──────────────────────────────────────────┘
```

The download streams through Tauri's Rust HTTP layer directly to disk — no gigabytes in memory. Progress is reported per-file and as an overall percentage. Once complete, the model loads automatically and you land in the chat interface.

On all subsequent launches the cached weights are used immediately — no network required.

---

## Using the App

### Chat
Type your message in the input bar and press **Enter** (or **Shift+Enter** for a new line). The assistant's response streams in token by token. Click **Stop** at any time to halt generation — the partial response is kept.

### Thinking Mode
Toggle the **Thinking** button in the toolbar to enable Qwen3.5's built-in chain-of-thought reasoning. When on, a collapsible **Reasoning** block appears above each answer showing the model's internal deliberation. Toggle off for faster, direct responses.

### Settings
Click ⚙ to open the settings drawer:

| Setting | Default | Range | Effect |
|---|---|---|---|
| `max_new_tokens` | 512 | 1 – 2048 | Maximum response length |
| `temperature` | 0.7 | 0.0 – 2.0 | Randomness; lower = more deterministic |
| `top_p` | 0.9 | 0.1 – 1.0 | Nucleus sampling threshold |
| `repetition_penalty` | 1.1 | 1.0 – 2.0 | Penalises repeated phrases |

Settings are persisted between sessions.

### Conversation History
Past sessions are saved automatically. Click 📋 in the toolbar to open the history sidebar and reload any previous conversation.

### Status Bar
The footer shows the active execution provider (`WEBGPU` or `WASM`), live tokens/second, and GPU adapter name where available.

---

## Model Information

| Property | Value |
|---|---|
| Model | [Qwen3.5-0.8B](https://huggingface.co/Qwen/Qwen3.5-0.8B) |
| ONNX source | [onnx-community/Qwen3.5-0.8B-ONNX](https://huggingface.co/onnx-community/Qwen3.5-0.8B-ONNX) |
| Quantization | Q4 (4-bit weights) |
| Parameters | 0.8 billion |
| Context window | 32,768 tokens |
| License | [Apache 2.0](https://huggingface.co/Qwen/Qwen3.5-0.8B/blob/main/LICENSE) |

Qwen3.5-0.8B is Alibaba Cloud's compact instruct model with native chain-of-thought reasoning support. The Q4 ONNX variant offers a good balance between response quality and inference speed on consumer hardware.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Desktop shell | [Tauri v2](https://tauri.app/) (Rust) |
| Frontend | React 19 + TypeScript 5 |
| Build tool | Vite 6 |
| LLM runtime | [`@huggingface/transformers`](https://github.com/huggingface/transformers.js) ≥ 3.5 |
| Inference engine | [ONNX Runtime Web](https://onnxruntime.ai/docs/get-started/with-javascript/web.html) |
| Execution providers | WebGPU (primary) · WASM (fallback) |
| State management | [Zustand](https://zustand-demo.pmnd.rs/) |
| Styling | Tailwind CSS v4 |
| Installer | MSI via `tauri-bundler` |

---

## Troubleshooting

**The app shows "WASM" in the footer instead of "WEBGPU"**
Your GPU or driver does not support WebGPU. The app will still work via WASM, but expect lower tokens/second (~5–10 tok/s vs ~40+ tok/s on WebGPU). Ensure your GPU drivers are up to date.

**Download fails or hangs**
Check your internet connection. If a partial download occurred, the app will detect the incomplete cache and offer to re-download cleanly. Ensure you have ~700 MB of free disk space.

**"Failed to load model — insufficient memory"**
Close other applications to free RAM/VRAM. The Q4 model requires ~900 MB VRAM on WebGPU or ~1.2 GB system RAM on WASM.

**SmartScreen warning on install**
The MSI is not code-signed in development builds. Click "More info → Run anyway" to proceed. Release builds should be signed; see the [contributing guide](#contributing) for details.

---

Areas of interest:
- macOS / Linux support (Tauri supports both; the main gap is installer format and path handling)
- RAG / document ingestion pipeline
- Multiple model support (model switcher UI)
- Code-signing pipeline for release MSI builds

---

## Licence

This project is released under the **MIT Licence**. See [LICENSE](./LICENSE) for details.

The bundled model weights are governed by the [Apache 2.0 licence](https://huggingface.co/Qwen/Qwen3.5-0.8B/blob/main/LICENSE) from Alibaba Cloud / Qwen Team.

---

## Acknowledgements

- [Qwen Team @ Alibaba Cloud](https://github.com/QwenLM) — for the Qwen3.5 model family
- [Xenova / onnx-community](https://huggingface.co/onnx-community) — for the ONNX-converted and quantized model weights
- [Hugging Face Transformers.js](https://github.com/huggingface/transformers.js) — for the ORT-backed JavaScript inference library
- [Tauri](https://tauri.app/) — for making lightweight, secure desktop apps with web frontends practical

---

## Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{qwen3508b_onnx_desktop,
  title = {Qwen3508B-onnx-desktop: A portable Tauri v2 desktop application for local LLM inference},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/Qwen35-08B-onnx-desktop},
  version = {0.5.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
