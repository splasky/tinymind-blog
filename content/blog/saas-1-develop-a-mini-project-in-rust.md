---
title: SaaS-1. Develop a mini project in rust
date: 2026-04-17T07:14:32.953Z
---

新的專案除了串街金流外為了取得使用者信任，我將核心功能編譯成WASM並簽章，在User的瀏覽器端執行，系統部署在Cloudflare worker。整個專案只使用Rust開發(Frontend使用leptos)。

### 小收穫
* Rust + WASM該如何在vscode中除錯
* 比較目前已知的serverless平台收費，技術限制，細節差異
* 開發工具只用vscode與claude code, Rust 工具鍊
* 如果不是為了WASM，還是用傳統的Backend+js開發會比較快

### Debugging Guide

This project compiles three separate Rust crates targeting `wasm32-unknown-unknown`:

- **`worker/`** — Cloudflare Worker (runs in `workerd` locally via Wrangler)
- **`frontend/`** — Leptos app (runs in the browser, built by Trunk)
- **`core/`** + **`wasm-bridge/`** — conversion library exposed to JS

All three are debuggable from VSCode. Required extensions:

- `rust-lang.rust-analyzer`
- `ms-vscode.js-debug` (Chrome/Edge debugger, built-in)

---

#### 1. Debug the backend (Cloudflare Worker)

The Worker runs inside `workerd` — you cannot attach a native Rust debugger. Practical strategies:

#### A. Run with `wrangler dev` from VSCode

Use the launch config **`Debug: Wrangler Dev`** (already in [.vscode/launch.json](.vscode/launch.json)). It runs the `wrangler dev debug` task and opens Chrome pointed at `http://localhost:8788`.

```text
F5 → select "Debug: Wrangler Dev"
```

#### B. Use `console_log!` for tracing

The worker runtime only gives you logs. Sprinkle these anywhere in `worker/src/`:

```rust
console_log!("user_id = {:?}, balance = {}", user_id, balance_cents);
```

Logs appear in the VSCode terminal where `wrangler dev` is running.

#### C. Step through with Chrome DevTools

1. Start the compound task **`Debug API: price endpoint`** (or any of the "Debug API" compounds)
2. In the opened Chrome window, press `F12` → **Sources** tab
3. Wrangler exposes the worker WASM under `wasm://` — set breakpoints on the JS shim or view raw WAT
4. Trigger the endpoint via the auto-launched `curl` task

> **Note**: Rust→WASM in Cloudflare Workers does not produce DWARF source maps. You'll see WAT instructions, not `.rs` lines. For line-level debugging, run core logic as a native unit test (see section 4).

#### D. Inspect requests without a debugger

The bundled curl configs in `launch.json` let you fire requests with one click:

- `Test: Auth - send-code`
- `Test: Auth - login (admin)`
- `Test: Get public price`
- `Test: Get OpenAPI spec`
- `Test: Docs page`

Pair any of these with `Debug: Wrangler Dev` via a **compound** configuration.

#### E. Database inspection

Local D1 state lives in `.wrangler/state/v3/d1/`. Query it with:

```bash
npx wrangler d1 execute datastore-convert-db --local --command "SELECT * FROM users"
```

Add a new task in [.vscode/tasks.json](.vscode/tasks.json) if you want it one-click.

---

### 2. Debug the frontend (Leptos)

The frontend is CSR Leptos compiled to WASM. Source maps are limited (same as the worker), so most debugging is through browser DevTools + `web_sys::console`.

#### A. Start the dev server

Run `wrangler dev --env dev` (the same Worker serves the compiled frontend from `frontend/dist/`). Launch it via:

```text
Terminal: npx wrangler dev --env dev
```

Or use the **`Debug: Wrangler Dev`** launch config, which also opens Chrome.

#### B. Log from Rust to the browser console

```rust
use web_sys::console;
console::log_1(&format!("upload_key = {:?}", upload_key.get()).into());
```

Output appears in Chrome DevTools Console.

Alternative with `log`-style macros (define once in `frontend/src/lib.rs`):

```rust
macro_rules! console_log {
    ($($t:tt)*) => (web_sys::console::log_1(&format!($($t)*).into()))
}
```

#### C. Inspect reactive state

Leptos signals are plain Rust values — you cannot introspect them from DevTools. Either:

- Log when a signal is updated: `Effect::new(move |_| console_log!("status = {}", status.get()));`
- Use `#[cfg(debug_assertions)]` blocks to render extra debug text in the view

#### D. Network tab

DevTools → **Network** tab is the fastest way to see why `/api/convert-local` is returning 402 or 403. It shows request/response bodies directly.

#### E. Set breakpoints in JS glue

`wasm-bindgen` generates JS shim files. Chrome can stop on the shim (`frontend/dist/*.js`), which gives you the boundary where Rust is called from JS.

---

### 3. Debug the core WASM (`wasm-bridge`)

The `wasm-bridge` crate exposes `core` to vanilla JS. It builds to `wasm-bridge-pkg/datastore_convert_wasm_bg.wasm`.

