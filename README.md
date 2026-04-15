# Debugging ESP-IDF CMake in VSCode / Cursor / Command Line

CMake 3.27+ supports a built-in debugger (DAP protocol) that integrates with
VSCode via the **CMake Tools** extension. CMake also provides powerful
command-line trace options for debugging without any IDE at all. This document
covers both approaches for the ESP-IDF build system — including the main
configure phase and the early expansion phase.

## Prerequisites

- CMake >= 3.27 (verify with `cmake --version`)
- ESP-IDF environment sourced (`. ./export.sh`) so toolchain paths are on `PATH`
- For VSCode/Cursor debugging: the **CMake Tools** extension

## Background: How `idf.py build` invokes CMake

When you run `idf.py build`, it translates to something like:

```bash
cmake -G Ninja \
  -B <project>/build \
  -DPYTHON_DEPS_CHECKED=1 \
  -DPYTHON=<path-to-idf-python> \
  -DESP_PLATFORM=1 \
  -DCCACHE_ENABLE=False \
  <project-source-dir>
```

This is a normal CMake **project configure**, not script mode. The debugger must
attach to this kind of invocation.

## Background: Two-phase CMake in ESP-IDF

ESP-IDF's build system has **two distinct CMake phases**:

1. **Main configure** — the parent `cmake` process that runs `project.cmake`,
   `idf.cmake`, `build.cmake`, `kconfig.cmake`, component `project_include.cmake`
   files, etc.

2. **Early expansion** — a **separate child** `cmake -P` process spawned via
   `execute_process()` in `tools/cmake/component.cmake`. This child runs
   `tools/cmake/scripts/component_get_requirements.cmake`, which sets
   `CMAKE_BUILD_EARLY_EXPANSION=1` and `include()`s every component's
   `CMakeLists.txt` to collect `REQUIRES` / `PRIV_REQUIRES`.

Because early expansion runs in a child process, **a debugger attached to the
parent cannot see it**. Each phase needs its own debugger connection.

## Configuration files

### `.vscode/launch.json`

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "cmake",
            "request": "launch",
            "name": "Debug IDF CMake Configure",
            "cmakeDebugType": "configure",
            "clean": true
        },
        {
            "type": "cmake",
            "request": "launch",
            "name": "Debug IDF CMake (external)",
            "cmakeDebugType": "external",
            "pipeName": "/tmp/cmake-debugger-pipe"
        },
        {
            "type": "cmake",
            "request": "launch",
            "name": "Debug IDF Early Expansion (external)",
            "cmakeDebugType": "external",
            "pipeName": "/tmp/cmake-early-debugger"
        }
    ]
}
```

### `.vscode/settings.json` (for the `configure` debug type)

The `configure` type uses CMake Tools' own configure mechanism. These settings
replicate the flags that `idf.py` would pass:

```json
{
    "cmake.sourceDirectory": "${workspaceFolder}/examples/get-started/blink",
    "cmake.buildDirectory": "${workspaceFolder}/examples/get-started/blink/build",
    "cmake.generator": "Ninja",
    "cmake.configureArgs": [
        "-DPYTHON_DEPS_CHECKED=1",
        "-DPYTHON=${env:HOME}/.espressif/python_env/idf6.1_py3.12_env/bin/python",
        "-DESP_PLATFORM=1",
        "-DCCACHE_ENABLE=False"
    ],
    "cmake.configureEnvironment": {
        "IDF_PATH": "${workspaceFolder}"
    }
}
```

Change `cmake.sourceDirectory` and `cmake.buildDirectory` to point to
whichever example project you want to debug.

---

## Phase 1: Debugging the main configure

You can debug the main configure phase in two ways.

### Option A: `configure` type (simplest)

This uses CMake Tools to drive the configure step with the debugger attached.
Requires `.vscode/settings.json` to be set up (see above).

1. Set breakpoints in CMake files (e.g. `tools/cmake/project.cmake`,
   `tools/cmake/build.cmake`, a component's `project_include.cmake`).
2. Select **"Debug IDF CMake Configure"** in the Run & Debug dropdown.
3. Press **F5**. The configure starts and stops at your breakpoints.

### Option B: `external` type (full control)

This gives you the exact same cmake command line as `idf.py build`, with the
debugger pipe injected. Use this when you need precise control over the
invocation or when the `configure` type doesn't behave correctly.

**Step 1 — Start the debugger listener in VSCode:**

Select **"Debug IDF CMake (external)"** in the Run & Debug dropdown and press
F5. VSCode opens the pipe `/tmp/cmake-debugger-pipe` and waits for a cmake
process to connect.

**Step 2 — Run cmake from the terminal (with export.sh sourced):**

```bash
cmake -G Ninja \
  -B examples/get-started/blink/build \
  -DPYTHON_DEPS_CHECKED=1 \
  -DPYTHON=$HOME/.espressif/python_env/idf6.1_py3.12_env/bin/python \
  -DESP_PLATFORM=1 \
  -DCCACHE_ENABLE=False \
  --debugger \
  --debugger-pipe /tmp/cmake-debugger-pipe \
  examples/get-started/blink
