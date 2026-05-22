# ESP32 firmware

A Fliphetic app can flash one or more ESP32 boards every time it loads. This
page explains how the cabinet flashes firmware, and how to produce a flashable
binary from the common toolchains: Arduino IDE, PlatformIO, ESP-IDF, and
MicroPython.

## How the cabinet flashes

When an app loads, the cabinet runs `esptool` for each `[esp32.<name>]` block in
the manifest. The command is, in effect:

```sh
esptool --chip <chip> --port <port> --baud <baud> \
        write_flash <flash_addr> <firmware> [<extra_addr> <extra_file> ...]
```

* `<chip>`, `<port>`, and `<baud>` come from the cabinet device registration
  and from the manifest.
* `<flash_addr>` and `<firmware>` come from the manifest.
* `<extra_*>` are optional additional images, also from the manifest.

The cabinet does not build firmware. You build it once, commit the resulting
binary to your repository, and the cabinet flashes that exact file.

## Two kinds of image

Understanding this is the key to a working firmware.

### App image

An **app image** contains only your program. It must be flashed at the
application offset, `0x10000` on a default partition layout. It only works if a
bootloader and a partition table are already present on the chip.

### Merged image

A **merged image** contains the bootloader, the partition table, and the app in
one file. It is flashed at offset `0x0` and does not depend on what was on the
chip before. This is the recommended choice for Fliphetic, because a load is
self contained and predictable.

You can always turn a set of separate images into a merged one with `esptool`:

```sh
esptool --chip esp32 merge_bin -o merged.bin \
    0x1000  bootloader.bin \
    0x8000  partitions.bin \
    0x10000 app.bin
```

The output `merged.bin` is then flashed at `0x0`.

## Flash addresses

| Image type | `flash_addr` in the manifest |
|------------|------------------------------|
| Merged image (built with `esptool merge_bin` or exported as merged) | `0x0` |
| App image, classic ESP32, default partitions | `0x10000` |
| MicroPython prebuilt, classic ESP32 | `0x1000` |
| MicroPython prebuilt, ESP32-S3 / C3 / C6 | `0x0` |

When in doubt, build a merged image and use `0x0`.

## Tutorial: Arduino IDE

1. Install the ESP32 board support. In Arduino IDE, open
   **File > Preferences**, add the Espressif board manager URL, then in
   **Tools > Board > Boards Manager** install **esp32 by Espressif Systems**.
2. Select your board under **Tools > Board**, and the correct port.
3. Write and verify your sketch.
4. Choose **Sketch > Export Compiled Binary**. The IDE writes the build output
   next to your sketch, in a `build/` subfolder.

Recent ESP32 cores export several files, including a `*.merged.bin`. Use that
one and flash it at `0x0`.

If your core does not produce a merged file, it exports the app, the
bootloader, and the partition table separately. Merge them yourself with the
`esptool merge_bin` command shown above, or list them as `extra` images in the
manifest.

## Tutorial: PlatformIO

