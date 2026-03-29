# Qwen3508B-onnx-desktop

<!-- TINS Specification v1.0 -->
<!-- ZS:COMPLEXITY:HIGH -->
<!-- ZS:PRIORITY:HIGH -->
<!-- ZS:PLATFORM:DESKTOP -->
<!-- ZS:LANGUAGE:TYPESCRIPT -->

---

## Description

A portable Tauri v2 desktop application (distributed as a single `.msi` installer) that runs the **Qwen3.5-0.8B** language model fully **on-device** using ONNX Runtime Web (ORT) with WebGPU acceleration and WASM fallback. On first run the application automatically downloads the Q4-quantized ONNX model weights from Hugging Face (`onnx-community/Qwen3.5-0.8B-ONNX`) into a persistent local cache directory, then loads the model into the browser-side ORT session embedded inside the Tauri WebView.

The application surfaces a clean React/TypeScript chat interface built with Vite. All inference runs inside the Tauri WebView process — no network calls after the initial model download, no server, no cloud dependency. The `.msi` bundle ships without model weights; the weights are downloaded once on first launch and cached permanently, enabling fully offline use on subsequent runs.

Target audience: developers who want a reference implementation for shipping a local LLM inside a Windows desktop app with zero Python/server-side runtime requirements.

---

## Functionality

### Core Features

1. **First-Run Model Download**
   - On first launch, detect the absence of the local model cache.
   - Display a full-screen download progress UI with byte-level progress bar, ETA, and cancel button.
   - Download the Q4-quantized ONNX weights from `https://huggingface.co/onnx-community/Qwen3.5-0.8B-ONNX` using Tauri's HTTP plugin (streams through Rust to disk to avoid holding gigabytes in JS heap).
   - Save weights to the platform cache directory: `{app_cache_dir}/qwen3508b-onnx/`.
   - On completion, persist a `manifest.json` marker recording the model version and download timestamp.
   - If download is cancelled or fails, clean up partial files and show retry UI.

2. **Model Initialisation**
   - After weights are confirmed present, initialise an `AutoModelForCausalLM` session via `@huggingface/transformers` (ORT-backed) with execution provider priority: `["webgpu", "wasm"]`.
   - Show a "Loading model…" spinner with a sub-status line (e.g., "Compiling WebGPU shaders…").
   - Once the session is ready, transition to the chat interface.

3. **Chat Interface**
   - A standard conversational UI: message history list + bottom input bar.
   - User messages appear right-aligned; assistant messages stream in left-aligned with a blinking cursor.
   - Token streaming is implemented via the `TextStreamer` callback from `@huggingface/transformers`.
   - Each assistant turn shows live tokens as they are generated.
   - A **Stop** button appears during generation and aborts the stream.
   - Generation parameters are user-configurable via a settings drawer: `max_new_tokens` (default 512), `temperature` (default 0.7), `top_p` (default 0.9), `repetition_penalty` (default 1.1).
   - Full conversation history is maintained for multi-turn context.

4. **Thinking Mode Toggle**
   - A toggle in the toolbar enables/disables Qwen3.5's built-in chain-of-thought reasoning mode.
   - When enabled, `<think>…</think>` blocks in the model output are rendered in a collapsible grey box labelled "Reasoning" above the final answer.
   - When disabled, the system prompt suppresses thinking output.

5. **Persistence**
   - Conversation history is persisted to `{app_data_dir}/conversations/` as newline-delimited JSON files.
   - Each session is a separate file named by ISO timestamp.
   - A sidebar lists past sessions; clicking loads them into the message view (read-only replay mode).

6. **System Information Panel**
   - A footer bar shows: execution provider in use (`WebGPU` or `WASM`), current tokens/sec, GPU adapter name (if WebGPU), and RAM usage.

### UI Layout

```
+------------------------------------------------------------------+
| Qwen3.5-0.8B  [Thinking: ON/OFF]  [Settings ⚙]  [History 📋]   |
+------------------------------------------------------------------+
|                                                                  |
|  ┌──────────────────────────────────────────────────────────┐   |
|  │ [Reasoning ▶]                                            │   |
|  │  The user is asking about…                               │   |
|  └──────────────────────────────────────────────────────────┘   |
|                                                                  |
|  Assistant: Here is my answer…▌                                  |
|                                                                  |
|                              User: Tell me about Rust traits.    |
|                                                                  |
+------------------------------------------------------------------+
| [WebGPU | NVIDIA RTX 4070 | 47 tok/s | 1.1 GB]                  |
+------------------------------------------------------------------+
| [        Type a message…                        ] [Send ▶]       |
+------------------------------------------------------------------+
```

### User Flows

**First Run:**
1. App launches → check `{app_cache_dir}/qwen3508b-onnx/manifest.json` → not found.
2. Show download screen with model name, total size (~600 MB for Q4), progress bar.
3. User clicks **Download** → Tauri Rust command streams bytes to disk, emits progress events.
4. Progress bar updates in real time. ETA shown in `MM:SS` format.
5. Download completes → manifest written → automatic transition to "Loading model…" screen.
6. ORT session initialised → chat screen shown.

