# How to create and run a persistent FreeBSD VM on macOS with QEMU

This guide walks through installing QEMU on macOS, creating a **persistent** virtual disk (and persistent UEFI variable store where applicable), installing FreeBSD once, then booting the same environment on every session. Day-to-day use is through the launcher script [`bin/freebsd-vm`](bin/freebsd-vm). To do the download and disk prep automatically (including optional Homebrew and QEMU install), use [`bin/setup-freebsd-vm`](bin/setup-freebsd-vm).

## Clone location and paths

You may **rename this directory or place it anywhere** (any parent path). The scripts resolve everything from **`bin/freebsd-vm`** and **`bin/setup-freebsd-vm`** themselves (including the path to this HOWTO when printing help).

Command examples use **`./bin/...`**: run them from the **project root for these scripts** — the single directory that contains both **`HOWTO.md`** and the **`bin/`** folder (in this layout, that directory is **`freebsd_vm/`** inside the git repo). For example, after `cd ~/path/to/HOWTOs/freebsd_vm`, run `./bin/freebsd-vm run`. If your clone only contains **`freebsd_vm/`** at the top of the repo, that folder is already the project root. The default VM files still live under **`~/VMs/freebsd/`** on your Mac, which is independent of where you keep this repository.

## What you get

- **Persistence**: Your installed system and UEFI boot configuration live in ordinary files under a state directory (by default `~/VMs/freebsd/`). Nothing is thrown away when QEMU exits, as long as you keep those files.
- **One command to launch**: From the **project root** (the directory that contains **`HOWTO.md`** and **`bin/`**), run **`./bin/freebsd-vm`** to start QEMU in the **same terminal** (no separate window): output and login use the serial console (`-nographic` / `-serial mon:stdio`). For a **macOS Cocoa** window instead, pass **`--gui`** or set **`FREEBSD_QEMU_GUI=1`**.

## Prerequisites

1. **Apple Silicon or Intel Mac** running a recent macOS.
2. **Xcode Command Line Tools** (compiler toolchain; QEMU does not always need it at runtime, but many macOS setups already have it):

   ```bash
   xcode-select --install
   ```

