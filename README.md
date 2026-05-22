# Charybdis ZMK

How to build ZMK keyboard firmware locally without Docker on apple silicon mac
and Arch Linux.

> **_Note:_** This `README.md` is only about commands and instructions — you don't
> need to clone this repository to build yours locally.

Build time difference:

| Build Method         | Time            | Speedup        |
| -------------------- | --------------- | -------------- |
| GitHub Actions       | ~2 minutes      | 1x             |
| Local (Apple M1 Pro) | **~12 seconds** | **10x faster** |

> GitHub action is significantly slower because every action it executed on
> fresh VM environment.

## Table of Contents

- [Environment Setup](#environment-setup)
  - [1. Clone Your Keyboard Repository](#1-clone-your-keyboard-repository)
  - [2. Create Python venv](#2-create-python-venv)
  - [3. Install Dependencies](#3-install-dependencies)
  - [4. Install Zephyr SDK](#4-install-zephyr-sdk)
  - [5. Initialize West and Fetch Dependencies](#5-initialize-west-and-fetch-dependencies)
  - [6. Export Zephyr Environment](#6-export-zephyr-environment)
- [Build Commands](#build-commands)
- [Build Script](#build-script)

- [Arch Linux](#arch-linux)
  - [3. Dependencies](#3-dependencies)
  - [4. Zephyr SDK](#4-zephyr-sdk)

- [TroubleShooting](#troubleshooting)
  - [Configuring Incomplete](#configuring-incomplete-errors-occurred)
  - [Invalid SHIELD](#invalid-shield)

---

## Environment Setup

### 1. Clone Your Keyboard Repository

```sh
git clone https://github.com/unemotioned/charybdis-zmk
cd charybdis-zmk
```

---

### 2. Create Python venv

```sh
mkdir -p ~/venv # create dir for venvs
python3 -m venv ~/venv/zmk  # create zmk named venv under the ~/venv
source ~/venv/zmk/bin/activate
pip3 install -U pip  # update the pip itself
```

Go to [Arch linux step 3 - Install Dependencies](#3-dependencies)

---

### 3. Install Dependencies

```sh
brew install cmake ninja gperf python3 ccache qemu dtc wget openocd llvm
pip install west pyelftools
```

```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

---

### 4. Install Zephyr SDK

[GitHub - zephyrproject](https://github.com/zephyrproject-rtos/sdk-ng/releases/tag/v1.0.1)

Download the SDK for apple silicon with `wget` at home directory and install

```sh
cd ~

wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.17.0/zephyr-sdk-0.17.0_macos-aarch64.tar.xz
tar xvf zephyr-sdk-0.17.0_macos-aarch64.tar.xz

cd zephyr-sdk-0.17.0
./setup.sh

rm ~/zephyr-sdk-0.17.0_macos-aarch64.tar.xz
```

---

### 5. Initialize West and Fetch Dependencies

This initializes the project and downloads the required Zephyr/ZMK dependencies.

You may need to run `west update` multiple times before everything downloads successfully.

```sh
west init -l config

# this takes some time, and create some directories
#   which should be included in .gitignore
west update
```

---

### 6. Export Zephyr Environment

```sh
west zephyr-export
```

You may also need:

```sh
export ZEPHYR_TOOLCHAIN_VARIANT=zephyr
export ZEPHYR_SDK_INSTALL_DIR=$HOME/zephyr-sdk-0.17.0
```

which is in `zephyr_env.sh` that gets sourced by `build.sh` when executed

> **_NOTE:_** `west init -l config`, `west update`, `west zephyr-export` command
> must be run on each zmk repo if you have them separate.

---

## Build Commands

What each options do:

- `west build`: Runs the Zephyr/ZMK build system.

- `-d build/left`: Stores build output in the build/left directory.

- `-p always`: Performs a pristine build by cleaning previous build artifacts
  first. This helps avoid stale-cache issues when changing configs, shields, or
  board definitions. (**_You can remove this options is you only change keymaps
  file. Which sill reduce build time._**)

- `-b nice_nano_v2`: Targets the nice!nano v2 microcontroller board.

- `-s zmk/app`: Specifies the ZMK application source directory.

- `--`: Separates west build arguments from CMake arguments passed to Zephyr.

- `-DSHIELD="charybdis_left"`: Selects the keyboard shield to build. In this
  case, the left half of the Charybdis keyboard.

- `-DZMK_CONFIG="$PWD/config"`: Uses your custom ZMK configuration directory
  from the current project.

- `-DCMAKE_POLICY_VERSION_MINIMUM=3.5`: Forces compatibility with newer CMake
  versions (such as Homebrew-installed CMake on macOS). Some Zephyr modules
  still reference older policy versions, so this prevents configuration errors
  during build generation.

### Build with Display

- If your using [nice view](https://nicekeyboards.com/nice-view/) display change
  the keyboard name to `"charybdis_left nice_view"`
  or to `"charybdis_left nice_view_adapter nice_view`

---

### Build Left Half

```sh
west build \
    -d build/left \
    -p always \
    -b nice_nano_v2 \
    -s zmk/app \
    -- \
    -DSHIELD="charybdis_left" \
    -DZMK_CONFIG="$PWD/config" \
    -DCMAKE_POLICY_VERSION_MINIMUM=3.5
```

### Build Right Half

```sh
west build \
    -d build/right \
    -p always \
    -b nice_nano_v2 \
    -s zmk/app \
    -- \
    -DSHIELD="charybdis_right" \
    -DZMK_CONFIG="$PWD/config" \
    -DCMAKE_POLICY_VERSION_MINIMUM=3.5
```

### Build Settings Reset Firmware

```sh
west build \
    -d build/settings_reset \
    -p always \
    -b nice_nano_v2 \
    -s zmk/app \
    -- \
    -DSHIELD="settings_reset" \
    -DZMK_CONFIG="$PWD/config" \
    -DCMAKE_POLICY_VERSION_MINIMUM=3.5
```

#### Generated Firmware Files

The resulting firmware files will be located at:

- `build/left/zephyr/zmk.uf2`
- `build/right/zephyr/zmk.uf2`
- `build/settings_reset/zephyr/zmk.uf2`

---

## Build Script

1. Activate python venv
2. Source Zephyr SDK env var
3. Build left, right and reset files (reset file is commented out)
4. Backup previously built firmware files to `output/bak/`
5. Copy `.uf2` files to output directory at project root
6. Prompt to open the `output` directory (default: **_Yay_**)

execute with:

```sh
./build.sh
```

---

## Arch Linux

### 3. Dependencies

```sh
sudo pacman -S cmake ninja gperf python ccache qemu dtc wget openocd llvm base-devel
pip install west pyelftools
```

```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 4. Zephyr SDK

x86 architecture linux

```sh
cd ~

wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.17.0/zephyr-sdk-0.17.0_linux-x86_64.tar.xz
tar xvf zephyr-sdk-0.17.0_linux-x86_64.tar.xz

cd zephyr-sdk-0.17.0
./setup.sh

rm ~/zephyr-sdk-0.17.0_linux-x86_64.tar.xz
```

Go to [step 5 - Initialize West](#5-initialize-west-and-fetch-dependencies)

---

## TroubleShooting

### `Configuring incomplete, errors occurred!`

Remove `build` directory and then try running `build.sh` again.

---

### `Invalid SHIELD;`

If you have the `board/` outside `config/` then west will not find it and fail
at build time.

Add following option at the `$ west build` to make it look at `board/` under
project root.

```sh
west build -DBOARD_ROOT="$PWD"
```

Like this:

```sh
west build \
    -d build/left \
    -p always \
    -b nice_nano_v2 \
    -s zmk/app \
    -- \
    -DSHIELD="charybdis_left" \
    -DZMK_CONFIG="$PWD/config" \
    -DBOARD_ROOT="$PWD" \
    -DCMAKE_POLICY_VERSION_MINIMUM=3.5
```

In the `build.sh` change it to:

```sh
  west build \
    -d "$build_dir" \
    -p always \
    -b "$controller" \
    -s zmk/app \
    -- \
    -DSHIELD="$shield" \
    -DZMK_CONFIG="$ROOT_DIR/config" \
    -DBOARD_ROOT="$ROOT_DIR" \
    -DCMAKE_POLICY_VERSION_MINIMUM=3.5
```

> Or move `board/` under `config/`.

---

#### Happy Remapping 🎉
