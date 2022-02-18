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