**Subsequent Runs:**
1. App launches → manifest found → skip to "Loading model…" screen.
2. ORT session initialised → chat screen shown.

**Sending a Message:**
1. User types in the input bar and presses Enter or clicks Send.
2. User message appended to history list.
3. Full conversation formatted as chat template tokens via `AutoTokenizer`.
4. `model.generate()` called with `TextStreamer`; tokens stream into the assistant message bubble.
5. **Stop** button aborts generation; partial response is kept in history.
6. Generation complete → input re-enabled.

**Cancelling Download:**
1. User clicks Cancel → Tauri command cancels the HTTP stream, deletes partial files.
2. App returns to download prompt screen.

### Edge Cases and Error States

- **No internet on first run:** show error "Model download requires an internet connection. Please connect and restart." with a Retry button.
- **Insufficient disk space:** detect before download (Tauri `fs` stat); show "Insufficient disk space. ~700 MB required."
- **WebGPU unavailable:** silently fall back to WASM; note `WASM` in footer. No error shown to user.
- **OOM during model load:** catch the ORT error; display "Failed to load model — insufficient GPU/system memory. Try closing other applications." with a Reload button.
- **Corrupted cache:** if `manifest.json` is present but model files are missing or size-mismatched, delete cache and re-trigger download flow.
- **Generation error:** display "Generation failed" inline in the assistant bubble with the raw error message collapsed.
- **Empty input:** Send button and Enter key are disabled when the input is blank or whitespace-only.

---

## Technical Implementation

### Stack

| Layer | Technology |
|---|---|
| Desktop shell | Tauri v2 (Rust) |
| Installer | MSI via `tauri-bundler` |
| Frontend framework | React 19 + TypeScript 5 |
| Build tool | Vite 6 |
| LLM runtime | `@huggingface/transformers` ≥ 3.5 (ORT-backed) |
| Execution providers | WebGPU (primary), WASM (fallback) |
| Model | `onnx-community/Qwen3.5-0.8B-ONNX` — Q4 quantized |
| Styling | Tailwind CSS v4 |
| State management | Zustand |
| Persistence | Tauri `fs` plugin + `path` plugin |
| HTTP streaming | Tauri `http` plugin (Rust-side streaming) |

---

### Project Structure

```
qwen3508b-desktop/
├── src-tauri/
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   └── src/
│       ├── main.rs
│       ├── lib.rs
│       ├── commands/
│       │   ├── download.rs       # HTTP streaming download command
│       │   ├── cache.rs          # Cache validation / manifest logic
│       │   └── sysinfo.rs        # RAM / adapter info
│       └── state.rs              # AppState (download handle)
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── vite-env.d.ts
│   ├── workers/
│   │   └── model.worker.ts       # ORT session runs in dedicated Web Worker
│   ├── stores/
│   │   ├── modelStore.ts         # model lifecycle state (Zustand)
│   │   ├── chatStore.ts          # conversation history (Zustand)
│   │   └── settingsStore.ts      # generation params (Zustand)
│   ├── components/
│   │   ├── DownloadScreen.tsx
│   │   ├── LoadingScreen.tsx
│   │   ├── ChatScreen.tsx
│   │   ├── MessageBubble.tsx
│   │   ├── ThinkingBlock.tsx
│   │   ├── SettingsDrawer.tsx
│   │   ├── HistorySidebar.tsx
│   │   └── StatusBar.tsx
│   ├── lib/
│   │   ├── modelPaths.ts         # Resolves local file:// paths for ORT
│   │   ├── chatTemplate.ts       # Qwen3.5 chat template helper
│   │   └── parseThinking.ts      # Strips / extracts <think> blocks
│   └── types/
│       └── index.ts
├── public/
│   └── ort-wasm/                 # ORT WASM binaries (copied by Vite plugin)
├── index.html
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.ts
└── package.json
```

---

### Tauri Configuration (`tauri.conf.json`)

```json
{
  "productName": "Qwen3508B Desktop",
  "version": "0.1.0",
  "identifier": "com.yourorg.qwen3508b",
  "build": {
    "frontendDist": "../dist",
    "devUrl": "http://localhost:5173"
  },
  "app": {
    "windows": [
      {
        "title": "Qwen3.5-0.8B",
        "width": 1024,
        "height": 768,
        "minWidth": 640,
        "minHeight": 480,
        "resizable": true
      }
    ],
    "security": {
      "csp": "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' blob:; style-src 'self' 'unsafe-inline'; connect-src 'self' https://huggingface.co https://cdn-lfs.huggingface.co blob:; worker-src 'self' blob:;"
    }
  },
  "bundle": {
    "active": true,
    "targets": ["msi"],
    "icon": ["icons/icon.ico"],
    "windows": {
      "wix": {
        "language": "en-US"
      }
    }
  },
  "plugins": {
    "fs": { "scope": { "allow": ["$APPDATA/**", "$APPCACHE/**"] } },
    "http": { "dangerousAllowAllOrigins": false }
  }
}
```

---

### Cargo.toml (relevant sections)

