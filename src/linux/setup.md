---
minutes: 30
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Setup & Virtual Machine Environment

Developing kernel code in Rust requires a Linux kernel build environment configured with Rust support (`CONFIG_RUST=y`).

Because your host operating system's kernel version will rarely match our custom development kernel tree, we do not run `insmod` directly on the host machine. Instead, we test our code inside a **QEMU virtual machine** running a minimal Debian root filesystem (`debian.img`), with our host working directory shared into the VM via 9p/virtiofs.

## 1. Downloading & Setting Up the Toolchain

To ensure compatibility without conflicting with host package managers, download the prebuilt, kernel-tested LLVM and Rust toolchain tarball from `kernel.org`:

<!-- mdbook-xgettext: skip -->

```bash
# Download and extract the prebuilt LLVM + Rust toolchain:
curl -O https://mirrors.edge.kernel.org/pub/tools/llvm/rust/files/llvm-22.1.6-rust-1.97.0-x86_64.tar.xz
sha256sum llvm-22.1.6-rust-1.97.0-x86_64.tar.xz
tar xvf llvm-22.1.6-rust-1.97.0-x86_64.tar.xz

# Add the toolchain binaries and libclang to your environment:
llvm_prefix=$(realpath llvm-22.1.6-rust-1.97.0-x86_64)
export PATH=$llvm_prefix/bin:$PATH
export LIBCLANG_PATH=$llvm_prefix/lib/libclang.so
```

> [!NOTE]
> Ensure these environment variable exports are added to your `~/.bashrc` or shell profile so they persist across new terminal sessions when compiling or running QEMU later.

## 2. Workspace Setup & Minimal Debian VM Image

We recommend creating a dedicated workspace directory (`~/learn-rust`) and cloning your Linux kernel source tree into `~/learn-rust/linux` so your kernel git tree remains clean of test images and scripts:

<!-- mdbook-xgettext: skip -->

```bash
mkdir -p ~/learn-rust && cd ~/learn-rust

# Clone the development kernel branch for the course:
git clone --branch rfl-course-edu https://github.com/Darksonn/linux.git
```

To build a reusable Debian Bookworm disk image with `build-essential` (`gcc`, `make`) pre-installed and automatic 9p workspace mounting, save the following script as `create-image.sh` in your workspace directory (`~/learn-rust`) and run it:

```bash
#!/usr/bin/env bash
# create-image.sh - Creates a minimal Debian disk image for QEMU kernel testing
set -euo pipefail

IMG="debian.img"
DIR="/tmp/debian-mount-$$"
DISTRO="bookworm"

echo "Creating 2GB ext4 disk image: $IMG"
dd if=/dev/zero of="$IMG" bs=1M seek=2047 count=1
mkfs.ext4 -F "$IMG"

echo "Mounting $IMG at $DIR..."
mkdir -p "$DIR"
mount -o loop "$IMG" "$DIR"

echo "Installing minimal Debian ($DISTRO) with debootstrap..."
debootstrap --arch=amd64 \
    --include=build-essential,kmod,udev,procps,strace,gdb,vim,nano,pciutils \
    "$DISTRO" "$DIR" http://deb.debian.org/debian/

echo "Configuring hostname, fstab, and serial console autologin..."
echo "debian-vm" > "$DIR/etc/hostname"
mkdir -p "$DIR/mnt"
cat <<EOF > "$DIR/etc/fstab"
/dev/root / ext4 defaults 0 0
hostshare /mnt 9p trans=virtio,version=9p2000.L,defaults 0 0
EOF

# Allow passwordless root login on the serial console (ttyS0)
sed -i 's/^root:[^:]*:/root::/' "$DIR/etc/shadow"
mkdir -p "$DIR/etc/systemd/system/serial-getty@ttyS0.service.d"
cat <<EOF > "$DIR/etc/systemd/system/serial-getty@ttyS0.service.d/autologin.conf"
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin root -o '-p -- \\u' --keep-baud 115200,38400,9600 %I \$TERM
EOF

echo "Unmounting image..."
umount "$DIR"
rmdir "$DIR"

if [ -n "${SUDO_USER:-}" ]; then
    chown "$SUDO_USER" "$IMG"
fi

echo "Success! Created $IMG ready for QEMU."
```

