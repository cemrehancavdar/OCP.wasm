# Porting OCP.wasm to Python 3.12 (Pyodide 0.27.x)

This fork of [yeicor/OCP.wasm](https://github.com/yeicor/OCP.wasm) builds
OCP for **Python 3.12** instead of 3.13. The upstream repo targets Pyodide
0.30.x (Python 3.13), but [marimo](https://github.com/marimo-team/marimo)
WASM uses Pyodide 0.27.x (Python 3.12) and cannot upgrade yet because
Pyodide 0.28+ dropped `polars`, `duckdb`, and `pyarrow`
([marimo PR #5995](https://github.com/marimo-team/marimo/pull/5995)).

## Version matrix

| Component        | Upstream (yeicor) | This fork       |
|------------------|-------------------|-----------------|
| pyodide-build    | 0.30.9            | **0.27.3**      |
| pyodide-xbuildenv| 0.29.0            | **0.27.7**      |
| Python           | 3.13              | **3.12**        |
| Emscripten       | 4.0.x             | **3.1.58**      |
| LLVM / wasm-ld   | 19+               | **18**          |
| OCCT             | 7.9.3             | 7.9.3 (same)   |
| OCP bindings     | 7.9.3.0           | 7.9.3.0 (same) |

The single root cause of every change below is **LLVM 18's `wasm-ld` is
stricter than LLVM 19's**. Specifically it does not support
`--allow-multiple-definition`, which the upstream build relies on.

## Changes from upstream (and why)

### 1. `requirements.txt` — target Pyodide 0.27.x

```
pyodide-build==0.27.3
# pyodide-xbuildenv==0.27.7
```

The xbuildenv version is in a comment because it's not pip-installable; it's
parsed by `.github/scripts/xbuildenv_data.sh` via grep.

### 2. `.github/workflows/main.yml` — two small fixes

**`wheel<0.45` pin:** Pyodide 0.27.x's `auditwheel-emscripten` is
incompatible with `wheel>=0.45` (changed internal API). Added
`pip install "wheel<0.45"` before `pyodide-build`.

**Emscripten version parsing:** The upstream uses `pyodide config get
emscripten_version` which in 0.27.3 outputs extra text. Changed to:

```bash
pyodide config get emscripten_version 2>/dev/null | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1
```

### 3. `cadquery-ocp/CMakeLists.txt` — embuilder library names

Emscripten 3.1.58 uses old library names. The upstream had
`-legacysjlj` suffixed names (e.g. `libc++-legacysjlj`). Changed to:

```cmake
embuilder build libdlmalloc libcompiler_rt libc++ libc++abi libunwind libc++abi-debug freetype --pic
```

### 4. `cadquery-ocp/patch_OCP.cmake` — remove `--allow-multiple-definition`

The OCP source ZIP contains at line 66 of its `CMakeLists.txt`:

```cmake
target_link_options(OCP PRIVATE "LINKER:--allow-multiple-definition")
```

This flag doesn't exist in LLVM 18's `wasm-ld`. Patched out via regex:

```cmake
string(REGEX REPLACE "target_link_options\\([^)]*allow-multiple-definition[^)]*\\)" "" ...)
```

### 5. `cadquery-ocp/CMakeLists.txt` — clear OCCT transitive link deps

**Problem:** CMake's transitive dependency resolution causes each OCCT
static library (`.a`) to appear multiple times in the linker command.
GNU `ld` handles this via archive rescanning. LLVM 18's `wasm-ld` does not
and reports ~40 "duplicate symbol" errors.

**Fix:** After `FetchContent_MakeAvailable(OpenCASCADE)`, clear
`INTERFACE_LINK_LIBRARIES` on all OCCT targets. This is safe because the
OCP build already lists every OCCT library explicitly via
`${OpenCASCADE_LIBRARIES}`.

```cmake
foreach(__TK ${BUILD_TOOLKITS})
  if(TARGET ${__TK})
    set_property(TARGET ${__TK} PROPERTY INTERFACE_LINK_LIBRARIES "")
  endif()
endforeach()
```

### 6. `cadquery-ocp/patch_OpenCASCADE.cmake` — remove DISCRETALGO duplicate

**Problem:** OCCT's `BRepMesh_PluginMacro.hxx` defines a `DISCRETPLUGIN(name)`
macro that creates an `extern "C" DISCRETALGO()` function. Two source files
invoke it:

- `BRepMesh_IncrementalMesh.cxx` (in TKMesh) — `DISCRETPLUGIN(BRepMesh_IncrementalMesh)`
- `XBRepMesh.cxx` (in TKXMesh) — `DISCRETPLUGIN(XBRepMesh)`

This produces two definitions of the same `DISCRETALGO` symbol. The upstream
build handles this with `--allow-multiple-definition`. We can't.

**Fix:** Patch `XBRepMesh.cxx` to comment out the `DISCRETPLUGIN()` call.
The `DISCRETALGO` C export is an OCCT plugin-loading mechanism irrelevant for
static WASM linking. `XBRepMesh::Discret` (the C++ method) remains available.
In practice, OCP bindings don't reference any symbol from `libTKXMesh.a`.

```cmake
string(REGEX REPLACE "DISCRETPLUGIN\\([^)]*\\)" "// DISCRETPLUGIN removed for wasm-ld compatibility" ...)
```

## When this fork becomes unnecessary

This fork can be retired when either:

1. **Marimo upgrades to Pyodide 0.28+** (blocked on `polars`/`duckdb`/`pyarrow`
   being rebuilt for the new ABI — tracked in
   [pyodide-recipes#99](https://github.com/pyodide/pyodide-recipes/issues/99))
2. **Pyodide 0.27.x gets a new xbuildenv** that ships LLVM 19+ (unlikely)

At that point, use the upstream `yeicor/OCP.wasm` wheels directly.

## Local testing

To iterate on linking without waiting for 2-hour CI builds:

1. Download the CI cache artifact (`cache-cadquery-ocp-Debug`)
2. Install emsdk 3.1.58 and `embuilder build freetype --pic`
3. Run the link command manually (see `.agent/journal.md` for the exact invocation)