```toml
[dependencies]
tauri = { version = "2", features = ["protocol-asset"] }
tauri-plugin-fs = "2"
tauri-plugin-http = "2"
tauri-plugin-path = "2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["stream"] }
futures-util = "0.3"
```

---

### Rust: Download Command (`src-tauri/src/commands/download.rs`)

```rust
use std::path::PathBuf;
use tauri::{AppHandle, Emitter};
use tokio::io::AsyncWriteExt;
use futures_util::StreamExt;

/// Files to download from the Q4 ONNX model repo.
/// Resolved to: https://huggingface.co/{repo}/resolve/main/{file}
const MODEL_REPO: &str = "onnx-community/Qwen3.5-0.8B-ONNX";

const MODEL_FILES: &[(&str, &str)] = &[
    ("onnx/model_q4.onnx",              "model_q4.onnx"),
    ("onnx/model_q4.onnx_data",         "model_q4.onnx_data"),
    ("tokenizer.json",                   "tokenizer.json"),
    ("tokenizer_config.json",            "tokenizer_config.json"),
    ("special_tokens_map.json",          "special_tokens_map.json"),
    ("generation_config.json",           "generation_config.json"),
    ("config.json",                      "config.json"),
];

#[derive(Clone, serde::Serialize)]
struct DownloadProgress {
    file: String,
    downloaded: u64,
    total: u64,
    file_index: usize,
    file_count: usize,
}

#[tauri::command]
pub async fn download_model(app: AppHandle, cache_dir: String) -> Result<(), String> {
    let base = PathBuf::from(&cache_dir);
    tokio::fs::create_dir_all(&base).await.map_err(|e| e.to_string())?;

    let client = reqwest::Client::new();
    let file_count = MODEL_FILES.len();

    for (idx, (hf_path, local_name)) in MODEL_FILES.iter().enumerate() {
        let url = format!(
            "https://huggingface.co/{}/resolve/main/{}",
            MODEL_REPO, hf_path
        );
        let dest = base.join(local_name);

        let resp = client.get(&url).send().await.map_err(|e| e.to_string())?;
        let total = resp.content_length().unwrap_or(0);
        let mut downloaded: u64 = 0;
        let mut file = tokio::fs::File::create(&dest).await.map_err(|e| e.to_string())?;
        let mut stream = resp.bytes_stream();

        while let Some(chunk) = stream.next().await {
            let chunk = chunk.map_err(|e| e.to_string())?;
            file.write_all(&chunk).await.map_err(|e| e.to_string())?;
            downloaded += chunk.len() as u64;

            app.emit("download-progress", DownloadProgress {
                file: local_name.to_string(),
                downloaded,
                total,
                file_index: idx,
                file_count,
            }).ok();
        }
    }

    // Write manifest
    let manifest = serde_json::json!({
        "model": "onnx-community/Qwen3.5-0.8B-ONNX",
        "quantization": "q4",
        "downloaded_at": chrono::Utc::now().to_rfc3339()
    });
    tokio::fs::write(
        base.join("manifest.json"),
        serde_json::to_string_pretty(&manifest).unwrap(),
    ).await.map_err(|e| e.to_string())?;

    Ok(())
}

#[tauri::command]
pub fn check_model_cache(cache_dir: String) -> bool {
    let base = PathBuf::from(&cache_dir);
    let manifest = base.join("manifest.json");
    if !manifest.exists() { return false; }
    // Verify all files present
    for (_, local_name) in MODEL_FILES {
        if !base.join(local_name).exists() { return false; }
    }
    true
}
```

---

### Rust: lib.rs

```rust
mod commands;
use commands::{download::download_model, download::check_model_cache};

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_fs::init())
        .plugin(tauri_plugin_http::init())
        .plugin(tauri_plugin_path::init())
        .invoke_handler(tauri::generate_handler![
            download_model,
            check_model_cache,
        ])
        .run(tauri::generate_context!())
        .expect("error while running Tauri application");
}
```

---

### Vite Configuration (`vite.config.ts`)

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { internalIpV4 } from "internal-ip";
import { viteStaticCopy } from "vite-plugin-static-copy";
import path from "path";

export default defineConfig(async () => ({
  plugins: [
    react(),
    // Copy ORT WASM binaries to public so the browser can fetch them
    viteStaticCopy({
      targets: [
        {
          src: path.resolve(
            __dirname,
            "node_modules/onnxruntime-web/dist/*.wasm"
          ),
          dest: "ort-wasm",
        },
      ],
    }),
  ],
  // Required for Tauri to work in dev mode on Windows
  server: {
    host: "0.0.0.0",
    port: 5173,
    strictPort: true,
    hmr: {
      protocol: "ws",
      host: await internalIpV4(),
      port: 5183,
    },
  },
  // Allow Vite to import .onnx files as URLs
  assetsInclude: ["**/*.onnx"],
  resolve: {
    alias: { "@": path.resolve(__dirname, "src") },
  },
  // Allow top-level await (needed by @huggingface/transformers)
  optimizeDeps: {
    exclude: ["@huggingface/transformers"],
  },
  build: {
    target: "esnext",
    rollupOptions: {
      output: {
        format: "esm",
      },
    },
  },
  worker: {
    format: "es",
  },
}));
```

---

### ORT WASM Path Bootstrap (`src/main.tsx`)

```typescript
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import "./index.css";
import * as ort from "onnxruntime-web";