<!-- mdbook-xgettext: skip -->

```bash
chmod +x create-image.sh
sudo ./create-image.sh
```

## 3. Configuring & Building the Kernel

Navigate into your `linux/` directory, verify toolchain support, and configure your kernel source tree with QEMU guest support (`kvm_guest.config`), Rust, sample modules, 9p filesystem sharing, and ACPI poweroff support:

<!-- mdbook-xgettext: skip -->

```bash
cd ~/learn-rust/linux

# Verify that the kernel build system recognizes the Rust toolchain:
make LLVM=1 rustavailable

# Start with standard x86-64 defaults optimized for QEMU VMs:
make LLVM=1 x86_64_defconfig
make LLVM=1 kvm_guest.config

# Enable Rust for Linux and the sample miscdevice driver:
./scripts/config --enable CONFIG_RUST
./scripts/config --enable CONFIG_SAMPLES
./scripts/config --enable CONFIG_SAMPLES_RUST
./scripts/config --module SAMPLE_RUST_MINIMAL

# Enable 9p filesystem support (required for sharing hostshare via /mnt):
./scripts/config --enable CONFIG_NET_9P
./scripts/config --enable CONFIG_NET_9P_VIRTIO
./scripts/config --enable CONFIG_9P_FS
./scripts/config --enable CONFIG_9P_FS_POSIX_ACL
./scripts/config --enable CONFIG_9P_FS_SECURITY

# Ensure ACPI poweroff and serial console are enabled:
./scripts/config --enable CONFIG_ACPI
./scripts/config --enable CONFIG_SERIAL_8250
./scripts/config --enable CONFIG_SERIAL_8250_CONSOLE

# Resolve remaining dependent options to defaults:
make LLVM=1 olddefconfig
```

Once configured, build the kernel and sample modules:

<!-- mdbook-xgettext: skip -->

```bash
make LLVM=1 -j$(nproc)
```

## 4. Booting QEMU with Workspace Sharing

Return to your parent workspace directory (`~/learn-rust`) and boot QEMU with `-machine q35,acpi=on`, `-no-reboot`, and `acpi=force` so that ACPI power management and host directory sharing work cleanly:

<!-- mdbook-xgettext: skip -->

```bash
cd ~/learn-rust
qemu-system-x86_64 \
    -machine q35,acpi=on \
    -kernel linux/arch/x86/boot/bzImage \
    -drive file=debian.img,format=raw,if=virtio \
    -append "root=/dev/vda console=ttyS0 acpi=force" \
    -nographic \
    -no-reboot \
    -m 2G -smp 2 \
    -virtfs local,path=$PWD,mount_tag=hostshare,security_model=none,id=hostshare
```

## 5. Testing Inside the Virtual Machine

Once booted, QEMU automatically logs you into a `root@debian-vm:~#` prompt on the serial console. Your host workspace is automatically mounted at `/mnt` via `/etc/fstab`:

<!-- mdbook-xgettext: skip -->

```bash
# Enter the shared workspace (mounted automatically at /mnt):
cd /mnt/linux

# Verify custom kernel version:
uname -r

# Load the sample miscdevice driver and test reading from it:
insmod samples/rust/rust_minimal.ko
rmmod rust_minimal

# Exit the virtual machine
poweroff
```

<details>

- **Workspace Layout:** Keeping `debian.img`, `create-image.sh`, and C test programs in the parent directory (`~/learn-rust`) ensures that `git status` inside `linux/` remains clean.
- **Why `q35,acpi=on` and `acpi=force`?** Passing these ACPI flags to QEMU and the kernel boot command line ensures that `poweroff` triggers ACPI sleep state S5 (`reboot: Power down`), cleanly exiting QEMU without hanging.
- **Automatic `/mnt` Mounting:** Because `/etc/fstab` maps `hostshare` to `/mnt`, both `/mnt/linux` and `/mnt/test_miscdev.c` are immediately accessible upon login.

</details>
