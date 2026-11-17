# Deskflow adapter for EdgeBridge

This macOS patch lets EdgeBridge carry a physical mouse gesture between a Mac primary and a Windows client while Deskflow routes the pointer. It adds a private local socket, tracks the original button press and verifies the window that receives a new drag on the Mac. EdgeBridge's native helpers handle the files and the destination selected in Finder or Explorer.

This repository distributes the source patch, license texts and build notes. Build the core locally and supply your own Deskflow configuration, screen names and pairing material. EdgeBridge is a separate MIT project that communicates with the adapter through IPC. The adapter must run in place of the ordinary Deskflow server during an interactive test.

## Apply the patch

Use a fresh checkout of [Deskflow v1.27.0](https://github.com/deskflow/deskflow/releases/tag/v1.27.0). The [tested upstream commit](https://github.com/deskflow/deskflow/commit/081f6478e654a112168879e67a41ce7d7e4ae3af) is pinned below. Run these commands from this adapter repository.

```sh
adapter_root="$(pwd -P)"
patch_file="$adapter_root/deskflow-1.27-mac-gesture-route.patch"
shasum -a 256 "$patch_file"
git clone --branch v1.27.0 --depth 1 https://github.com/deskflow/deskflow.git deskflow-upstream
cd deskflow-upstream
test "$(git rev-parse HEAD)" = 081f6478e654a112168879e67a41ce7d7e4ae3af &&
  git apply --check "$patch_file" &&
  git apply "$patch_file"
```

The patch SHA-256 is `d6543cf4eedbcc7db80769d05d3c4c50b1277ad782a479710a3535a5da331c4c`.

## Build on macOS

The verified Apple Silicon build used the Xcode Command Line Tools, CMake 4.5, Ninja 1.13, Qt 6.12 (including Qt Linguist Tools) and OpenSSL 4 from Homebrew. From the patched Deskflow checkout:

```sh
brew install cmake ninja qtbase qttools openssl@4
cmake -S . -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH="$(brew --prefix)" \
  -DOPENSSL_ROOT_DIR="$(brew --prefix openssl@4)" \
  -DBUILD_TESTS=ON -DBUILD_INSTALLER=OFF -DBUILD_OSX_BUNDLE=OFF -DBUILD_DEV_DOCS=OFF
cmake --build build --parallel --target deskflow-core \
  EdgeRouteInputTests EdgeRouteEndTimestampTests EdgeRouteSocketTests \
  EdgeRouteSocketLifecycleTests ServerTests ServerConfigTests OSXTisThreadTests
```

The executable is `build/bin/deskflow-core`. To use the adapter, configure its process with `EDGEBRIDGE_ROUTE_SOCKET` set to an absolute Unix socket path inside a directory owned by the current user with mode `0700`. The adapter is opt-in. Building it and running the tests below do not start a keyboard/mouse server.

## Verify the build

```sh
env -u EDGEBRIDGE_ROUTE_SOCKET QT_QPA_PLATFORM=offscreen \
  ctest --test-dir build/src/unittests/server \
  -R '^(EdgeRoute.*|ServerTests|ServerConfigTests)$' --output-on-failure
env -u EDGEBRIDGE_ROUTE_SOCKET QT_QPA_PLATFORM=offscreen \
  ctest --test-dir build/src/unittests/platform \
  -R '^OSXTisThreadTests$' --output-on-failure
```

The seven tests cover physical input generations, release timing, socket lifetime, scoped destination-window checks, two upstream server regressions and macOS input-source threading. The route tests use virtual input. An interactive drop is verified separately on both operating systems by observing the drag through release and checking the file at the selected destination. [macOS compatibility notes](MAC-TIS-COMPATIBILITY.md)

## License

The new route code uses `GPL-2.0-only WITH LicenseRef-OpenSSL-Exception`. Existing Deskflow files retain their upstream copyright and per-file license notices. The OpenSSL exception permits linking and distributing the combination under the terms in its accompanying text. [GNU GPL version 2](GPL-2.0-only.txt) · [OpenSSL exception](LicenseRef-OpenSSL-Exception.txt)

## MIT 主项目

独立的文件传输和原生手势程序见 [EdgeBridge](https://github.com/qwe12yuh/EdgeBridge)。本仓补丁保留自身许可，不属于该 MIT 源码包。
