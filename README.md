# freebsd-livecd-unionfs

A FreeBSD live ISO build system. Produces a bootable cd9660 image with a
mkuzip-compressed UFS rootfs and an in-kernel `unionfs(5)` writable
overlay backed by `tmpfs`, pivoted into via `init_chroot` (FreeBSD's
analog of Linux's `switch_root`).

## Download

**[Latest ISO →](https://github.com/pkgdemon/freebsd-livecd-unionfs/releases/tag/continuous)**

Continuous build, rebuilt automatically on every push to `main` after the
build + boot smoke-test both pass. Filename pattern:
`FreeBSD-<ver>-<arch>-unionfs-<YYYYMMDD>.iso` plus a matching `.sha256`.

## Architecture

```
cd9660 ISO  (kernel root, stays mounted forever, hidden from chroot)
  /boot/kernel/kernel.gz     gzipped kernel; loader auto-decompresses
  /boot/kernel/*.ko          ~8 essential modules (geom_uzip, unionfs,
                             virtio_*, ahci, acpi, mfi)
  /boot/loader.conf          loads geom_uzip + unionfs, sets
                             init_script=/init.sh + init_shell=/rescue/sh
  /sbin/init -> /rescue/init real FreeBSD init binary (statically linked)
  /rescue/                   statically linked busybox-equivalent;
                             includes mount_unionfs which the in-kernel
                             unionfs(5) needs at runtime
  /init.sh                   pivot script (silent in normal boot)
  /rootfs.uzip               compressed UFS rootfs (the real live system)
  /sysroot/                  empty mountpoint; becomes the merged view
  /upper/                    empty mountpoint for the writable tmpfs
  /EFI/BOOT/BOOTX64.EFI      UEFI loader (also inside the El Torito ESP)

Boot flow:

  loader -> kernel
  kernel mounts cd9660 as /
  /sbin/init runs from /rescue/init
  init reads init_script kenv, forks, execs /rescue/sh /init.sh

  /init.sh:
    mdconfig -t vnode -f /rootfs.uzip -u 0   ->  md0
                                                  (geom_uzip auto-tastes
                                                   /dev/md0 -> md0.uzip)
    mount -t ufs -o ro /dev/md0.uzip /sysroot   # read-only lower
    mount -t tmpfs tmpfs /upper                  # writable upper
                                                 # (no fixed size; pages
                                                 #  allocate on demand,
                                                 #  spill to swap)
    mount -t unionfs /upper /sysroot             # in-kernel union;
                                                 # /sysroot now exposes
                                                 # the merged writable view
    mount -t devfs devfs /sysroot/dev
    kenv init_chroot=/sysroot
    exit 0

  init re-reads init_chroot kenv on the next line of init.c, chroots into
  /sysroot, then continues to runcom -> /etc/rc -> multi-user.
```

The cd9660 stays mounted as the kernel's actual `/`. Userland sees the
unionfs merged view as `/`. The `/rootfs.uzip` file backing md0 lives on
the cd9660, which is never unmounted, so the vnode reference stays valid.
Only decompressed pages of accessed uzip data live in the page cache —
the full compressed image is never copied into kernel memory (unlike a
loader preload).

## Why this design

The naive "pivot via `reboot -r`" approach forces FreeBSD's
`vfs_unmountall(MNT_FORCE)`, which orphans any vnode-backed md and
breaks the overlay stack. Working around that requires preloading the
entire rootfs.uzip into kernel memory at boot — typically hundreds of
MB resident forever, plus a UEFI loader staging-area limit at scale.

`init_chroot` instead leaves the kernel's mount table alone. It works
because of the deliberate ordering at `sbin/init/init.c:326-336`:

```c
if (init_script kenv set)
    run_script(...)             // runs synchronously, blocks
if (init_chroot kenv set)       // RE-READ AFTER the script exits
    chroot(...)
```

The script can `kenv init_chroot=/sysroot` before exiting, and init reads
the value on the next line.

## Why unionfs + tmpfs (vs gunion)

This is the **file-level** overlay variant. The block-level alternative
(`gunion(8)`) lives in a sibling repo at
[freebsd-livecd-gunion](https://github.com/pkgdemon/freebsd-livecd-gunion).
This variant matches what Linux livecds do (file-level overlay, tmpfs
upper, RAM-scaled apparent disk size), using in-kernel `unionfs(5)`
rather than `unionfs-fuse` — same architectural model, kernel-speed
performance, no FUSE round-trips in the boot path.

## Writable headroom

The writable upper is a `tmpfs` mount with **no fixed size** — pages
allocate on demand from the VM subsystem, exactly like Linux's tmpfs.
Effective ceiling is host RAM + swap. `df` on the live system reports
free space from the tmpfs, so a 16 GiB-RAM machine sees substantially
more headroom than a 4 GiB-RAM machine without any rebuilds.

## Trade-offs vs Linux squashfs+overlayfs

- File-level overlay just like Linux. Copy-up is per file; small edits
  to many files are efficient.
- The cd9660 mount can't be removed during the live session (the live
  USB stick / CD has to stay attached). Linux livecds have the same
  limitation by default.
- FreeBSD's UEFI loader has a more rigid staging-area design than GRUB,
  so we *cannot* preload the rootfs through the loader. We mount it
  via mdconfig from cd9660 instead — same pattern Linux livecds use
  (`mount -o loop` from the iso9660), so this isn't a real handicap.

## Quickstart

Boot in qemu (UEFI):
```sh
qemu-system-x86_64 -m 4G -bios /usr/share/OVMF/OVMF_CODE.fd \
    -cdrom out/livecd.iso -boot d -nographic -serial mon:stdio
```

Boot under KVM for native speed (if `/dev/kvm` is available):
```sh
qemu-system-x86_64 -m 4G -accel kvm -cpu host \
    -bios /usr/share/OVMF/OVMF_CODE.fd \
    -cdrom out/livecd.iso -boot d -nographic -serial mon:stdio
```

Write to a USB stick:
```sh
sudo dd if=out/livecd.iso of=/dev/sdX bs=1M status=progress conv=fsync
```

## Building locally

Requires a FreeBSD 15.0+ machine or VM:
```sh
sh build.sh
ls -lh out/livecd.iso
```

Environment knobs:
- `FREEBSD_VERSION` (default `15.0`)
- `COMPRESS` (default `zstd`; `zlib` for FreeBSD 14 where mkuzip's
  zstd is broken — see [PR 267082](https://bugs.freebsd.org/267082))
- `LABEL` (default `LIVECD`)
- `ARCH` (default `amd64`)

## Building in CI

`.github/workflows/build.yml` runs the build inside `vmactions/freebsd-vm`
on `ubuntu-latest`. Each push produces an ISO artifact; a follow-up
job boots it in qemu (KVM if `/dev/kvm` is available, else TCG) and
asserts the live system reaches the getty `login:` prompt — that single
marker confirms the entire pipeline (loader → kernel → cd9660 mount →
init.sh → unionfs overlay → init_chroot pivot → multi-user) succeeded.

## Repository layout

```
freebsd-livecd-unionfs/
├── build.sh                  orchestrator (runs on FreeBSD)
├── ramdisk/init.sh           pivot script (lives at cd9660 root, silent
│                             at runtime)
├── boot/loader.conf          modules + init_script kenv
├── overlays/etc/rc.conf      live-system rc.conf
├── pkglist.txt               one pkg per line (empty = minimal base)
├── tests/boot-test.sh        qemu+expect smoke test (single login: marker)
├── .github/workflows/        CI
├── LICENSE                   BSD 2-clause, Joseph Maloney
└── README.md
```

## Further reading

[Architecture and design notes](https://pkgdemon.github.io/freebsd-livecd-plan.html)

## License

BSD 2-clause. Copyright (c) 2026, Joseph Maloney. See [LICENSE](./LICENSE).

This project bundles unmodified FreeBSD base and kernel artifacts at
build time; those remain under their original BSD 2-clause license
held by The FreeBSD Foundation and contributors.