```

The `--debugger` and `--debugger-pipe` flags tell cmake to connect to the
waiting VSCode debugger. Breakpoints in any CMake file hit during the main
configure phase will fire.

**Step 3 — Debug.** Step through, inspect variables, etc. When done, the cmake
process exits and the debug session ends.

---

## Phase 2: Debugging early expansion

Early expansion runs as a child `cmake -P` process spawned from within the main
configure. To debug it, you inject `--debugger` flags into that child process.

### Patch `tools/cmake/component.cmake`

In the `__component_get_requirements()` function (~line 221), the
`execute_process` call launches the child cmake. Add the debugger flags:

```cmake
    execute_process(COMMAND "${CMAKE_COMMAND}"
        -D "ESP_PLATFORM=1"
        -D "BUILD_PROPERTIES_FILE=${build_properties_file}"
        -D "COMPONENT_PROPERTIES_FILE=${component_properties_file}"
        -D "COMPONENT_REQUIRES_FILE=${component_requires_file}"
        --debugger
        --debugger-pipe /tmp/cmake-early-debugger
        -P "${idf_path}/tools/cmake/scripts/component_get_requirements.cmake"
        RESULT_VARIABLE result
        ERROR_VARIABLE error)
```

When not debugging, comment them out (or revert the change), otherwise
builds will hang waiting for a debugger connection:

```cmake
        # --debugger
        # --debugger-pipe /tmp/cmake-early-debugger
```

### Workflow

**Step 1 — Start the debugger listener in VSCode:**

Select **"Debug IDF Early Expansion (external)"** and press F5. VSCode opens
`/tmp/cmake-early-debugger` and waits.

**Step 2 — Set breakpoints:**

Set breakpoints inside component `CMakeLists.txt` files — specifically in code
guarded by `if(CMAKE_BUILD_EARLY_EXPANSION)`. You can also set breakpoints in
`tools/cmake/scripts/component_get_requirements.cmake` itself.

**Step 3 — Run the build from the terminal:**

```bash
idf.py build
```

Or run the raw cmake command (without `--debugger` on the parent, unless you
want to debug both phases):

```bash
cmake -G Ninja \
  -B examples/get-started/blink/build \
  -DPYTHON_DEPS_CHECKED=1 \
  -DPYTHON=$HOME/.espressif/python_env/idf6.1_py3.12_env/bin/python \
  -DESP_PLATFORM=1 \
  -DCCACHE_ENABLE=False \
  examples/get-started/blink
```

The parent cmake runs normally until it hits `execute_process()`, which spawns
the child cmake with `--debugger`. The child connects to the waiting VSCode
debugger. Your breakpoints fire.

**Step 4 — Debug.** When the early expansion child finishes, the debug session
ends and the parent cmake continues with the rest of the configure.

---

## Debugging both phases simultaneously

To debug both the main configure and early expansion in the same build, you need
**two separate debugger connections** (two different pipes). This requires two
VSCode windows or two debug sessions:

1. In one VSCode window, start **"Debug IDF CMake (external)"**
   (pipe: `/tmp/cmake-debugger-pipe`).
2. In another VSCode window, start **"Debug IDF Early Expansion (external)"**
   (pipe: `/tmp/cmake-early-debugger`).
3. Ensure the `--debugger` lines are uncommented in `component.cmake`.
4. Run cmake with both parent debugger flags:

```bash
cmake -G Ninja \
  -B examples/get-started/blink/build \
  -DPYTHON_DEPS_CHECKED=1 \
  -DPYTHON=$HOME/.espressif/python_env/idf6.1_py3.12_env/bin/python \
  -DESP_PLATFORM=1 \
  -DCCACHE_ENABLE=False \
  --debugger \
  --debugger-pipe /tmp/cmake-debugger-pipe \
  examples/get-started/blink
