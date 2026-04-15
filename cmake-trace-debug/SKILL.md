---
name: cmake-trace-debug
description: >-
  Debug ESP-IDF CMake build system using cmake --trace from the command line.
  Use when investigating IDF cmake build issues, component registration
  problems, dependency resolution, kconfig errors, or early expansion behavior.
  Use when the user mentions cmake debug, cmake trace, build system debug, or
  component CMakeLists.txt issues.
---

# Debug ESP-IDF CMake with `--trace`

## Key concept: IDF has two CMake phases

ESP-IDF runs cmake in **two separate processes** during configure. You must
trace each one differently.

| Phase | Process | What runs | Key variable |
|---|---|---|---|
| Main configure | Parent cmake | `project.cmake`, `build.cmake`, `kconfig.cmake`, component `project_include.cmake` files, second pass of component `CMakeLists.txt` | — |
| Early expansion | Child `cmake -P` (spawned via `execute_process` in `tools/cmake/component.cmake`) | `scripts/component_get_requirements.cmake`, which `include()`s every component `CMakeLists.txt` | `CMAKE_BUILD_EARLY_EXPANSION=1` |

A trace on the parent **does not** capture the child. Each needs its own flags.

## Step 1: Identify the build command

IDF's `idf.py build` translates to:

```bash
cmake -G Ninja \
  -B <project>/build \
  -DPYTHON_DEPS_CHECKED=1 \
  -DPYTHON=<idf-python-path> \
  -DESP_PLATFORM=1 \
  -DCCACHE_ENABLE=False \
  <project-source-dir>
```

Find the actual command by running `idf.py build` and reading the first lines
of output, or check the terminal history. The python path is typically
`$HOME/.espressif/python_env/idf<version>_py<pyver>_env/bin/python`.

## Step 2: Trace the main configure

Add `--trace` flags to the cmake command. Choose the appropriate level:

### Trace a specific file (recommended starting point)

```bash
cmake -G Ninja \
  -B <project>/build \
  -DPYTHON_DEPS_CHECKED=1 -DPYTHON=<path> -DESP_PLATFORM=1 -DCCACHE_ENABLE=False \
  --trace-expand \
  --trace-source=<file-to-trace> \
  --trace-redirect=trace.log \
  <project-source-dir>
```

Multiple `--trace-source` flags can be combined. Common files to trace:

| File | Trace when investigating |
|---|---|
| `tools/cmake/project.cmake` | Project setup, target init, top-level flow |
| `tools/cmake/build.cmake` | Component discovery, dependency resolution |
| `tools/cmake/component.cmake` | Component registration, requirement gathering |
| `tools/cmake/kconfig.cmake` | Kconfig/sdkconfig processing |
| `components/<name>/CMakeLists.txt` | Specific component build logic |
| `components/<name>/project_include.cmake` | Component project-level hooks |

### Full trace (very verbose — redirect to file)

```bash
cmake ... --trace-expand --trace-redirect=trace.log ...
```

### JSON trace (for scripted analysis)

```bash
cmake ... --trace --trace-format=json-v1 --trace-redirect=trace.json ...
```

Each line is a JSON object with `file`, `line`, `cmd`, `args`, `time`, `frame`.
Useful for grepping: `grep '"cmd":"idf_component_register"' trace.json`.

## Step 3: Trace early expansion

The early expansion child process is launched in
`tools/cmake/component.cmake`, function `__component_get_requirements()`.
The `execute_process` call looks like:

```cmake
execute_process(COMMAND "${CMAKE_COMMAND}"
    -D "ESP_PLATFORM=1"
    -D "BUILD_PROPERTIES_FILE=${build_properties_file}"
    -D "COMPONENT_PROPERTIES_FILE=${component_properties_file}"
    -D "COMPONENT_REQUIRES_FILE=${component_requires_file}"
    -P "${idf_path}/tools/cmake/scripts/component_get_requirements.cmake"
    RESULT_VARIABLE result
    ERROR_VARIABLE error)
```

**To trace it**, temporarily add trace flags before the `-P` argument:

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

You can also use `--trace-source` here to filter to a specific component:

```cmake
    --trace-source=${idf_path}/components/esp_wifi/CMakeLists.txt
    --trace-redirect ${build_dir}/early_expansion_trace.log
```

Then run a normal `idf.py build` and read the trace log from the build dir.

**Remember to revert** `component.cmake` after debugging.

## Step 4: Inspect trace output

### Human-readable format

Each line shows `file(line): command(args...)`:

```
/home/user/dev/idf/tools/cmake/project.cmake(29):  include(/home/user/dev/idf/tools/cmake/idf.cmake )
/home/user/dev/idf/tools/cmake/idf.cmake(5):  include(/home/user/dev/idf/tools/cmake/build.cmake )
```

With `--trace-expand`, variables are replaced with values.

### What to look for

- **Component not found**: grep for `__component_get_target` or
  `COMPONENT_TARGETS` to see which components are registered.
- **Wrong REQUIRES/PRIV_REQUIRES**: trace the component's `CMakeLists.txt` in
  early expansion to see what `idf_component_register` receives.
- **Kconfig issues**: trace `kconfig.cmake` to see sdkconfig processing.
- **Build property problems**: trace `build.cmake` and look for
  `idf_build_set_property` / `idf_build_get_property` calls.

## Additional techniques

### `variable_watch()`

Insert into any cmake file to log reads/writes of a variable:

```cmake
variable_watch(IDF_TARGET)
variable_watch(CMAKE_BUILD_EARLY_EXPANSION)
```

### `message()` with call stack

```cmake
message(STATUS "MY_VAR = ${MY_VAR}")
message(FATAL_ERROR "Stop here and print stack trace")
```

Use `FATAL_ERROR` to halt cmake and get a full call stack at any point.

### Inspecting temp files

During configure, IDF writes property snapshots to the build directory:
- `build_properties.temp.cmake` — all build-level properties
- `component_properties.temp.cmake` — all component-level properties
- `component_requires.temp.cmake` — resolved component requirements

Read these after a successful configure to see the build system's internal
state without needing a trace.

## Checklist

- [ ] Identify which phase the problem is in (main configure vs early expansion)
- [ ] Add `--trace` flags to the correct process
- [ ] Use `--trace-source` to narrow output to relevant files
- [ ] Redirect output to a file (`--trace-redirect` or `2>`)
- [ ] Inspect the trace log for the relevant commands
- [ ] Revert any modifications to `component.cmake` after debugging