// Tell ORT where to find the WASM binaries (served by Vite from /ort-wasm/)
ort.env.wasm.wasmPaths = "/ort-wasm/";

// Enable WebGPU if available; otherwise ORT will auto-fall back to WASM
ort.env.webgpu = { powerPreference: "high-performance" };

ReactDOM.createRoot(document.getElementById("root") as HTMLElement).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

---

### Model Worker (`src/workers/model.worker.ts`)

The ORT session is initialised inside a dedicated Web Worker to avoid blocking the UI thread during inference. The worker communicates with the main thread via `postMessage`.

```typescript
import {
  AutoModelForCausalLM,
  AutoTokenizer,
  TextStreamer,
  InterruptableStopCriteria,
} from "@huggingface/transformers";
import { convertTokenizerToHF } from "@huggingface/transformers";
import type { PreTrainedModel, PreTrainedTokenizer } from "@huggingface/transformers";

// Resolve local file:// paths injected from main thread
let modelPath: string;
let model: PreTrainedModel | null = null;
let tokenizer: PreTrainedTokenizer | null = null;
let stopCriteria = new InterruptableStopCriteria();

// ─── Message Protocol ───────────────────────────────────────────────────────

type WorkerInMessage =
  | { type: "init"; payload: { modelPath: string } }
  | { type: "generate"; payload: GenerateRequest }
  | { type: "abort" };

type GenerateRequest = {
  messages: { role: string; content: string }[];
  maxNewTokens: number;
  temperature: number;
  topP: number;
  repetitionPenalty: number;
  thinkingEnabled: boolean;
};

// ─── Init ────────────────────────────────────────────────────────────────────

async function init(localModelPath: string) {
  postMessage({ type: "status", payload: "Loading tokenizer…" });

  tokenizer = await AutoTokenizer.from_pretrained(localModelPath, {
    local_files_only: true,
  });

  postMessage({ type: "status", payload: "Loading model (this may take a moment)…" });

  model = await AutoModelForCausalLM.from_pretrained(localModelPath, {
    dtype: "q4",
    device: "webgpu",     // ORT will fall back to wasm if webgpu unavailable
    local_files_only: true,
  });

  postMessage({ type: "ready", payload: { device: (model as any).device ?? "wasm" } });
}

// ─── Generate ────────────────────────────────────────────────────────────────

async function generate(req: GenerateRequest) {
  if (!model || !tokenizer) return;

  stopCriteria = new InterruptableStopCriteria();

  const systemPrompt = req.thinkingEnabled
    ? "You are a helpful assistant. Think step by step before answering."
    : "You are a helpful assistant. /no_think";

  const messages = [
    { role: "system", content: systemPrompt },
    ...req.messages,
  ];

  const inputs = tokenizer.apply_chat_template(messages, {
    add_generation_prompt: true,
    return_dict: true,
  });

  const streamer = new TextStreamer(tokenizer, {
    skip_prompt: true,
    callback_function: (token: string) => {
      postMessage({ type: "token", payload: token });
    },
  });

  await model.generate({
    ...inputs,
    max_new_tokens: req.maxNewTokens,
    temperature: req.temperature,
    top_p: req.topP,
    repetition_penalty: req.repetitionPenalty,
    do_sample: req.temperature > 0,
    streamer,
    stopping_criteria: stopCriteria,
  });

  postMessage({ type: "done" });
}

// ─── Message Handler ─────────────────────────────────────────────────────────

self.addEventListener("message", async (e: MessageEvent<WorkerInMessage>) => {
  const msg = e.data;
  switch (msg.type) {
    case "init":
      await init(msg.payload.modelPath).catch((err) =>
        postMessage({ type: "error", payload: String(err) })
      );
      break;
    case "generate":
      await generate(msg.payload).catch((err) =>
        postMessage({ type: "error", payload: String(err) })
      );
      break;
    case "abort":
      stopCriteria.setInterrupted(true);
      break;
  }
});
```

---

### Model Path Resolution (`src/lib/modelPaths.ts`)

Tauri exposes the app cache directory at runtime. ORT requires a base URL or directory path. We convert the native filesystem path to a `asset://` protocol URL which Tauri's asset protocol handler serves directly.

```typescript
import { appCacheDir, join } from "@tauri-apps/api/path";
import { convertFileSrc } from "@tauri-apps/api/core";

const MODEL_DIR = "qwen3508b-onnx";

/**
 * Returns the local asset:// URL base path that @huggingface/transformers
 * can use as a `pretrained_model_name_or_path`.
 */
export async function getLocalModelUrl(): Promise<string> {
  const cacheDir = await appCacheDir();
  const modelDir = await join(cacheDir, MODEL_DIR);
  // Tauri asset protocol: asset://localhost/{absolute_path}
  return convertFileSrc(modelDir);
}

export async function getCacheDirPath(): Promise<string> {
  const cacheDir = await appCacheDir();
  return join(cacheDir, MODEL_DIR);
}
```