```

The parent connects to the first debugger; when it spawns the child, the child
connects to the second debugger.

---

---

## Command-line debugging (no IDE required)

CMake's `--debugger` uses the Debug Adapter Protocol (DAP), **not** GDB's
protocol. You cannot attach GDB, LLDB, or similar tools to a cmake process —
they speak entirely different protocols. However, there are good alternatives
for debugging from the command line.

### Option 1: `cmake --trace` (printf-style, always available)

CMake has built-in tracing that prints every command as it executes. This is the
most reliable CLI debugging method and requires no extra tools.

**Basic trace** — prints every cmake command and its source location:

```bash
cmake -G Ninja \
  -B examples/get-started/blink/build \
  -DPYTHON_DEPS_CHECKED=1 \
  -DPYTHON=$HOME/.espressif/python_env/idf6.1_py3.12_env/bin/python \
  -DESP_PLATFORM=1 \
  -DCCACHE_ENABLE=False \
  --trace \
  examples/get-started/blink \
  2>trace.log
```

**Trace with variable expansion** — shows actual values instead of
`${variable}` names:

```bash
cmake ... --trace-expand ... 2>trace.log
```

**Trace a specific file only** — avoids the massive output by filtering to one
file. Multiple `--trace-source` flags can be combined:

```bash
cmake ... \
  --trace-source=tools/cmake/project.cmake \
  --trace-source=components/esp_wifi/CMakeLists.txt \
  ... 2>trace.log
```

**Redirect trace to a file** (instead of stderr):

```bash
cmake ... --trace --trace-redirect=trace.log ...
```

**JSON output** for structured processing:

```bash
cmake ... --trace --trace-format=json-v1 --trace-redirect=trace.json ...
```

Each line in the JSON output is a separate document:

```json
{
  "file": "/home/user/dev/idf/tools/cmake/project.cmake",
  "line": 29,
  "cmd": "include",
  "args": ["/home/user/dev/idf/tools/cmake/idf.cmake"],
  "time": 1579512535.968,
  "frame": 2
}
```

**Watch a specific variable** — add `variable_watch(<varname>)` in any cmake
file to print a message whenever that variable is read or modified:

```cmake
variable_watch(CMAKE_BUILD_EARLY_EXPANSION)
variable_watch(IDF_TARGET)
```

### Tracing early expansion from the CLI

The `--trace` flags are inherited by the parent cmake process but **not** by
the child `cmake -P` process that runs early expansion (it's a separate
`execute_process` invocation). To trace early expansion, temporarily add the
trace flags to the `execute_process` call in `tools/cmake/component.cmake`:

```cmake
    execute_process(COMMAND "${CMAKE_COMMAND}"
        -D "ESP_PLATFORM=1"
        -D "BUILD_PROPERTIES_FILE=${build_properties_file}"
        -D "COMPONENT_PROPERTIES_FILE=${component_properties_file}"
        -D "COMPONENT_REQUIRES_FILE=${component_requires_file}"
        --trace-expand
        --trace-redirect ${build_dir}/early_expansion_trace.log
        -P "${idf_path}/tools/cmake/scripts/component_get_requirements.cmake"
        RESULT_VARIABLE result
        ERROR_VARIABLE error)