3. **Homebrew** from [https://brew.sh/](https://brew.sh/), then install QEMU:

   ```bash
   brew install qemu
   ```

4. Confirm the expected QEMU user-mode binary exists:

   ```bash
   uname -m
   ```

   - On **Apple Silicon** (`arm64`), you should have `qemu-system-aarch64`.
   - On **Intel** (`x86_64`), you should have `qemu-system-x86_64`.

   Optional: on Apple Silicon, check that Hypervisor.framework acceleration is advertised:

   ```bash
   qemu-system-aarch64 -accel help
   ```

   You should see `hvf` in the list.

## Setting up with `setup-freebsd-vm`

[`bin/setup-freebsd-vm`](bin/setup-freebsd-vm) is a **one-time (or repeat-with-`--force`) preparation** script. It does **not** start the VM; after it finishes, you use [`bin/freebsd-vm`](#using-freebsd-vm) to boot.

### Before you run it

1. **`cd` to the repository root** (the folder that contains `HOWTO.md` and `bin/`).
2. Ensure **`curl`** and **`xz`** exist on your Mac (Xcode Command Line Tools usually provide them).
3. Decide whether you need **`--install-brew`**: use it only if Homebrew is not installed yet. The script runs the official installer with **`NONINTERACTIVE=1`**; macOS may still ask for **sudo** once.

### What happens when you run `./bin/setup-freebsd-vm`

The script performs these steps **in order** (a **`--dry-run`** performs checks and HTTP probes only; see below):

1. **Parse options** — release, state directory, disk size, mirror, `--with-iso`, `--install-brew`, `--force`, `--dry-run` (see `./bin/setup-freebsd-vm --help` and environment variables at the end of the script).
2. **Pick download URLs** from your host architecture (`uname -m`): **Apple Silicon** uses the **aarch64** UFS VM qcow2 and optional arm64 **disc1.iso**; **Intel** uses the **amd64** UFS VM qcow2 and optional amd64 **disc1.iso**. Files are fetched from **`FREEBSD_MIRROR`** (default `https://ftp.freebsd.org/pub/FreeBSD`) and **`FREEBSD_SETUP_RELEASE`** (default **14.4**).
3. **`--dry-run` (optional)** — If set: print the resolved paths and URLs; confirm **`bin/freebsd-vm`** exists next to this script; check **`curl`** / **`xz`** / **`brew`** / **`qemu-img`** / **`qemu-system-*`** without installing anything; **`curl`** the VM (and ISO, if `--with-iso`) URLs and require **HTTP 200**; note whether a real run would refuse or rename an existing **`disk.qcow2`**; then **exit** without changing disks or Homebrew.
4. **Otherwise (real run):** require **`curl`** and **`xz`** or exit with an error.
5. **`mkdir -p`** on the state directory (default **`~/VMs/freebsd`**, or **`FREEBSD_QEMU_STATE_DIR`** / **`--state-dir`**).
6. **Existing disk guard** — If **`disk.qcow2`** already exists and you did **not** pass **`--force`**, the script **aborts** so it cannot overwrite your data. With **`--force`**, it **renames** the current **`disk.qcow2`** to **`disk.qcow2.bak.<timestamp>`** first.
7. **QEMU on the host** — If **`qemu-img`** or the correct **`qemu-system-aarch64`** / **`qemu-system-x86_64`** is missing, the script ensures Homebrew is available (install with **`--install-brew`** if needed), then runs **`brew install qemu`**.
8. **Download the compressed VM image** — **`curl -fL`** saves **`FreeBSD-<RELEASE>-RELEASE-…-ufs.qcow2.xz`** into the state directory.
9. **Decompress** — **`xz -d`** produces the **`.qcow2`** file, which is then **moved** to **`disk.qcow2`**.
10. **Resize** — **`qemu-img resize disk.qcow2 <size>`** (default **64G**, overridable with **`--disk-size`** / **`FREEBSD_SETUP_DISK_SIZE`**).
11. **Optional ISO** — With **`--with-iso`**, **`curl`** downloads **`disc1.iso`** into the state directory for later **`./bin/freebsd-vm install`**.
12. **UEFI vars** — **`rm -f edk2-vars.fd`** in the state directory so the next **`freebsd-vm`** boot creates or copies a **fresh NVRAM** file for the new disk.
13. **Print** absolute paths to the disk, optional ISO, **`HOWTO.md`**, and example **`./bin/freebsd-vm`** commands.

### Quick examples

Validate only (no downloads, no disk changes):

```bash
./bin/setup-freebsd-vm --dry-run --with-iso
```

Typical first install of the pre-built disk plus installer ISO:

```bash
./bin/setup-freebsd-vm --with-iso
```

Replace an existing disk after backup rename:

```bash
./bin/setup-freebsd-vm --force --with-iso
```

## Using `freebsd-vm`

[`bin/freebsd-vm`](bin/freebsd-vm) **starts QEMU** with the right **`qemu-system-*`**, **UEFI firmware** from your Homebrew QEMU install, **virtio** disk and network, and either a **terminal** console or a **Cocoa** window. It runs in the **foreground** until you quit QEMU or shut down the guest.

### Subcommands

| Command | Purpose |
|---------|---------|
| **`./bin/freebsd-vm`** or **`./bin/freebsd-vm run`** | Boot from **`disk.qcow2`** (no install ISO). |
| **`./bin/freebsd-vm install`** | Boot with install media attached so you can run **bsdinstall** (needs **`--iso`** or **`FREEBSD_QEMU_ISO`**). |
| **`./bin/freebsd-vm help`** | Show usage, environment variables, and the absolute path to this **`HOWTO.md`**. |

### Default UI: terminal (no extra window)

Unless you opt into GUI mode, the launcher passes **`-nographic`** and **`-serial mon:stdio`**, so the FreeBSD console appears in **the same terminal** you launched from. **QEMU monitor:** press **Ctrl-a**, then **c**; repeat to return to the guest serial stream.

### GUI mode (separate Cocoa window)

Pass **`--gui`** anywhere on the command line, or set **`FREEBSD_QEMU_GUI=1`**, to open a **macOS Cocoa** window with framebuffer, **USB keyboard and tablet** (XHCI), and **`full-grab=on`** so typing goes to the guest while the window is focused. Click inside the window before typing.

### Paths and persistence

The script reads **`FREEBSD_QEMU_STATE_DIR`** (default **`~/VMs/freebsd`**), **`FREEBSD_QEMU_DISK`**, and **`FREEBSD_QEMU_VARS`**. It refuses to start if **`disk.qcow2`** is missing unless you pass **`--create-disk [SIZE]`** to create an empty qcow2 first (used mainly for **ISO-first** installs). **`edk2-vars.fd`** is created on first boot (copy from QEMU’s template or a blank **64 MiB** image if the template is absent) and reused so **EFI settings persist**.

### Flags and passthrough

- **`--create-disk [SIZE]`** — Create **`disk.qcow2`** if missing (default size **64G**).
- **`--iso PATH`** — Install media for **`install`** mode (overrides **`FREEBSD_QEMU_ISO`** for that run).
- **`--`** — Everything after **`--`**, and unknown options before **`--`**, are passed **verbatim to QEMU** (for power users).

### Environment variables (common)

| Variable | Role |
|----------|------|
| `FREEBSD_QEMU_STATE_DIR` | Directory for **`disk.qcow2`** and **`edk2-vars.fd`** |
| `FREEBSD_QEMU_DISK` / `FREEBSD_QEMU_VARS` | Override disk or UEFI vars file paths |
| `FREEBSD_QEMU_RAM` / `FREEBSD_QEMU_CPUS` | Memory (MiB, default 4096) and SMP (default 4) |
| `FREEBSD_QEMU_ISO` | Default ISO path for **`install`** |
| `FREEBSD_QEMU_GUI` | Set to **`1`** for Cocoa |
| `FREEBSD_QEMU_SERIAL` | Set to **`1`** to force terminal mode (same as default) |
| `FREEBSD_QEMU_SHARE` | Host directory for optional **9p** `shared` mount tag |
| `FREEBSD_QEMU_NETDEV` | Replaces the default **user** networking **`-netdev` / `-device`** pair (space-separated tokens) |

Networking default: **user mode** with **host TCP 2222 → guest 22** for SSH. See the [Networking and SSH](#networking-and-ssh) section below.

## Where files live (state directory)

By default the launcher uses:

| File | Purpose |
|------|---------|
| `~/VMs/freebsd/disk.qcow2` | Main virtual disk (your FreeBSD install lives here) |
| `~/VMs/freebsd/edk2-vars.fd` | Writable **UEFI variables** image (boot order, EFI boot entries) |

You can override locations with environment variables (same names the launcher reads):

- `FREEBSD_QEMU_STATE_DIR` — directory for defaults below (default: `~/VMs/freebsd`)
- `FREEBSD_QEMU_DISK` — disk image path (default: `$STATE_DIR/disk.qcow2`)
- `FREEBSD_QEMU_VARS` — UEFI vars path (default: `$STATE_DIR/edk2-vars.fd`)

Create the state directory once:

```bash
mkdir -p "${HOME}/VMs/freebsd"
```

## Pick the correct FreeBSD media

Match the guest architecture to what the launcher selects for your Mac:

| Host `uname -m` | QEMU system target | Typical FreeBSD download |
|-----------------|--------------------|---------------------------|
| `arm64` | `qemu-system-aarch64` | `*-arm64-aarch64.iso` (or memstick image) |
| `x86_64` | `qemu-system-x86_64` | `*-amd64.iso` |

Official release files are linked from [https://www.freebsd.org/where/](https://www.freebsd.org/where/). Use a **recent RELEASE** ISO for the architecture you need.

**Download path (aarch64):** release ISOs live under `releases/arm64/aarch64/ISO-IMAGES/<version>/`, not under `releases/ISO-IMAGES/`. For a full installer, use the **`disc1.iso`** image (for example `FreeBSD-14.4-RELEASE-arm64-aarch64-disc1.iso`). Example base URL: [https://download.freebsd.org/releases/arm64/aarch64/ISO-IMAGES/](https://download.freebsd.org/releases/arm64/aarch64/ISO-IMAGES/) (mirrors such as [https://ftp.freebsd.org/pub/FreeBSD/releases/arm64/aarch64/ISO-IMAGES/](https://ftp.freebsd.org/pub/FreeBSD/releases/arm64/aarch64/ISO-IMAGES/) carry the same layout).

**Intel Mac note**: There is no KVM on macOS for `qemu-system-x86_64`, so the CPU is emulated (TCG). Expect **much lower performance** than on Apple Silicon with HVF; the same persistence model still applies.

## Create the persistent disk

The launcher expects a disk image to already exist (so it cannot silently create a tiny disk by accident). Create one explicitly — example **64 GiB** qcow2:

```bash
qemu-img create -f qcow2 "${HOME}/VMs/freebsd/disk.qcow2" 64G
```

Alternatively, the launcher can create the disk the first time if you pass `--create-disk` (optional size; default `64G`). You normally combine that with `install` when you are starting from an empty state directory:

```bash
./bin/freebsd-vm --create-disk 64G install --iso "${HOME}/VMs/freebsd/FreeBSD-14.4-RELEASE-arm64-aarch64-disc1.iso"
```

## UEFI firmware and why `edk2-vars.fd` matters

On both **AArch64 `virt`** and **`q35` x86_64** guests, the launcher uses QEMU’s **EDK II** firmware files shipped with Homebrew QEMU, usually under:

```text
$(brew --prefix qemu)/share/qemu/
```

Examples:

- Apple Silicon: `edk2-aarch64-code.fd` (read-only code) and, when present, `edk2-aarch64-vars.fd` (template for variables). Some Homebrew QEMU builds ship only `edk2-aarch64-code.fd`; in that case the launcher creates a **64 MiB blank raw** `edk2-vars.fd` on first boot so EDK2 can initialize NVRAM.
- Intel: `edk2-x86_64-code.fd` and, when present, `edk2-x86_64-vars.fd` (same fallback applies if the template file is missing).

On **first boot**, the launcher either **copies** the packaged `*-vars.fd` template into your state directory as `edk2-vars.fd`, or **creates** a blank 64 MiB raw file there if your QEMU package does not ship a template. Either way, that file is reused on every later boot so **EFI boot order and similar settings persist** across QEMU sessions.

## First-time install (attach ISO)

1. Download the correct FreeBSD ISO for your architecture.
2. Run the launcher in **`install`** mode and point it at the ISO (either environment variable or flag):

   ```bash
   export FREEBSD_QEMU_ISO="${HOME}/VMs/freebsd/FreeBSD-14.4-RELEASE-arm64-aarch64-disc1.iso"
   ./bin/freebsd-vm install
   ```

   Or explicitly:

   ```bash
   ./bin/freebsd-vm install --iso "${HOME}/VMs/freebsd/FreeBSD-14.4-RELEASE-arm64-aarch64-disc1.iso"
   ```

3. In your **terminal** (default) or in a **Cocoa** window if you used `--gui` / `FREEBSD_QEMU_GUI=1`, complete the FreeBSD installer and **install to the virtio disk** presented as the non-CD target (the installer should show both the install media and the target disk). Graphical installers are easier with `--gui`.

   Practical tips:

   - **UFS or ZFS** both work; pick what you know how to maintain.
   - Enable **sshd** in the installer’s network/user configuration if you want to SSH in from macOS later.

4. Shut down FreeBSD cleanly from inside the guest when the installer finishes (`shutdown -p now`).

## Daily use (boot installed system)

After installation, boot **without** install media:

```bash
./bin/freebsd-vm
```

Equivalent explicit form:

```bash
./bin/freebsd-vm run
```

QEMU runs in the foreground in your terminal by default. **QEMU monitor:** press **Ctrl-a** then **c** (press **Ctrl-a** then **c** again to return to the guest serial console). When QEMU exits, your disk and UEFI vars files remain on disk for the next session.

## Networking and SSH

The default launcher configuration uses **user-mode networking** with a TCP forward from the host to the guest’s SSH port:

- Host port **2222** → guest port **22**

If `sshd` is enabled in the guest and you are on the same machine:

```bash
ssh -p 2222 youruser@127.0.0.1
```

To replace the default netdev line entirely, set `FREEBSD_QEMU_NETDEV` to a **space-separated** list of QEMU arguments (for example a different `-netdev ...` and matching `-device virtio-net-pci,...`). Advanced setups can also append extra QEMU flags after `--`.

## Optional: host folder sharing (9p / `virtio-9p`)

Set `FREEBSD_QEMU_SHARE` to a host directory. The launcher adds a `virtio-9p` device with mount tag `shared`. In FreeBSD you still need to load the module and mount (exact module availability can vary by release):

```bash
kldload virtio_pci
kldload 9pfs
mkdir -p /mnt/host
mount -t 9p shared /mnt/host
```

If a command fails, consult the FreeBSD handbook for your release’s 9p/virtio notes.

## Terminal console (default)

The launcher does **not** open a separate GUI window unless you opt in. It uses `-nographic` and `-serial mon:stdio` so boot messages and login appear in the terminal where you ran **`./bin/freebsd-vm`**.

If you see no output after the firmware banner, configure the guest for a serial console (for example `comconsole` in `/boot/loader.conf` and a matching `ttyu0` entry in `/etc/ttys`). Exact lines depend on your FreeBSD version; see the handbook for your release.

## Optional: Cocoa GUI window

For a separate **macOS window** with framebuffer, keyboard, and pointer (helpful for `bsdinstall` or desktop use). The launcher attaches a **USB keyboard and tablet** (not virtio-keyboard), which tends to work reliably with **Cocoa** on macOS:

```bash
./bin/freebsd-vm --gui run
# or
export FREEBSD_QEMU_GUI=1
./bin/freebsd-vm run
```

The flag `--gui` can appear before or after `run` / `install` (it is parsed in an early pass).

## Optional: tune CPU and RAM

| Variable | Meaning | Default |
|----------|---------|---------|
| `FREEBSD_QEMU_RAM` | Memory (MB) | `4096` |
| `FREEBSD_QEMU_CPUS` | SMP vCPU count | `4` |

## Making `freebsd-vm` easy to invoke

Either call it by path, symlink it into a directory on your `PATH`, or add this repository’s `bin` directory to `PATH` in your shell profile:

```bash
# Run from the repository root (the directory that contains bin/ and HOWTO.md):
ln -sf "${PWD}/bin/freebsd-vm" "${HOME}/bin/freebsd-vm"
```

Ensure `${HOME}/bin` exists and is included in `PATH`. If you prefer an absolute path, resolve it once (for example `ln -sf "$(pwd)/bin/freebsd-vm" ...` right after `cd` to the clone).

## Persistence checklist

- Keep **`disk.qcow2`**; that is your installed system.
- Keep **`edk2-vars.fd`** alongside it; that is your **writable UEFI variable store** for the launcher’s EDK2 setup.
- Prefer **clean shutdown** from inside FreeBSD before closing QEMU to reduce filesystem inconsistencies.
- Back up the state directory if the data matters.

## Troubleshooting

- **`Missing .../edk2-aarch64-code.fd` (or x86_64)**  
  QEMU is installed but firmware files are missing or renamed. Inspect `$(brew --prefix qemu)/share/qemu` and adjust your QEMU package (reinstall `qemu` from Homebrew).

- **`qemu-system-aarch64 not found`**  
  Homebrew’s `bin` is not on your `PATH`, or QEMU is not installed. Run `brew install qemu` and ensure `$(brew --prefix qemu)/bin` is on `PATH`.

- **Installer does not see the disk**  
  Confirm you used the **matching** architecture ISO and that the disk image existed before launching `install` mode.

- **High idle CPU in the guest**  
  Some QEMU-on-macOS setups benefit from lowering the guest timer frequency, for example adding `kern.hz=100` to `/boot/loader.conf` inside the VM (test for your workload).

- **Very slow on Intel Macs**  
  Expected with TCG emulation; prefer lighter workloads or native x86 hardware for heavy builds.

- **GUI mode (`--gui`): keyboard or mouse does not reach the guest**  
  The launcher uses a **USB keyboard and tablet** on an **XHCI** controller plus `full-grab=on` so the Cocoa window receives keystrokes while it is focused. **Click inside the QEMU window** before typing. If your QEMU build errors on `full-grab=on`, append a display override after `--` (see *Extra QEMU arguments*).

## Extra QEMU arguments

Anything after `--`, or unknown flags before `--`, is forwarded to QEMU:

```bash
./bin/freebsd-vm -- -snapshot
```

(Be careful: options like `-snapshot` can change persistence behavior.)

## Reference: environment variables

Step-by-step behavior for the launcher is in [**Using `freebsd-vm`**](#using-freebsd-vm) above; setup script options are in [**Setting up with `setup-freebsd-vm`**](#setting-up-with-setup-freebsd-vm).

| Variable | Purpose |
|----------|---------|
| `FREEBSD_QEMU_STATE_DIR` | Default directory for disk and vars |
| `FREEBSD_QEMU_DISK` | qcow2/raw disk path |
| `FREEBSD_QEMU_VARS` | Writable UEFI vars file |
| `FREEBSD_QEMU_ISO` | Install media path (`install` mode) |
| `FREEBSD_QEMU_RAM` | RAM in MB |
| `FREEBSD_QEMU_CPUS` | vCPU count |
| `FREEBSD_QEMU_SHARE` | Host dir for 9p `shared` tag |
| `FREEBSD_QEMU_NETDEV` | Replaces default user networking line |
| `FREEBSD_QEMU_GUI` | Set to `1` to use a Cocoa window instead of the terminal |
| `FREEBSD_QEMU_SERIAL` | Set to `1` to force terminal/serial mode (same as the default; optional for scripts) |

## Reference: launcher subcommands

Run these from the repository root (see *Clone location and paths* above).

| Command | Description |
|---------|-------------|
| `./bin/freebsd-vm` / `./bin/freebsd-vm run` | Boot the persistent disk |
| `./bin/freebsd-vm install [--iso PATH]` | Boot with install media attached |
| `./bin/freebsd-vm help` | Print built-in usage |