---

### Zustand Model Store (`src/stores/modelStore.ts`)

```typescript
import { create } from "zustand";

export type ModelPhase =
  | "checking"       // Checking if cache exists
  | "needs-download" // Cache absent, prompt user
  | "downloading"    // Tauri download in progress
  | "loading"        // ORT session initialising
  | "ready"          // Model loaded, chat available
  | "error";         // Fatal error

interface DownloadProgress {
  fileIndex: number;
  fileCount: number;
  fileName: string;
  downloaded: number;
  total: number;
}

interface ModelState {
  phase: ModelPhase;
  downloadProgress: DownloadProgress | null;
  loadStatus: string;
  error: string | null;
  device: string | null;         // "webgpu" | "wasm"
  tokensPerSecond: number;

  setPhase: (phase: ModelPhase) => void;
  setDownloadProgress: (p: DownloadProgress) => void;
  setLoadStatus: (s: string) => void;
  setError: (e: string) => void;
  setDevice: (d: string) => void;
  setTPS: (tps: number) => void;
}

export const useModelStore = create<ModelState>((set) => ({
  phase: "checking",
  downloadProgress: null,
  loadStatus: "",
  error: null,
  device: null,
  tokensPerSecond: 0,

  setPhase: (phase) => set({ phase }),
  setDownloadProgress: (p) => set({ downloadProgress: p }),
  setLoadStatus: (s) => set({ loadStatus: s }),
  setError: (e) => set({ error: e, phase: "error" }),
  setDevice: (d) => set({ device: d }),
  setTPS: (tps) => set({ tokensPerSecond: tps }),
}));
```

---

### Zustand Chat Store (`src/stores/chatStore.ts`)

```typescript
import { create } from "zustand";

export interface ChatMessage {
  id: string;
  role: "user" | "assistant";
  content: string;             // Final content (no <think> blocks)
  thinking?: string;           // Extracted <think> content if present
  isStreaming?: boolean;
  timestamp: number;
}

interface ChatState {
  messages: ChatMessage[];
  isGenerating: boolean;

  addMessage: (msg: Omit<ChatMessage, "id" | "timestamp">) => string;
  appendToken: (id: string, token: string) => void;
  finaliseMessage: (id: string) => void;
  clearMessages: () => void;
}

let idCounter = 0;

export const useChatStore = create<ChatState>((set) => ({
  messages: [],
  isGenerating: false,

  addMessage: (msg) => {
    const id = `msg-${Date.now()}-${idCounter++}`;
    set((state) => ({
      messages: [...state.messages, { ...msg, id, timestamp: Date.now() }],
      isGenerating: msg.role === "assistant",
    }));
    return id;
  },

  appendToken: (id, token) =>
    set((state) => ({
      messages: state.messages.map((m) =>
        m.id === id ? { ...m, content: m.content + token } : m
      ),
    })),

  finaliseMessage: (id) =>
    set((state) => ({
      isGenerating: false,
      messages: state.messages.map((m) =>
        m.id === id ? { ...m, isStreaming: false } : m
      ),
    })),

  clearMessages: () => set({ messages: [] }),
}));
```

---

### Think Block Parser (`src/lib/parseThinking.ts`)

```typescript
/**
 * Separates <think>…</think> content from the main response.
 * Returns { thinking, answer } where either may be empty string.
 */
export function parseThinkingBlocks(raw: string): {
  thinking: string;
  answer: string;
} {
  const thinkRegex = /<think>([\s\S]*?)<\/think>/g;
  let thinking = "";
  let answer = raw;

  const match = thinkRegex.exec(raw);
  if (match) {
    thinking = match[1].trim();
    answer = raw.replace(thinkRegex, "").trim();
  }

  return { thinking, answer };
}
```

---

### App.tsx — Phase Router

