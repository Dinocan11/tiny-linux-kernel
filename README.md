# minisis — an extremely minimal Linux system

A from-scratch minimal Linux system built for QEMU: a custom-configured
Linux kernel paired with a statically-linked, musl-based BusyBox userspace,
booted via initramfs. No distro, no init system beyond BusyBox `ash`, no
unnecessary drivers.

Final boot artifacts are tiny:

- `bzImage` — kernel, built from `tinyconfig` with only what's needed
- `initramfs.cpio.gz` — ~200KB, single static BusyBox binary + symlinks

## What's included

- **Kernel**: Linux 7.2, built from `make tinyconfig` with the following
  added on top:
  - `CONFIG_64BIT`
  - `CONFIG_BLK_DEV_INITRD`
  - `CONFIG_DEVTMPFS` + `CONFIG_DEVTMPFS_MOUNT`
  - `CONFIG_BINFMT_ELF` + `CONFIG_BINFMT_SCRIPT`
  - `CONFIG_SERIAL_8250` + `CONFIG_SERIAL_8250_CONSOLE`
  - `CONFIG_PROC_FS` + `CONFIG_SYSFS`
- **Userspace**: BusyBox 1.37.0, statically linked against musl
  (`CC=musl-gcc`), ~336KB binary, ~35 applets (ash, ls, mount, init, vi,
  grep, find, ps, etc.)
- **Init**: a single shell script (`rootfs/init`) that mounts `/proc`,
  `/sys`, `/dev` (devtmpfs) and drops into a shell.

## Building it yourself

### 1. Prerequisites (Arch Linux)

```sh
sudo pacman -S base-devel ncurses openssl libelf bc cpio musl qemu-full
```

### 2. Build BusyBox (static, musl)

```sh
wget https://busybox.net/downloads/busybox-1.37.0.tar.bz2
tar xf busybox-1.37.0.tar.bz2
cd busybox-1.37.0
cp ../busybox.config .config      # use the config from this repo
make CC=musl-gcc -j$(nproc)
```

> Note: on modern GCC, BusyBox's `lxdialog` check script fails to build
> because it uses a K&R-style `main(){}` test program, which newer GCC
> rejects by default (`-Wimplicit-int` is now an error). If `make
> menuconfig` fails with a bogus "ncurses not found" message, wrap `gcc`
> to force an older standard, e.g.:
> ```sh
> printf '#!/bin/sh\nexec /usr/bin/gcc -std=gnu89 "$@"\n' > /tmp/gcc-wrap/gcc
> chmod +x /tmp/gcc-wrap/gcc
> PATH="/tmp/gcc-wrap:$PATH" make menuconfig
> ```

### 3. Assemble the root filesystem

```sh
mkdir -p rootfs/{bin,sbin,etc,proc,sys,dev}
cp busybox-1.37.0/busybox rootfs/bin/
cd rootfs/bin
for a in ash sh ls mount init cat chmod chown cp echo env head install \
         ln mkdir mv pwd rm sleep tail tty whoami which sed vi find grep \
         poweroff reboot hostname free kill killall ps tar unzip zcat; do
  ln -sf busybox "$a"
done
cd ../..
cp init.sh rootfs/init   # from this repo
chmod +x rootfs/init
```

### 4. Build the initramfs

```sh
cd rootfs
find . | cpio -o -H newc | gzip -9 > ../initramfs.cpio.gz
cd ..
```

### 5. Build the kernel

```sh
wget https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.2.tar.xz
tar xf linux-7.2.tar.xz
cd linux-7.2
cp ../kernel.config .config      # use the config from this repo
make -j$(nproc)
```

### 6. Boot it in QEMU

```sh
qemu-system-x86_64 \
  -kernel linux-7.2/arch/x86/boot/bzImage \
  -initrd initramfs.cpio.gz \
  -append "console=ttyS0 rdinit=/init" \
  -nographic -m 128M
```

Exit QEMU with `Ctrl-A` then `X`.

## Repo contents

```
kernel.config    — kernel .config used above
busybox.config   — BusyBox .config used above
init.sh          — the /init script run inside the initramfs
README.md
```

Kernel and BusyBox source are **not** vendored here — they're downloaded
from kernel.org / busybox.net as part of the build steps above.

## License

The files in this repository (configs, scripts, README) are released
under the MIT License — see `LICENSE`.

The Linux kernel and BusyBox are separate projects, both licensed under
GPL-2.0. This repo does not redistribute their source or binaries; it
only provides the configuration and glue needed to build them yourself.
