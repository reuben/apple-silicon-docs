# Apple Silicon Docs

Notes from figuring out development on Apple Silicon.

## TensorFlow

Builds fine from source, pre-built binaries are completely busted.

Linux wheels crash on QEMU/Docker due to AVX instructions: [1](https://github.com/tensorflow/tensorflow/issues/19584) [2](https://gitlab.com/qemu-project/qemu/-/issues/164) [3](https://github.com/tensorflow/tensorflow/issues/52845)

Workarounds:
 - [Build from source](https://www.tensorflow.org/install/source)
 - [Install community wheel built with AVX disabled](https://github.com/yaroslavvb/tensorflow-community-wheels/issues/208)

Not tested:
- Set up a linux/aarch64 dev environment under Docker and [install aarch64 community wheels](https://github.com/KumaTea/tensorflow-aarch64/)

## Python

Still figuring out universal builds, Python 3.10 + `python -m build` seems to have some built-in support, some of our packages got automatically upgraded to a universal tag. Older versions more complicated: lack of NumPy support, build system awareness of fat binaries, etc.

## Xcode

CommandLineTools toolchain seems fairly broken, had to uninstall it locally to avoid architecture related errors:

```
$ sudo rm -r /Library/Developer/CommandLineTools
```

Cross-building from Intel Macs to ARM Macs seems to work fairly smoothly with Xcode after that. Some notes:

- `-march` flags are not supported on ARM
- Universal builds require building twice and then running `lipo` to create a fat binary. Can be annoying.
- Unclear how min/max macOS version constraints interact when building fat binaries. ARM binaries have a min version of 11.0, but you usually want to build Intel macOS binaries with compatibility for older versions. See for example [1](https://github.com/pypa/wheel/issues/387).

## Docker

Don't forget to specify `--platform` when building/running images, eg. `--platform linux/amd64`.