```typescript
import React, { useEffect } from "react";
import { invoke } from "@tauri-apps/api/core";
import { listen } from "@tauri-apps/api/event";
import { useModelStore } from "./stores/modelStore";
import { getCacheDirPath, getLocalModelUrl } from "./lib/modelPaths";
import { DownloadScreen } from "./components/DownloadScreen";
import { LoadingScreen } from "./components/LoadingScreen";
import { ChatScreen } from "./components/ChatScreen";

// Singleton Web Worker — persists across phase changes
let modelWorker: Worker | null = null;

function getWorker(): Worker {
  if (!modelWorker) {
    modelWorker = new Worker(
      new URL("./workers/model.worker.ts", import.meta.url),
      { type: "module" }
    );
  }
  return modelWorker;
}

export default function App() {
  const { phase, setPhase, setDownloadProgress, setLoadStatus, setDevice, setError } =
    useModelStore();

  // ── Step 1: Check cache on mount ──────────────────────────────────────────
  useEffect(() => {
    (async () => {
      try {
        const cacheDir = await getCacheDirPath();
        const cached: boolean = await invoke("check_model_cache", { cacheDir });
        setPhase(cached ? "loading" : "needs-download");
        if (cached) initModel();
      } catch (e) {
        setError(String(e));
      }
    })();
  }, []);

  // ── Step 2 (optional): Download ───────────────────────────────────────────
  async function startDownload() {
    setPhase("downloading");
    const cacheDir = await getCacheDirPath();

    // Listen for Rust progress events
    const unlisten = await listen<{
      file: string; downloaded: number; total: number;
      file_index: number; file_count: number;
    }>("download-progress", (event) => {
      setDownloadProgress({
        fileName: event.payload.file,
        downloaded: event.payload.downloaded,
        total: event.payload.total,
        fileIndex: event.payload.file_index,
        fileCount: event.payload.file_count,
      });
    });

    try {
      await invoke("download_model", { cacheDir });
      unlisten();
      setPhase("loading");
      initModel();
    } catch (e) {
      unlisten();
      setError(String(e));
    }
  }

  // ── Step 3: Init ORT model in worker ─────────────────────────────────────
  async function initModel() {
    const modelUrl = await getLocalModelUrl();
    const worker = getWorker();

    worker.onmessage = (e) => {
      const msg = e.data;
      switch (msg.type) {
        case "status":
          setLoadStatus(msg.payload);
          break;
        case "ready":
          setDevice(msg.payload.device);
          setPhase("ready");
          break;
        case "error":
          setError(msg.payload);
          break;
      }
    };

    worker.postMessage({ type: "init", payload: { modelPath: modelUrl } });
  }

  // ── Render ────────────────────────────────────────────────────────────────
  switch (phase) {
    case "checking":
      return <LoadingScreen status="Checking local model cache…" />;
    case "needs-download":
      return <DownloadScreen onDownload={startDownload} />;
    case "downloading":
      return <DownloadScreen downloading />;
    case "loading":
      return <LoadingScreen />;
    case "ready":
      return <ChatScreen worker={getWorker()} />;
    case "error":
      return <LoadingScreen isError />;
    default:
      return null;
  }
}
```

---

### DownloadScreen Component (`src/components/DownloadScreen.tsx`)

```typescript
import React from "react";
import { useModelStore } from "../stores/modelStore";

interface Props {
  onDownload?: () => void;
  downloading?: boolean;
}

export function DownloadScreen({ onDownload, downloading = false }: Props) {
  const { downloadProgress } = useModelStore();

  const totalPct = downloadProgress
    ? Math.round((downloadProgress.downloaded / downloadProgress.total) * 100)
    : 0;

  const overallPct = downloadProgress
    ? Math.round(
        ((downloadProgress.fileIndex + totalPct / 100) /
          downloadProgress.fileCount) *
          100
      )
    : 0;

  return (
    <div className="flex flex-col items-center justify-center h-screen bg-gray-950 text-white gap-6 p-8">
      <h1 className="text-2xl font-bold">Qwen3.5-0.8B</h1>
      <p className="text-gray-400 text-sm text-center max-w-sm">
        The model weights (~600 MB, Q4 quantized) will be downloaded once and
        cached locally for offline use.
      </p>

      {!downloading ? (
        <button
          onClick={onDownload}
          className="px-6 py-3 bg-blue-600 hover:bg-blue-500 rounded-lg font-semibold transition"
        >
          Download Model
        </button>
      ) : (
        <div className="w-full max-w-md space-y-3">
          <div className="text-xs text-gray-400">
            File {(downloadProgress?.fileIndex ?? 0) + 1} of{" "}
            {downloadProgress?.fileCount ?? "…"}:{" "}
            {downloadProgress?.fileName ?? ""}
          </div>
          <div className="w-full bg-gray-800 rounded-full h-3">
            <div
              className="bg-blue-500 h-3 rounded-full transition-all"
              style={{ width: `${overallPct}%` }}
            />
          </div>
          <div className="text-xs text-gray-500 text-right">{overallPct}%</div>
        </div>
      )}
    </div>
  );
}
```

---

### ChatScreen Component (`src/components/ChatScreen.tsx`)