#### A. Build with debug symbols

```bash
wasm-pack build wasm-bridge --target web --dev --out-dir ../wasm-bridge-pkg --out-name datastore_convert_wasm
```

`--dev` keeps DWARF info. The WASM file will be ~10× larger but Chrome can map instructions back to Rust lines.

#### B. Enable Chrome's DWARF plugin

1. Install the [C/C++ DevTools Support (DWARF)](https://goo.gle/wasm-debugging-extension) Chrome extension
2. DevTools → **Sources** → WASM source appears under the origin as real `.rs` files
3. Set breakpoints inside `core/src/parser.rs`, `core/src/sqlite_writer.rs`, etc.

#### C. Verify signature and build hash

The compiled WASM embeds `SIGNATURE` and `BUILD_HASH` as string constants:

```bash
strings wasm-bridge-pkg/datastore_convert_wasm_bg.wasm | grep -E "HYChang|Build-Hash"
```

From JS:

```js
import init, { signature, build_hash } from './datastore_convert_wasm.js';
await init();
console.log(signature(), build_hash());
```

#### D. Call `wasm-bridge` from a standalone HTML harness

Quickest way to isolate a conversion bug is a tiny test page that loads the bridge without the Leptos app:

```html
<script type="module">
  import init, { convert } from './wasm-bridge-pkg/datastore_convert_wasm.js';
  await init();
  const file = document.querySelector('input[type=file]').files[0];
  const bytes = new Uint8Array(await file.arrayBuffer());
  const sql = convert(bytes);
  console.log(new TextDecoder().decode(sql));
</script>
```

Serve it via `python3 -m http.server` and open in Chrome with DevTools.

---

### 4. Debug `core` as a native unit test

For anything that does not require the browser (parser logic, SQLite writer, format detection), skip WASM entirely:

```bash
# Build for host instead of WASM
cargo test -p datastore-convert-core --target x86_64-unknown-linux-gnu
```

This gives you:

- Full `println!` / `dbg!` output
- Native step-through with `rust-analyzer`'s **Debug** lens (click "Debug" above any `#[test]`)
- Real DWARF → VSCode can stop on any line

Use the **CodeLLDB** extension if you want integrated native debugging.

---

### 5. Breakpoint support matrix

Not every Rust line can accept a breakpoint — it depends on where the code runs and whether debug info is preserved.

### Where breakpoints work

| Target | Tool | Requirement |
|--------|------|-------------|
| `core/` unit tests (native) | CodeLLDB / rust-analyzer Debug lens | `cargo test --target x86_64-*` — full DWARF |
| `wasm-bridge/` `.rs` lines | Chrome DevTools + DWARF extension | `wasm-pack build --dev` (debug info retained) |
| `frontend/` `.rs` lines | Chrome DevTools + DWARF extension | Trunk built in debug profile (not release) |
| JS glue / wasm-bindgen shim | Chrome DevTools Sources | Always — plain JS |
| HTTP request/response | DevTools Network tab | Always — not a breakpoint, but shows full payloads |

#### Where breakpoints do NOT work

| Target | Reason |
|--------|--------|
| `worker/` `.rs` lines | `workerd` + wrangler do not expose DWARF mapping. Only WAT is visible, no source-level stepping. |
| Release-built frontend WASM | Trunk release profile strips debug info. WAT without symbols. |
| Release-built `wasm-bridge` | `wasm-pack build --release` omits DWARF. |
| Leptos signal updates | Reactive state lives in Rust memory; DevTools cannot introspect it. Use `Effect::new` to log on change. |

#### Workarounds when breakpoints are unavailable

| Scenario | Alternative |
|----------|-------------|
| Bug in Worker route logic | Extract logic into `core/`, write a `#[test]` → native breakpoints |
| Bug in frontend flow | `web_sys::console::log_1` + `Effect::new` to trace signals |
| Bug in conversion algorithm | `cargo test -p datastore-convert-core` → real debugger |
| Want to see WASM call order | DevTools **Performance** tab → record call graph |

**Rule of thumb**: anything you can pull into `core/` and cover with a unit test gets full native debugger support. UI flows (frontend) and route handlers (worker) are `console_log!` + Network tab + the Chrome DWARF extension.

---

### 6. Common issues

| Symptom | Likely cause |
|---------|--------------|
| `cargo check` errors on `tokio` / non-WASM crates | `rust-analyzer.cargo.target` is set to `wasm32-unknown-unknown` in [.vscode/settings.json](.vscode/settings.json). Temporarily remove it to analyze the host target. |
| Breakpoints not hit in Chrome | Build with `--dev`, install the DWARF extension, and hard-reload (Ctrl+Shift+R). |
| `console_log!` output missing | You're looking at the wrong terminal. Logs from the Worker appear where `wrangler dev` is running, not in Chrome. |
| Cached WASM after rebuild | Open DevTools → **Application** → **Storage** → **Clear site data**, or hard-reload. |
| `/api/convert-local` returns 402 | Insufficient balance. Top up via the dashboard or update the user's `balance_cents` directly in local D1. |

