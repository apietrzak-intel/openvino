# GPU plugin Level Zero support

OpenVINO GPU plugin can be compiled with **experimental** Level Zero support. When enabled, plugin will use Level Zero as GPU compute API instead of OpenCL.

Level Zero is the better-performing runtime on Xe2 and newer Intel GPU architectures; on pre-Xe2 GPUs, it is not supported.

## How to build with Level Zero support

1. Configure cmake with the following additional option:
    - `-DGPU_RT_TYPE=ZE`
2. Build OpenVINO
    - `cmake --build . --config Release`

### Link-time optimization

Release builds support `-DENABLE_LTO=ON` together with `-DGPU_RT_TYPE=ZE`
(`L0` is an alias for `ZE`). The bundled oneDNN build inherits the LTO setting.
OpenVINO and oneDNN use the same staged Level Zero API headers and Intel GPU
driver-extension headers. The `prepare_ze_headers` target stages these headers
before oneDNN configures, including when building oneDNN directly. This applies
to both bundled and system Level Zero loaders. With `-DENABLE_SYSTEM_LEVEL_ZERO=ON`,
the API headers come from the selected `level-zero` pkg-config package; they must
meet oneDNN's minimum API version requirement (currently 1.11).

To diagnose header or LTO mismatches, inspect the oneDNN configure output for
`Found Level Zero headers:`. This must point
to `<build>/thirdparty/level_zero/include`, not oneDNN's `third_party/level_zero`.
In `<build>/src/plugins/intel_gpu/thirdparty/onednn_gpu_build/CMakeCache.txt`,
check that `DNNL_ZE_INCLUDE_DIR` points to that directory and
`CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE` is `ON`. Use a fresh build directory
when reproducing a mismatch to avoid cached oneDNN configuration checks.

## How to check what runtime is used
### With [DEBUG_CAPS cmake option](gpu_debug_utils.md)
0. Prerequisite: Build OpenVINO with [DEBUG_CAPS cmake option](gpu_debug_utils.md)
1. Set `OV_VERBOSE=1` environment variable.
2. Run workload
3. Find what variant of stream is reported in the logs (`ze_stream` or `ocl_stream`)
    - Example output for Level Zero runtime: `ze_stream: [GPU] Created Level Zero stream`
### Without DEBUG_CAPS
* Check which libraries are loaded at runtime.
  * On Linux, use `strace` to confirm that Level Zero libraries are loaded.
  * On Windows, use `Process Explorer` to confirm that Level Zero libraries are loaded.
  * If you see Level Zero libraries such as `ze_loader.dll` and `ze_intel_gpu64.dll`, Level Zero is in use.
* Use [clintercept](https://github.com/intel/opencl-intercept-layer/tree/main) to check whether any OpenCL calls are made.
  * If no OpenCL calls appear, Level Zero is in use.

## Limitations

* This is an experimental, single-runtime build: the resulting plugin binary talks to the GPU over Level Zero only and cannot fall back to OpenCL.
* To ship both OpenCL and Level Zero runtimes side by side, with automatic per-device selection, see [GPU dual-runtime dispatch (COMBINED build)](gpu_dual_runtime_dispatch.md) instead.
* For OpenCL-based remote context/tensor interop while running on the Level Zero runtime, see [Level Zero - OpenCL interoperability](blog/ze_ocl_interop_2026_3.md).