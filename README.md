# HOWTOs

This folder is the **git repository root** for Cursor and for `git push`.

Each **subdirectory** is one guide. Inside it you will find **`HOWTO.md`** (the instructions) plus any **scripts**, **tests**, or other files used to carry out or support that guide.

## Table of contents

| How-to | Description | Quick start |
| --- | --- | --- |
| [FreeBSD on QEMU (macOS)](freebsd_vm/HOWTO.md) | Install QEMU on macOS, create a **persistent** FreeBSD disk (and UEFI state where needed), install once, then boot the same VM every time using **`bin/setup-freebsd-vm`** for setup and **`bin/freebsd-vm`** for daily use—serial console by default, optional GUI. | `cd freebsd_vm`<br>`./bin/setup-freebsd-vm --with-iso`<br>`./bin/freebsd-vm run`<br>Optional checks first: `./bin/setup-freebsd-vm --dry-run --with-iso` and `./bin/freebsd-vm help` |

## More detail

See each row’s **How-to** link for prerequisites, flags, environment variables, and alternate flows (for example **`--force`**, GUI mode, and ISO-only installs).