```

Then run a normal `idf.py build` and inspect `build/early_expansion_trace.log`.

### Option 2: dcmake (standalone GUI debugger)

[dcmake](https://github.com/skeeto/dcmake) is a standalone CMake debugger GUI
(built with Dear ImGui) that acts as a DAP client without requiring VSCode. It
provides breakpoints, stepping, and variable inspection in a lightweight window.

Install and run it by prepending `d` to your cmake command:

```bash
dcmake -G Ninja \
  -B examples/get-started/blink/build \
  -DPYTHON_DEPS_CHECKED=1 \
  -DPYTHON=$HOME/.espressif/python_env/idf6.1_py3.12_env/bin/python \
  -DESP_PLATFORM=1 \
  -DCCACHE_ENABLE=False \
  examples/get-started/blink
```

Keybindings: **F5** continue, **F10** step over, **F11** step in, click line
numbers to set breakpoints, hover over variables to inspect.

### Option 3: Any DAP client + `--debugger-pipe`

Since cmake's `--debugger` uses the standard Debug Adapter Protocol over a Unix
domain socket, any DAP-capable client can connect. This includes:

- **Neovim** with [nvim-dap](https://github.com/mfussenegger/nvim-dap)
- **Emacs** with [dap-mode](https://github.com/emacs-lsp/dap-mode)

The setup is the same as the VSCode `external` approach — the client listens on
a socket, then you run cmake with `--debugger --debugger-pipe <socket>`.

Example DAP configuration for nvim-dap (in `init.lua`):

```lua
local dap = require('dap')
dap.adapters.cmake = {
  type = 'pipe',
  pipe = '/tmp/cmake-debugger-pipe',
}
dap.configurations.cmake = {
  {
    type = 'cmake',
    request = 'launch',
    name = 'Debug CMake',
  },
}
```

Then start the DAP session in neovim, and run cmake with `--debugger
--debugger-pipe /tmp/cmake-debugger-pipe` from another terminal.

### `--debugger-dap-log` for protocol-level debugging

If you need to understand exactly what the DAP client and cmake are saying to
each other, log all DAP messages:

```bash
cmake ... --debugger --debugger-pipe /tmp/cmake-debugger-pipe \
  --debugger-dap-log /tmp/cmake-dap.log ...
```

---

## Why `cmakeDebugType: "script"` doesn't work for IDF

The `script` debug type runs `cmake -P <file>`, which is CMake's script mode.
Script mode has **no project context** — no `project()` command, no directory
properties, no generators, no toolchain files. Since ESP-IDF's build system
relies on all of these (`define_property(DIRECTORY ...)`, `project()`,
toolchain selection, etc.), script mode fails immediately with errors like:

```
define_property(DIRECTORY PROPERTY "EP_BASE" INHERITED)
  -- not scriptable
```

The `configure` and `external` types run a real project configure, which is what
IDF needs.

## Quick reference

| What to debug | Method | How |
|---|---|---|
| Main configure (easy) | VSCode `configure` | Press F5 with "Debug IDF CMake Configure" |
| Main configure (full control) | VSCode `external` | Start debug session, then `cmake ... --debugger --debugger-pipe /tmp/cmake-debugger-pipe ...` |
| Early expansion | VSCode `external` | Start "Early Expansion" debug session, then `idf.py build` (with component.cmake patched) |
| Main configure (CLI) | `--trace` | `cmake ... --trace-expand --trace-source=<file> ... 2>trace.log` |
| Early expansion (CLI) | `--trace` in component.cmake | Add trace flags to `execute_process`, inspect `build/early_expansion_trace.log` |
| Main configure (standalone GUI) | dcmake | `dcmake ... <same flags as cmake> ...` |
| Any phase | Any DAP client | nvim-dap / emacs dap-mode + `--debugger --debugger-pipe` |

## Useful breakpoint locations

| File | What it does |
|---|---|
| `tools/cmake/project.cmake` | Top-level project setup, target init, `idf_build_process` call |
| `tools/cmake/build.cmake` | Component discovery, dependency resolution, build orchestration |
| `tools/cmake/component.cmake` | Component registration, requirement gathering |
| `tools/cmake/kconfig.cmake` | Kconfig/sdkconfig processing |
| `components/<name>/CMakeLists.txt` | Individual component build logic |
| `components/<name>/project_include.cmake` | Per-component project-level hooks (main configure only) |
| `tools/cmake/scripts/component_get_requirements.cmake` | Early expansion orchestrator |