1. Install the [PlatformIO](https://platformio.org/) extension for VS Code, or
   the PlatformIO Core CLI.
2. In your project, build:

   ```sh
   pio run
   ```

3. The app image is written to
   `.pio/build/<env>/firmware.bin`. The bootloader and partition table are in
   the same directory as `bootloader.bin` and `partitions.bin`.

To produce a merged image, run `esptool merge_bin` on those three files as
shown above, then commit the merged binary.

## Tutorial: ESP-IDF

1. Install [ESP-IDF](https://docs.espressif.com/projects/esp-idf/) and load its
   environment.
2. Build:

   ```sh
   idf.py set-target esp32
   idf.py build
   ```

3. The artifacts are in `build/`: the app at `build/<project>.bin`, the
   bootloader at `build/bootloader/bootloader.bin`, and the partition table at
   `build/partition_table/partition-table.bin`.

ESP-IDF can produce a merged image directly:

```sh
idf.py merge-bin -o merged.bin
```

Commit `merged.bin` and flash it at `0x0`.

## Tutorial: MicroPython

MicroPython has an important difference. The firmware binary is the MicroPython
interpreter. Your actual program is a set of `.py` files that normally live on
the board's filesystem, and Fliphetic does **not** upload files. So flashing a
plain MicroPython binary gives you a bare interpreter with no program.

There are two ways to use MicroPython with Fliphetic:

* **Frozen code.** Build a custom MicroPython firmware with your `.py` files
  frozen into the image (a build time manifest). The resulting `.bin` contains
  both the interpreter and your code, and behaves like any other firmware. This
  is the recommended path.
* **Prebuilt interpreter.** Flash the prebuilt firmware from
  [micropython.org/download](https://micropython.org/download/) and accept that
  your program must be put on the board another way. This does not fit the
  Fliphetic model well.

If you flash a prebuilt MicroPython image, mind the flash address: classic
ESP32 uses `0x1000`, while ESP32-S3, C3, and C6 use `0x0`.

For most cabinet projects, Arduino IDE, PlatformIO, or ESP-IDF are a better fit
because they produce a single binary that contains the whole program.

## Automating the build with GitHub Actions

Exporting firmware by hand from an IDE does not scale and is easy to forget.
Because the cabinet flashes a binary that is committed in the app repository,
the clean approach is a GitHub Actions workflow that builds the firmware and
commits the resulting binary back to the repository whenever the sources
change.

The watchdog then sees the new commit, and the next load flashes the fresh
firmware. Students never run a build by hand.

For this to work, keep the firmware sources and the built binary in the app
repository, for example:

```
firmware/
  src/                 your sources
  build/firmware.bin   committed by CI, referenced by the manifest
```

### PlatformIO in CI

PlatformIO is fully driven from the command line, so it automates cleanly. Add
this workflow at `.github/workflows/firmware.yml`:

```yaml
name: Build ESP32 firmware

on:
  push:
    branches: [main]
    paths:
      - "firmware/**"
      - "!firmware/build/**"
  workflow_dispatch:

permissions:
  contents: write

jobs:
  firmware:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install PlatformIO and esptool
        run: pip install platformio esptool

      - name: Build
        run: pio run --project-dir firmware

      - name: Merge into a single image
        run: |
          BUILD=firmware/.pio/build/esp32dev   # the env name from platformio.ini
          mkdir -p firmware/build
          esptool --chip esp32 merge_bin -o firmware/build/firmware.bin \
            0x1000  "$BUILD/bootloader.bin" \
            0x8000  "$BUILD/partitions.bin" \
            0x10000 "$BUILD/firmware.bin"

      - name: Commit the binary
        run: |
          git config user.name  "firmware-ci"
          git config user.email "firmware-ci@users.noreply.github.com"
          git add firmware/build/firmware.bin
          git diff --staged --quiet && echo "no change" && exit 0
          git commit -m "ci: rebuild ESP32 firmware [skip ci]"
          git push
```

Replace `esp32dev` with the environment name from your `platformio.ini`.

### ESP-IDF in CI

ESP-IDF is the C framework from Espressif. Espressif publishes a ready made
action that builds an ESP-IDF project inside the official toolchain container.

```yaml
name: Build ESP32 firmware

on:
  push:
    branches: [main]
    paths:
      - "firmware/**"
      - "!firmware/build/**"
  workflow_dispatch:

permissions:
  contents: write

jobs:
  firmware:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build with ESP-IDF
        uses: espressif/esp-idf-ci-action@v1
        with:
          esp_idf_version: v5.3
          target: esp32
          path: firmware
          command: idf.py build && idf.py merge-bin -o build/firmware.bin

      - name: Commit the binary
        run: |
          sudo chown -R "$(id -u):$(id -g)" firmware/build
          git config user.name  "firmware-ci"
          git config user.email "firmware-ci@users.noreply.github.com"
          git add firmware/build/firmware.bin
          git diff --staged --quiet && echo "no change" && exit 0
          git commit -m "ci: rebuild ESP32 firmware [skip ci]"
          git push
```

The build step runs in a container, so the output files may be owned by root.
The `chown` line hands them back to the runner user before `git add`.

### Why the binary is committed

The cabinet reads the `firmware` path from the manifest, relative to the
repository, and clones the app with a plain `git clone`. The binary must
therefore exist in the git tree. Committing it from CI keeps it reproducible
and in sync with the sources, while no one builds anything by hand.

### Avoiding a build loop

The workflow commits into `firmware/build/`. The `paths` filter excludes that
directory (`"!firmware/build/**"`), so the commit does not trigger the workflow
again. The `[skip ci]` marker in the commit message is a second safeguard.

### Manifest for a CI built image

Both workflows above produce a merged image, so flash it at offset `0x0`:

```toml
[esp32.buttons]
firmware   = "firmware/build/firmware.bin"
chip       = "esp32"
flash_addr = "0x0"
```

### Arduino projects in CI

If your firmware uses the Arduino framework, you do not need the Arduino IDE in
CI. Either build the Arduino project through PlatformIO (which supports the
Arduino framework) and reuse the PlatformIO workflow above, or use
`arduino-cli`, the command line companion to the IDE, in the same
build then commit pattern.

## Putting the binary in your repository

Commit the built binary into your app repository. A common layout:

```
firmware/
  src/                 your sources
  build/merged.bin     the binary you commit
  README.md            how to rebuild it
```

Add the rest of `build/` to `.gitignore`. Commit only the final binary, so the
cabinet flashes a reproducible artifact and loads stay fast.

## Manifest block

Reference the binary from `fliphetic.toml`. The key after `esp32.` is the name
of the cabinet device you target, as registered on the dashboard System page.

A merged image:

```toml
[esp32.buttons]
firmware   = "firmware/build/merged.bin"
chip       = "esp32"
flash_addr = "0x0"
required   = true
```

An app image with separate bootloader and partition table:

```toml
[esp32.buttons]
firmware   = "firmware/build/app.bin"
chip       = "esp32"
flash_addr = "0x10000"
extra = [
  { addr = "0x1000", file = "firmware/build/bootloader.bin" },
  { addr = "0x8000", file = "firmware/build/partitions.bin" },
]
```

Field reference:

| Field | Meaning |
|-------|---------|
| `firmware` | Path to the main binary, relative to the repository root. |
| `chip` | Target chip. Optional; defaults to the cabinet device's configured chip. |
| `baud` | Flashing baud rate. Optional; defaults to the device's configured baud. |
| `flash_addr` | Offset to flash `firmware` at. Default `0x10000`. |
| `extra` | A list of `{ addr, file }` images flashed alongside the main one. |
| `required` | If `false`, the load continues when the device is missing or the flash fails. Default `true`. |

Multiple boards: add one block per device, for example `[esp32.buttons]` and
`[esp32.lights]`.

An app that does not use any ESP32 simply omits all `[esp32.*]` blocks.

## Optional devices

Set `required = false` when a board is not essential. If the cabinet has no
device by that name, or the board is not plugged in, the load logs a warning
and continues instead of failing. Required devices that are missing abort the
load.

## Testing a firmware

Before relying on a load, you can flash by hand on the cabinet to confirm the
binary and addresses are right:

```sh
esptool --chip esp32 --port /dev/serial/by-id/usb-... --baud 921600 \
        write_flash 0x0 firmware/build/merged.bin
```

If this succeeds, the same parameters in your manifest will succeed too.

## The button protocol

Fliphetic flashes firmware but does not define how the firmware talks to your
app. The protocol (over USB serial, over the network, as emulated key presses,
or anything else) is decided between your firmware and your app. Document it in
your repository so the two halves stay in sync.

If the firmware reaches your app over the network, see
[Wi-Fi access point](wifi.md) for connecting an ESP32 to the cabinet's Wi-Fi
and reaching its services.