```typescript
import React, { useRef, useEffect, useState } from "react";
import { useChatStore } from "../stores/chatStore";
import { useSettingsStore } from "../stores/settingsStore";
import { useModelStore } from "../stores/modelStore";
import { MessageBubble } from "./MessageBubble";
import { StatusBar } from "./StatusBar";
import { SettingsDrawer } from "./SettingsDrawer";

interface Props {
  worker: Worker;
}

export function ChatScreen({ worker }: Props) {
  const [input, setInput] = useState("");
  const [thinkingEnabled, setThinkingEnabled] = useState(true);
  const [settingsOpen, setSettingsOpen] = useState(false);
  const bottomRef = useRef<HTMLDivElement>(null);

  const { messages, addMessage, appendToken, finaliseMessage, isGenerating } =
    useChatStore();
  const { maxNewTokens, temperature, topP, repetitionPenalty } =
    useSettingsStore();
  const { setTPS } = useModelStore();

  const currentAssistantId = useRef<string | null>(null);
  const tokenCount = useRef(0);
  const genStart = useRef(0);

  // Wire worker → store
  useEffect(() => {
    const handler = (e: MessageEvent) => {
      const msg = e.data;
      if (msg.type === "token" && currentAssistantId.current) {
        appendToken(currentAssistantId.current, msg.payload);
        tokenCount.current++;
        setTPS(
          Math.round(tokenCount.current / ((Date.now() - genStart.current) / 1000))
        );
      } else if (msg.type === "done" && currentAssistantId.current) {
        finaliseMessage(currentAssistantId.current);
        currentAssistantId.current = null;
      }
    };
    worker.addEventListener("message", handler);
    return () => worker.removeEventListener("message", handler);
  }, [worker]);

  // Auto-scroll to bottom
  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  function send() {
    if (!input.trim() || isGenerating) return;
    const userText = input.trim();
    setInput("");

    addMessage({ role: "user", content: userText });

    const assistantId = addMessage({
      role: "assistant",
      content: "",
      isStreaming: true,
    });
    currentAssistantId.current = assistantId;
    tokenCount.current = 0;
    genStart.current = Date.now();

    // Build history for context (exclude streaming placeholder)
    const history = useChatStore.getState().messages
      .filter((m) => !m.isStreaming && m.id !== assistantId)
      .map((m) => ({ role: m.role, content: m.content }));

    worker.postMessage({
      type: "generate",
      payload: {
        messages: history,
        maxNewTokens,
        temperature,
        topP,
        repetitionPenalty,
        thinkingEnabled,
      },
    });
  }

  function stop() {
    worker.postMessage({ type: "abort" });
  }

  return (
    <div className="flex flex-col h-screen bg-gray-950 text-white">
      {/* Toolbar */}
      <div className="flex items-center justify-between px-4 py-2 border-b border-gray-800 text-sm">
        <span className="font-semibold">Qwen3.5-0.8B</span>
        <div className="flex items-center gap-3">
          <button
            onClick={() => setThinkingEnabled((v) => !v)}
            className={`text-xs px-2 py-1 rounded ${
              thinkingEnabled ? "bg-purple-700" : "bg-gray-700"
            }`}
          >
            Thinking: {thinkingEnabled ? "ON" : "OFF"}
          </button>
          <button
            onClick={() => setSettingsOpen(true)}
            className="text-gray-400 hover:text-white"
          >
            ⚙
          </button>
        </div>
      </div>

      {/* Messages */}
      <div className="flex-1 overflow-y-auto px-4 py-4 space-y-4">
        {messages.map((m) => (
          <MessageBubble key={m.id} message={m} />
        ))}
        <div ref={bottomRef} />
      </div>

      {/* Input */}
      <div className="flex items-center gap-2 px-4 py-3 border-t border-gray-800">
        <textarea
          rows={1}
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => {
            if (e.key === "Enter" && !e.shiftKey) {
              e.preventDefault();
              send();
            }
          }}
          placeholder="Type a message…"
          disabled={isGenerating}
          className="flex-1 bg-gray-800 rounded-lg px-3 py-2 text-sm resize-none outline-none"
        />
        {isGenerating ? (
          <button
            onClick={stop}
            className="px-4 py-2 bg-red-600 hover:bg-red-500 rounded-lg text-sm font-medium"
          >
            Stop
          </button>
        ) : (
          <button
            onClick={send}
            disabled={!input.trim()}
            className="px-4 py-2 bg-blue-600 hover:bg-blue-500 disabled:opacity-40 rounded-lg text-sm font-medium"
          >
            Send
          </button>
        )}
      </div>

      <StatusBar />
      {settingsOpen && <SettingsDrawer onClose={() => setSettingsOpen(false)} />}
    </div>
  );
}
```

---

### StatusBar Component (`src/components/StatusBar.tsx`)

```typescript
import React from "react";
import { useModelStore } from "../stores/modelStore";

export function StatusBar() {
  const { device, tokensPerSecond } = useModelStore();
  return (
    <div className="px-4 py-1 bg-gray-900 border-t border-gray-800 text-xs text-gray-500 flex gap-4">
      <span>{device ? device.toUpperCase() : "—"}</span>
      <span>{tokensPerSecond > 0 ? `${tokensPerSecond} tok/s` : ""}</span>
    </div>
  );
}
```

---

### Settings Store (`src/stores/settingsStore.ts`)

```typescript
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface SettingsState {
  maxNewTokens: number;
  temperature: number;
  topP: number;
  repetitionPenalty: number;
  setMaxNewTokens: (v: number) => void;
  setTemperature: (v: number) => void;
  setTopP: (v: number) => void;
  setRepetitionPenalty: (v: number) => void;
}

export const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      maxNewTokens: 512,
      temperature: 0.7,
      topP: 0.9,
      repetitionPenalty: 1.1,
      setMaxNewTokens: (v) => set({ maxNewTokens: v }),
      setTemperature: (v) => set({ temperature: v }),
      setTopP: (v) => set({ topP: v }),
      setRepetitionPenalty: (v) => set({ repetitionPenalty: v }),
    }),
    { name: "qwen-settings" }
  )
);
```

---

### package.json (key dependencies)

