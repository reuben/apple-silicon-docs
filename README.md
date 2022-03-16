# Apple Silicon Docs

Notes from figuring out development on Apple Silicon.

## TensorFlow

Builds fine from source, pre-built binaries are completely busted:

```
2022-03-10 10:38:34.891590: F tensorflow/core/platform/cpu_feature_guard.cc:37] The TensorFlow library was compiled to use AVX instructions, but these aren't available on your machine.
qemu: uncaught target signal 6 (Aborted) - core dumped
Aborted
```

Linux wheels crash on QEMU/Docker due to AVX instructions: [1](https://github.com/tensorflow/tensorflow/issues/19584) [2](https://gitlab.com/qemu-project/qemu/-/issues/164) [3](https://github.com/tensorflow/tensorflow/issues/52845)

Workarounds:
 - [Build from source](https://www.tensorflow.org/install/source)
 - [Install community wheel built with AVX disabled](https://github.com/yaroslavvb/tensorflow-community-wheels/issues/208)

Not tested:
- Set up a linux/aarch64 dev environment under Docker and [install aarch64 community wheels](https://github.com/KumaTea/tensorflow-aarch64/)

My own pre-built wheels (Python 3.10, macOS 12.0) are here: https://github.com/reuben/apple-silicon-docs/releases/tag/prebuilt-py3.10-wheels

## Python

Still figuring out universal builds, Python 3.10 + `python -m build` seems to have some built-in support, some of our packages got automatically upgraded to a universal tag. Older versions more complicated: lack of NumPy support, build system awareness of fat binaries, etc.

## Xcode

CommandLineTools toolchain seems fairly broken, had to uninstall it locally to avoid architecture related errors:

```
$ sudo rm -r /Library/Developer/CommandLineTools
```

...but then this breaks installing some Homebrew bottles, forcing build from source. For example:

```
$ brew install hdf5
==> Auto-updated Homebrew!
Updated 1 tap (homebrew/core).
==> Updated Formulae
Updated 2 formulae.

==> Downloading https://ghcr.io/v2/homebrew/core/gmp/manifests/6.2.1_1
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/gmp/blobs/sha256:a43a2ae4c44d90626b835a968a32327c8b8bbf754ec1d2590f8ac656c71dace9
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:a43a2ae4c44d90626b835a968a32327c8b8bbf754ec1d2590f8ac656c71dace9?se=2022-02-17T12%3A15%3A00Z&sig=LMrE9hpxI8jTitsSn5LBp7
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/isl/manifests/0.24
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/isl/blobs/sha256:be08c3e9765655ad5bfd227f9b97acb0ef88ad2307dc214ea4064cc1f51db641
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:be08c3e9765655ad5bfd227f9b97acb0ef88ad2307dc214ea4064cc1f51db641?se=2022-02-17T12%3A15%3A00Z&sig=HPmNwCWHLSDjtvBwM6HE5j
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/mpfr/manifests/4.1.0
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/mpfr/blobs/sha256:81ced499f237acfc2773711a3f8aa985572eaab2344a70485c06f72405e4a5e7
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:81ced499f237acfc2773711a3f8aa985572eaab2344a70485c06f72405e4a5e7?se=2022-02-17T12%3A15%3A00Z&sig=PTZtTr60%2BH017DLTvm7I
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libmpc/manifests/1.2.1
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/libmpc/blobs/sha256:658a1d260b6f77c431451a554ef8fa36ea2b6db19b38adc05c52821598ce2647
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:658a1d260b6f77c431451a554ef8fa36ea2b6db19b38adc05c52821598ce2647?se=2022-02-17T12%3A15%3A00Z&sig=13KA9n4MqbHpFPnM%2Fz22
######################################################################## 100.0%
Error: gcc: the bottle needs the Apple Command Line Tools to be installed.
  You can install them, if desired, with:
    xcode-select --install

You can try to install from source with:
  brew install --build-from-source gcc
Please note building from source is unsupported. You will encounter build
failures with some formulae. If you experience any issues please create pull
requests instead of asking for help on Homebrew's GitHub, Twitter or any other
official channels.
```

Maybe just temporarily move the folder out to prevent its paths being injected at random builds.

Cross-building from Intel Macs to ARM Macs seems to work fairly smoothly with Xcode after that. Some notes:

- `-march`/`-mtune`/`-msse`/etc flags are not supported on ARM
- Universal builds require building twice for different targets and then merging the binaries with `lipo` tool to create a universal binary. Can be annoying.
- Unclear how min/max macOS version constraints interact when building fat binaries. ARM binaries have a min version of 11.0, but you usually want to build Intel macOS binaries with compatibility for older versions. See for example [1](https://github.com/pypa/wheel/issues/387).

## Docker

Don't forget to specify `--platform` when building/running images, eg. `--platform linux/amd64`.

Bazel doesn't like running inside Docker, needs to be started in batch mode and when building with QEMU it fails to self-extract: https://github.com/bazelbuild/bazel/pull/14391

## ONNX

[onnxruntime package doesn't have pre-built wheels for Python 3.10](https://github.com/microsoft/onnxruntime/issues/9782). Building from source in a virtualenv requires some manual fixes:

```
./build.sh --config Release --parallel --build_wheel --osx_arch arm64
2022-03-16 19:23:20,214 tools_python_utils [INFO] - flatbuffers module is not installed. parse_config will not be available
2022-03-16 19:23:20,215 build [DEBUG] - Command line arguments:
  --build_dir /Users/reuben/Development/onnxruntime/build/MacOS --config Release --parallel --build_wheel --osx_arch arm64
2022-03-16 19:23:20,217 build [DEBUG] - Defaulting to running update, build [and test for native builds].
2022-03-16 19:23:20,217 build [INFO] - Build started
2022-03-16 19:23:20,217 util.run [INFO] - Running subprocess in '/Users/reuben/Development/onnxruntime'
  git submodule sync --recursive
Synchronizing submodule url for 'cmake/external/SafeInt/safeint'
Synchronizing submodule url for 'cmake/external/cub'
Synchronizing submodule url for 'cmake/external/cxxopts'
Synchronizing submodule url for 'cmake/external/date'
Synchronizing submodule url for 'cmake/external/dlpack'
Synchronizing submodule url for 'cmake/external/eigen'
Synchronizing submodule url for 'cmake/external/emsdk'
Synchronizing submodule url for 'cmake/external/flatbuffers'
Synchronizing submodule url for 'cmake/external/googlebenchmark'
Synchronizing submodule url for 'cmake/external/googletest'
Synchronizing submodule url for 'cmake/external/json'
Synchronizing submodule url for 'cmake/external/libprotobuf-mutator'
Synchronizing submodule url for 'cmake/external/mimalloc'
Synchronizing submodule url for 'cmake/external/mp11'
Synchronizing submodule url for 'cmake/external/nsync'
Synchronizing submodule url for 'cmake/external/onnx'
Synchronizing submodule url for 'cmake/external/onnx/third_party/benchmark'
Synchronizing submodule url for 'cmake/external/onnx/third_party/pybind11'
Synchronizing submodule url for 'cmake/external/onnx-tensorrt'
Synchronizing submodule url for 'cmake/external/onnx-tensorrt/third_party/onnx'
Synchronizing submodule url for 'cmake/external/onnx-tensorrt/third_party/onnx/third_party/benchmark'
Synchronizing submodule url for 'cmake/external/onnx-tensorrt/third_party/onnx/third_party/pybind11'
Synchronizing submodule url for 'cmake/external/onnxruntime-extensions'
Synchronizing submodule url for 'cmake/external/protobuf'
Synchronizing submodule url for 'cmake/external/protobuf/third_party/benchmark'
Synchronizing submodule url for 'cmake/external/protobuf/third_party/googletest'
Synchronizing submodule url for 'cmake/external/pytorch_cpuinfo'
Synchronizing submodule url for 'cmake/external/re2'
Synchronizing submodule url for 'cmake/external/tensorboard'
Synchronizing submodule url for 'cmake/external/wil'
Synchronizing submodule url for 'server/external/spdlog'
2022-03-16 19:23:20,421 util.run [DEBUG] - Subprocess completed. Return code: 0
2022-03-16 19:23:20,422 util.run [INFO] - Running subprocess in '/Users/reuben/Development/onnxruntime'
  git submodule update --init --recursive
2022-03-16 19:23:21,190 util.run [DEBUG] - Subprocess completed. Return code: 0
2022-03-16 19:23:21,190 build [INFO] - Generating CMake build tree
2022-03-16 19:23:21,191 util.run [INFO] - Running subprocess in '/Users/reuben/Development/onnxruntime/build/MacOS/Release'
  /opt/homebrew/bin/cmake /Users/reuben/Development/onnxruntime/cmake -Donnxruntime_RUN_ONNX_TESTS=OFF -Donnxruntime_BUILD_WINML_TESTS=ON -Donnxruntime_GENERATE_TEST_REPORTS=ON -DPython_EXECUTABLE=/Users/reuben/.pyenv/versions/onnxruntime/bin/python3 -DPYTHON_EXECUTABLE=/Users/reuben/.pyenv/versions/onnxruntime/bin/python3 -Donnxruntime_ROCM_VERSION= -Donnxruntime_USE_MIMALLOC=OFF -Donnxruntime_ENABLE_PYTHON=ON -Donnxruntime_BUILD_CSHARP=OFF -Donnxruntime_BUILD_JAVA=OFF -Donnxruntime_BUILD_NODEJS=OFF -Donnxruntime_BUILD_OBJC=OFF -Donnxruntime_BUILD_SHARED_LIB=OFF -Donnxruntime_BUILD_APPLE_FRAMEWORK=OFF -Donnxruntime_USE_DNNL=OFF -Donnxruntime_DNNL_GPU_RUNTIME= -Donnxruntime_DNNL_OPENCL_ROOT= -Donnxruntime_USE_NNAPI_BUILTIN=OFF -Donnxruntime_USE_RKNPU=OFF -Donnxruntime_USE_OPENMP=OFF -Donnxruntime_USE_NUPHAR_TVM=OFF -Donnxruntime_USE_LLVM=OFF -Donnxruntime_ENABLE_MICROSOFT_INTERNAL=OFF -Donnxruntime_USE_VITISAI=OFF -Donnxruntime_USE_NUPHAR=OFF -Donnxruntime_USE_TENSORRT=OFF -Donnxruntime_TENSORRT_HOME= -Donnxruntime_USE_TVM=OFF -Donnxruntime_TVM_CUDA_RUNTIME=OFF -Donnxruntime_USE_MIGRAPHX=OFF -Donnxruntime_MIGRAPHX_HOME= -Donnxruntime_CROSS_COMPILING=OFF -Donnxruntime_DISABLE_CONTRIB_OPS=OFF -Donnxruntime_DISABLE_ML_OPS=OFF -Donnxruntime_DISABLE_RTTI=OFF -Donnxruntime_DISABLE_EXCEPTIONS=OFF -Donnxruntime_MINIMAL_BUILD=OFF -Donnxruntime_EXTENDED_MINIMAL_BUILD=OFF -Donnxruntime_MINIMAL_BUILD_CUSTOM_OPS=OFF -Donnxruntime_REDUCED_OPS_BUILD=OFF -Donnxruntime_ENABLE_LANGUAGE_INTEROP_OPS=OFF -Donnxruntime_USE_DML=OFF -Donnxruntime_USE_WINML=OFF -Donnxruntime_BUILD_MS_EXPERIMENTAL_OPS=OFF -Donnxruntime_USE_TELEMETRY=OFF -Donnxruntime_ENABLE_LTO=OFF -Donnxruntime_ENABLE_TRANSFORMERS_TOOL_TEST=OFF -Donnxruntime_USE_ACL=OFF -Donnxruntime_USE_ACL_1902=OFF -Donnxruntime_USE_ACL_1905=OFF -Donnxruntime_USE_ACL_1908=OFF -Donnxruntime_USE_ACL_2002=OFF -Donnxruntime_USE_ARMNN=OFF -Donnxruntime_ARMNN_RELU_USE_CPU=ON -Donnxruntime_ARMNN_BN_USE_CPU=ON -Donnxruntime_ENABLE_NVTX_PROFILE=OFF -Donnxruntime_ENABLE_TRAINING=OFF -Donnxruntime_ENABLE_TRAINING_OPS=OFF -Donnxruntime_ENABLE_TRAINING_TORCH_INTEROP=OFF -Donnxruntime_ENABLE_CPU_FP16_OPS=OFF -Donnxruntime_USE_NCCL=ON -Donnxruntime_BUILD_BENCHMARKS=OFF -Donnxruntime_USE_ROCM=OFF -Donnxruntime_ROCM_HOME= -DOnnxruntime_GCOV_COVERAGE=OFF -Donnxruntime_USE_MPI=ON -Donnxruntime_ENABLE_MEMORY_PROFILE=OFF -Donnxruntime_ENABLE_CUDA_LINE_NUMBER_INFO=OFF -Donnxruntime_BUILD_WEBASSEMBLY=OFF -Donnxruntime_BUILD_WEBASSEMBLY_STATIC_LIB=OFF -Donnxruntime_ENABLE_WEBASSEMBLY_SIMD=OFF -Donnxruntime_ENABLE_WEBASSEMBLY_EXCEPTION_CATCHING=ON -Donnxruntime_ENABLE_WEBASSEMBLY_EXCEPTION_THROWING=OFF -Donnxruntime_ENABLE_WEBASSEMBLY_THREADS=OFF -Donnxruntime_ENABLE_WEBASSEMBLY_DEBUG_INFO=OFF -Donnxruntime_ENABLE_WEBASSEMBLY_PROFILING=OFF -Donnxruntime_WEBASSEMBLY_MALLOC=dlmalloc -Donnxruntime_ENABLE_EAGER_MODE=OFF -Donnxruntime_ENABLE_EXTERNAL_CUSTOM_OP_SCHEMAS=OFF -Donnxruntime_NVCC_THREADS=0 -Donnxruntime_ENABLE_CUDA_PROFILING=OFF -DCMAKE_OSX_ARCHITECTURES=arm64 -Donnxruntime_DEV_MODE=ON -Donnxruntime_PYBIND_EXPORT_OPSCHEMA=OFF -Donnxruntime_ENABLE_MEMLEAK_CHECKER=OFF -DCMAKE_BUILD_TYPE=Release
Patch found: /usr/bin/patch
Adding flags for Mac builds
Use gtest from submodule
-- Found Python: /Users/reuben/.pyenv/versions/onnxruntime/bin/python3 (found version "3.10.2") found components: Interpreter
-- Could NOT find Python (missing: Development.Module NumPy) (found suitable version "3.10.2", minimum required is "3.6")
Use protobuf from submodule
--
-- 3.18.1.0
protoc can run
-- Using the single-header code from /Users/reuben/Development/onnxruntime/cmake/external/json/single_include/
NVCC_ERROR =
NVCC_OUT = No such file or directory
-- Found PythonInterp: /Users/reuben/.pyenv/versions/onnxruntime/bin/python3 (found version "3.10.2")
-- Could NOT find PythonLibs (missing: PYTHON_LIBRARIES PYTHON_INCLUDE_DIRS)
Generated: /Users/reuben/Development/onnxruntime/build/MacOS/Release/external/onnx/onnx/onnx-ml.proto
Generated: /Users/reuben/Development/onnxruntime/build/MacOS/Release/external/onnx/onnx/onnx-operators-ml.proto
Generated: /Users/reuben/Development/onnxruntime/build/MacOS/Release/external/onnx/onnx/onnx-data.proto
--
-- ******** Summary ********
--   CMake version             : 3.22.3
--   CMake command             : /opt/homebrew/Cellar/cmake/3.22.3/bin/cmake
--   System                    : Darwin
--   C++ compiler              : /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/c++
--   C++ compiler version      : 13.1.6.13160021
--   CXX flags                 :  -ffunction-sections -fdata-sections -fvisibility=hidden -fvisibility-inlines-hidden -fstack-protector-strong -DCPUINFO_SUPPORTED -Wno-deprecated -Wnon-virtual-dtor
--   Build type                : Release
--   Compile definitions       : EIGEN_MPL2_ONLY;PLATFORM_POSIX;__STDC_FORMAT_MACROS
--   CMAKE_PREFIX_PATH         :
--   CMAKE_INSTALL_PREFIX      : /usr/local
--   CMAKE_MODULE_PATH         : /Users/reuben/Development/onnxruntime/cmake/external
--
--   ONNX version              : 1.11.0
--   ONNX NAMESPACE            : onnx
--   ONNX_USE_LITE_PROTO       : ON
--   USE_PROTOBUF_SHARED_LIBS  : OFF
--   Protobuf_USE_STATIC_LIBS  : ON
--   ONNX_DISABLE_EXCEPTIONS   : OFF
--   ONNX_WERROR               : OFF
--   ONNX_BUILD_TESTS          : OFF
--   ONNX_BUILD_BENCHMARKS     : OFF
--   ONNXIFI_DUMMY_BACKEND     : OFF
--   ONNXIFI_ENABLE_EXT        : OFF
--
--   Protobuf compiler         :
--   Protobuf includes         :
--   Protobuf libraries        :
--   BUILD_ONNX_PYTHON         : OFF
-- Configuring done
CMake Error at CMakeLists.txt:1252 (target_compile_definitions):
  Error evaluating generator expression:

    $<TARGET_PROPERTY:Python::Module,INTERFACE_COMPILE_DEFINITIONS>

  Target "Python::Module" not found.
Call Stack (most recent call first):
  onnxruntime_python.cmake:76 (onnxruntime_add_include_to_target)
  CMakeLists.txt:1973 (include)


CMake Error at CMakeLists.txt:1252 (target_compile_definitions):
  Error evaluating generator expression:

    $<TARGET_PROPERTY:Python::NumPy,INTERFACE_COMPILE_DEFINITIONS>

  Target "Python::NumPy" not found.
Call Stack (most recent call first):
  onnxruntime_python.cmake:76 (onnxruntime_add_include_to_target)
  CMakeLists.txt:1973 (include)


CMake Error at CMakeLists.txt:1251 (target_include_directories):
  Error evaluating generator expression:

    $<TARGET_PROPERTY:Python::Module,INTERFACE_INCLUDE_DIRECTORIES>

  Target "Python::Module" not found.
Call Stack (most recent call first):
  onnxruntime_python.cmake:76 (onnxruntime_add_include_to_target)
  CMakeLists.txt:1973 (include)


CMake Error at CMakeLists.txt:1251 (target_include_directories):
  Error evaluating generator expression:

    $<TARGET_PROPERTY:Python::NumPy,INTERFACE_INCLUDE_DIRECTORIES>

  Target "Python::NumPy" not found.
Call Stack (most recent call first):
  onnxruntime_python.cmake:76 (onnxruntime_add_include_to_target)
  CMakeLists.txt:1973 (include)


CMake Error at CMakeLists.txt:1251 (target_include_directories):
  Error evaluating generator expression:

    $<TARGET_PROPERTY:Python::Module,INTERFACE_INCLUDE_DIRECTORIES>

  Target "Python::Module" not found.
Call Stack (most recent call first):
  onnxruntime_python.cmake:76 (onnxruntime_add_include_to_target)
  CMakeLists.txt:1973 (include)


CMake Error at CMakeLists.txt:1251 (target_include_directories):
  Error evaluating generator expression:

    $<TARGET_PROPERTY:Python::NumPy,INTERFACE_INCLUDE_DIRECTORIES>

  Target "Python::NumPy" not found.
Call Stack (most recent call first):
  onnxruntime_python.cmake:76 (onnxruntime_add_include_to_target)
  CMakeLists.txt:1973 (include)


CMake Error at CMakeLists.txt:1251 (target_include_directories):
  Error evaluating generator expression:

    $<TARGET_PROPERTY:Python::Module,INTERFACE_INCLUDE_DIRECTORIES>

  Target "Python::Module" not found.
Call Stack (most recent call first):
  onnxruntime_python.cmake:76 (onnxruntime_add_include_to_target)
  CMakeLists.txt:1973 (include)


CMake Error at CMakeLists.txt:1251 (target_include_directories):
  Error evaluating generator expression:

    $<TARGET_PROPERTY:Python::NumPy,INTERFACE_INCLUDE_DIRECTORIES>

  Target "Python::NumPy" not found.
Call Stack (most recent call first):
  onnxruntime_python.cmake:76 (onnxruntime_add_include_to_target)
  CMakeLists.txt:1973 (include)


-- Generating done
CMake Generate step failed.  Build files cannot be regenerated correctly.
Traceback (most recent call last):
  File "/Users/reuben/Development/onnxruntime/tools/ci_build/build.py", line 2427, in <module>
    sys.exit(main())
  File "/Users/reuben/Development/onnxruntime/tools/ci_build/build.py", line 2327, in main
    generate_build_tree(
  File "/Users/reuben/Development/onnxruntime/tools/ci_build/build.py", line 1138, in generate_build_tree
    run_subprocess(
  File "/Users/reuben/Development/onnxruntime/tools/ci_build/build.py", line 655, in run_subprocess
    return run(*args, cwd=cwd, capture_stdout=capture_stdout, shell=shell, env=my_env)
  File "/Users/reuben/Development/onnxruntime/tools/python/util/run.py", line 42, in run
    completed_process = subprocess.run(
  File "/Users/reuben/.pyenv/versions/3.10.2/lib/python3.10/subprocess.py", line 524, in run
    raise CalledProcessError(retcode, process.args,
subprocess.CalledProcessError: Command '['/opt/homebrew/bin/cmake', '/Users/reuben/Development/onnxruntime/cmake', '-Donnxruntime_RUN_ONNX_TESTS=OFF', '-Donnxruntime_BUILD_WINML_TESTS=ON', '-Donnxruntime_GENERATE_TEST_REPORTS=ON', '-DPython_EXECUTABLE=/Users/reuben/.pyenv/versions/onnxruntime/bin/python3', '-DPYTHON_EXECUTABLE=/Users/reuben/.pyenv/versions/onnxruntime/bin/python3', '-Donnxruntime_ROCM_VERSION=', '-Donnxruntime_USE_MIMALLOC=OFF', '-Donnxruntime_ENABLE_PYTHON=ON', '-Donnxruntime_BUILD_CSHARP=OFF', '-Donnxruntime_BUILD_JAVA=OFF', '-Donnxruntime_BUILD_NODEJS=OFF', '-Donnxruntime_BUILD_OBJC=OFF', '-Donnxruntime_BUILD_SHARED_LIB=OFF', '-Donnxruntime_BUILD_APPLE_FRAMEWORK=OFF', '-Donnxruntime_USE_DNNL=OFF', '-Donnxruntime_DNNL_GPU_RUNTIME=', '-Donnxruntime_DNNL_OPENCL_ROOT=', '-Donnxruntime_USE_NNAPI_BUILTIN=OFF', '-Donnxruntime_USE_RKNPU=OFF', '-Donnxruntime_USE_OPENMP=OFF', '-Donnxruntime_USE_NUPHAR_TVM=OFF', '-Donnxruntime_USE_LLVM=OFF', '-Donnxruntime_ENABLE_MICROSOFT_INTERNAL=OFF', '-Donnxruntime_USE_VITISAI=OFF', '-Donnxruntime_USE_NUPHAR=OFF', '-Donnxruntime_USE_TENSORRT=OFF', '-Donnxruntime_TENSORRT_HOME=', '-Donnxruntime_USE_TVM=OFF', '-Donnxruntime_TVM_CUDA_RUNTIME=OFF', '-Donnxruntime_USE_MIGRAPHX=OFF', '-Donnxruntime_MIGRAPHX_HOME=', '-Donnxruntime_CROSS_COMPILING=OFF', '-Donnxruntime_DISABLE_CONTRIB_OPS=OFF', '-Donnxruntime_DISABLE_ML_OPS=OFF', '-Donnxruntime_DISABLE_RTTI=OFF', '-Donnxruntime_DISABLE_EXCEPTIONS=OFF', '-Donnxruntime_MINIMAL_BUILD=OFF', '-Donnxruntime_EXTENDED_MINIMAL_BUILD=OFF', '-Donnxruntime_MINIMAL_BUILD_CUSTOM_OPS=OFF', '-Donnxruntime_REDUCED_OPS_BUILD=OFF', '-Donnxruntime_ENABLE_LANGUAGE_INTEROP_OPS=OFF', '-Donnxruntime_USE_DML=OFF', '-Donnxruntime_USE_WINML=OFF', '-Donnxruntime_BUILD_MS_EXPERIMENTAL_OPS=OFF', '-Donnxruntime_USE_TELEMETRY=OFF', '-Donnxruntime_ENABLE_LTO=OFF', '-Donnxruntime_ENABLE_TRANSFORMERS_TOOL_TEST=OFF', '-Donnxruntime_USE_ACL=OFF', '-Donnxruntime_USE_ACL_1902=OFF', '-Donnxruntime_USE_ACL_1905=OFF', '-Donnxruntime_USE_ACL_1908=OFF', '-Donnxruntime_USE_ACL_2002=OFF', '-Donnxruntime_USE_ARMNN=OFF', '-Donnxruntime_ARMNN_RELU_USE_CPU=ON', '-Donnxruntime_ARMNN_BN_USE_CPU=ON', '-Donnxruntime_ENABLE_NVTX_PROFILE=OFF', '-Donnxruntime_ENABLE_TRAINING=OFF', '-Donnxruntime_ENABLE_TRAINING_OPS=OFF', '-Donnxruntime_ENABLE_TRAINING_TORCH_INTEROP=OFF', '-Donnxruntime_ENABLE_CPU_FP16_OPS=OFF', '-Donnxruntime_USE_NCCL=ON', '-Donnxruntime_BUILD_BENCHMARKS=OFF', '-Donnxruntime_USE_ROCM=OFF', '-Donnxruntime_ROCM_HOME=', '-DOnnxruntime_GCOV_COVERAGE=OFF', '-Donnxruntime_USE_MPI=ON', '-Donnxruntime_ENABLE_MEMORY_PROFILE=OFF', '-Donnxruntime_ENABLE_CUDA_LINE_NUMBER_INFO=OFF', '-Donnxruntime_BUILD_WEBASSEMBLY=OFF', '-Donnxruntime_BUILD_WEBASSEMBLY_STATIC_LIB=OFF', '-Donnxruntime_ENABLE_WEBASSEMBLY_SIMD=OFF', '-Donnxruntime_ENABLE_WEBASSEMBLY_EXCEPTION_CATCHING=ON', '-Donnxruntime_ENABLE_WEBASSEMBLY_EXCEPTION_THROWING=OFF', '-Donnxruntime_ENABLE_WEBASSEMBLY_THREADS=OFF', '-Donnxruntime_ENABLE_WEBASSEMBLY_DEBUG_INFO=OFF', '-Donnxruntime_ENABLE_WEBASSEMBLY_PROFILING=OFF', '-Donnxruntime_WEBASSEMBLY_MALLOC=dlmalloc', '-Donnxruntime_ENABLE_EAGER_MODE=OFF', '-Donnxruntime_ENABLE_EXTERNAL_CUSTOM_OP_SCHEMAS=OFF', '-Donnxruntime_NVCC_THREADS=0', '-Donnxruntime_ENABLE_CUDA_PROFILING=OFF', '-DCMAKE_OSX_ARCHITECTURES=arm64', '-Donnxruntime_DEV_MODE=ON', '-Donnxruntime_PYBIND_EXPORT_OPSCHEMA=OFF', '-Donnxruntime_ENABLE_MEMLEAK_CHECKER=OFF', '-DCMAKE_BUILD_TYPE=Release']' returned non-zero exit status 1.
```

To fix it:

```
$ pushd /Users/reuben/Development/onnxruntime/build/MacOS/Release
$ /opt/homebrew/bin/cmake /Users/reuben/Development/onnxruntime/cmake -Donnxruntime_RUN_ONNX_TESTS=OFF -Donnxruntime_BUILD_WINML_TESTS=ON -Donnxruntime_GENERATE_TEST_REPORTS=ON -DPython_EXECUTABLE=/Users/reuben/.pyenv/versions/onnxruntime/bin/python3 -DPYTHON_EXECUTABLE=/Users/reuben/.pyenv/versions/onnxruntime/bin/python3 -Donnxruntime_ROCM_VERSION= -Donnxruntime_USE_MIMALLOC=OFF -Donnxruntime_ENABLE_PYTHON=ON -Donnxruntime_BUILD_CSHARP=OFF -Donnxruntime_BUILD_JAVA=OFF -Donnxruntime_BUILD_NODEJS=OFF -Donnxruntime_BUILD_OBJC=OFF -Donnxruntime_BUILD_SHARED_LIB=OFF -Donnxruntime_BUILD_APPLE_FRAMEWORK=OFF -Donnxruntime_USE_DNNL=OFF -Donnxruntime_DNNL_GPU_RUNTIME= -Donnxruntime_DNNL_OPENCL_ROOT= -Donnxruntime_USE_NNAPI_BUILTIN=OFF -Donnxruntime_USE_RKNPU=OFF -Donnxruntime_USE_OPENMP=OFF -Donnxruntime_USE_NUPHAR_TVM=OFF -Donnxruntime_USE_LLVM=OFF -Donnxruntime_ENABLE_MICROSOFT_INTERNAL=OFF -Donnxruntime_USE_VITISAI=OFF -Donnxruntime_USE_NUPHAR=OFF -Donnxruntime_USE_TENSORRT=OFF -Donnxruntime_TENSORRT_HOME= -Donnxruntime_USE_TVM=OFF -Donnxruntime_TVM_CUDA_RUNTIME=OFF -Donnxruntime_USE_MIGRAPHX=OFF -Donnxruntime_MIGRAPHX_HOME= -Donnxruntime_CROSS_COMPILING=OFF -Donnxruntime_DISABLE_CONTRIB_OPS=OFF -Donnxruntime_DISABLE_ML_OPS=OFF -Donnxruntime_DISABLE_RTTI=OFF -Donnxruntime_DISABLE_EXCEPTIONS=OFF -Donnxruntime_MINIMAL_BUILD=OFF -Donnxruntime_EXTENDED_MINIMAL_BUILD=OFF -Donnxruntime_MINIMAL_BUILD_CUSTOM_OPS=OFF -Donnxruntime_REDUCED_OPS_BUILD=OFF -Donnxruntime_ENABLE_LANGUAGE_INTEROP_OPS=OFF -Donnxruntime_USE_DML=OFF -Donnxruntime_USE_WINML=OFF -Donnxruntime_BUILD_MS_EXPERIMENTAL_OPS=OFF -Donnxruntime_USE_TELEMETRY=OFF -Donnxruntime_ENABLE_LTO=OFF -Donnxruntime_ENABLE_TRANSFORMERS_TOOL_TEST=OFF -Donnxruntime_USE_ACL=OFF -Donnxruntime_USE_ACL_1902=OFF -Donnxruntime_USE_ACL_1905=OFF -Donnxruntime_USE_ACL_1908=OFF -Donnxruntime_USE_ACL_2002=OFF -Donnxruntime_USE_ARMNN=OFF -Donnxruntime_ARMNN_RELU_USE_CPU=ON -Donnxruntime_ARMNN_BN_USE_CPU=ON -Donnxruntime_ENABLE_NVTX_PROFILE=OFF -Donnxruntime_ENABLE_TRAINING=OFF -Donnxruntime_ENABLE_TRAINING_OPS=OFF -Donnxruntime_ENABLE_TRAINING_TORCH_INTEROP=OFF -Donnxruntime_ENABLE_CPU_FP16_OPS=OFF -Donnxruntime_USE_NCCL=ON -Donnxruntime_BUILD_BENCHMARKS=OFF -Donnxruntime_USE_ROCM=OFF -Donnxruntime_ROCM_HOME= -DOnnxruntime_GCOV_COVERAGE=OFF -Donnxruntime_USE_MPI=ON -Donnxruntime_ENABLE_MEMORY_PROFILE=OFF -Donnxruntime_ENABLE_CUDA_LINE_NUMBER_INFO=OFF -Donnxruntime_BUILD_WEBASSEMBLY=OFF -Donnxruntime_BUILD_WEBASSEMBLY_STATIC_LIB=OFF -Donnxruntime_ENABLE_WEBASSEMBLY_SIMD=OFF -Donnxruntime_ENABLE_WEBASSEMBLY_EXCEPTION_CATCHING=ON -Donnxruntime_ENABLE_WEBASSEMBLY_EXCEPTION_THROWING=OFF -Donnxruntime_ENABLE_WEBASSEMBLY_THREADS=OFF -Donnxruntime_ENABLE_WEBASSEMBLY_DEBUG_INFO=OFF -Donnxruntime_ENABLE_WEBASSEMBLY_PROFILING=OFF -Donnxruntime_WEBASSEMBLY_MALLOC=dlmalloc -Donnxruntime_ENABLE_EAGER_MODE=OFF -Donnxruntime_ENABLE_EXTERNAL_CUSTOM_OP_SCHEMAS=OFF -Donnxruntime_NVCC_THREADS=0 -Donnxruntime_ENABLE_CUDA_PROFILING=OFF -DCMAKE_OSX_ARCHITECTURES=arm64 -Donnxruntime_DEV_MODE=ON -Donnxruntime_PYBIND_EXPORT_OPSCHEMA=OFF -Donnxruntime_ENABLE_MEMLEAK_CHECKER=OFF -DCMAKE_BUILD_TYPE=Release -DPYTHON_INCLUDE_DIR=$(python -c "from distutils.sysconfig import get_python_inc; print(get_python_inc())")  -DPYTHON_LIBRARY=$(python -c "import distutils.sysconfig as sysconfig; print(sysconfig.get_config_var('LIBDIR'))")
$ popd
$ /opt/homebrew/bin/cmake --build /Users/reuben/Development/onnxruntime/build/MacOS/Release --config Release -- -j10
```