```json
{
  "name": "qwen3508b-desktop",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "tauri": "tauri",
    "tauri:build": "tauri build"
  },
  "dependencies": {
    "@huggingface/transformers": "^3.5.0",
    "@tauri-apps/api": "^2.0.0",
    "@tauri-apps/plugin-fs": "^2.0.0",
    "@tauri-apps/plugin-http": "^2.0.0",
    "@tauri-apps/plugin-path": "^2.0.0",
    "onnxruntime-web": "^1.21.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "zustand": "^5.0.0"
  },
  "devDependencies": {
    "@tauri-apps/cli": "^2.0.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@vitejs/plugin-react": "^4.0.0",
    "internal-ip": "^7.0.0",
    "tailwindcss": "^4.0.0",
    "typescript": "^5.0.0",
    "vite": "^6.0.0",
    "vite-plugin-static-copy": "^1.0.0"
  }
}
```

---

### Data Models

```typescript
// ChatMessage
{
  id: string,                          // "msg-{timestamp}-{counter}"
  role: "user" | "assistant",
  content: string,                     // Display text (no <think> tags)
  thinking?: string,                   // Extracted chain-of-thought text
  isStreaming?: boolean,               // True during active generation
  timestamp: number                    // Unix ms
}

// DownloadProgress (from Rust event)
{
  file: string,                        // Local filename being downloaded
  downloaded: number,                  // Bytes received for this file
  total: number,                       // Total bytes for this file
  file_index: number,                  // 0-based index in file list
  file_count: number                   // Total files to download
}

// ModelManifest (written to disk)
{
  model: string,                       // "onnx-community/Qwen3.5-0.8B-ONNX"
  quantization: string,                // "q4"
  downloaded_at: string                // ISO 8601
}

// GenerationSettings
{
  maxNewTokens: number,                // 1–2048, default 512
  temperature: number,                 // 0.0–2.0, default 0.7
  topP: number,                        // 0.1–1.0, default 0.9
  repetitionPenalty: number            // 1.0–2.0, default 1.1
}
```

---

### Build & Distribution Steps

**Prerequisites:**
- Node.js ≥ 20
- Rust stable toolchain (`rustup update stable`)
- Tauri CLI v2 (`cargo install tauri-cli --version "^2.0"`)
- Windows SDK (for MSI bundling)

**Development:**
```bash
npm install
npm run tauri dev
```

**Production MSI Build:**
```bash
npm run tauri build
# Output: src-tauri/target/release/bundle/msi/Qwen3508B Desktop_0.1.0_x64_en-US.msi
```

**Important build notes:**
- The MSI ships without model weights. The download happens at runtime.
- Tauri's `protocol-asset` feature must be enabled so `convertFileSrc` can serve local files through `asset://localhost/`.
- The `--target x86_64-pc-windows-msvc` flag may be required on machines with multiple Rust targets.
- Code-sign the MSI with `signtool.exe` before distribution to avoid Windows SmartScreen warnings.

---

## Style Guide

- **Background:** `gray-950` (`#030712`)
- **Surface:** `gray-900` / `gray-800`
- **Accent:** `blue-600` (`#2563eb`)
- **Thinking block:** `purple-950` bg, `purple-200` text
- **Font:** System sans-serif stack; monospace for thinking blocks
- **Animations:** Streaming cursor blinks at 1 Hz via CSS `animate-pulse`
- **Border radius:** `rounded-lg` (8 px) throughout
- **Transitions:** `transition` (150 ms ease) on interactive elements

---

## Performance Goals

- Model load time: ≤ 10 seconds on WebGPU (NVIDIA RTX 3060+)
- First token latency: ≤ 2 seconds after send
- Streaming throughput: ≥ 20 tok/s on WebGPU; ≥ 5 tok/s on WASM
- Download resume: not required (files are small enough to re-download on failure)
- MSI installer size: ≤ 10 MB (weights excluded)

---

## Accessibility Requirements

- All interactive controls have `aria-label` attributes.
- Input textarea has `aria-multiline="true"` and `aria-placeholder`.
- Progress bar uses `role="progressbar"` with `aria-valuenow` and `aria-valuemax`.
- Keyboard navigation: Tab order follows visual reading order; Enter submits; Escape closes drawers.
- Minimum contrast ratio 4.5:1 for all text against backgrounds.

---

## Testing Scenarios

1. **Clean install:** Delete `{app_cache_dir}/qwen3508b-onnx/`, launch app → download flow initiates.
2. **Cached install:** With manifest and files present, launch → skips to loading screen.
3. **Partial cache:** Delete one model file, launch → cache check fails, re-download initiates.
4. **Generation abort:** Start a long generation → click Stop → generation halts, partial text retained.
5. **Thinking mode toggle:** Enable thinking, send a maths question → reasoning block shown. Disable → no `<think>` block appears.
6. **WASM fallback:** Disable WebGPU via browser flag / unsupported GPU → app loads using WASM, footer shows `WASM`.
7. **Settings persistence:** Change `temperature` to 1.5, close and reopen app → setting retained via Zustand `persist`.
8. **Long conversation:** Send 20 messages → token context is correctly truncated to model's max context (32 768 tokens) without crash.
