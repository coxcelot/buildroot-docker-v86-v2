# buildroot-docker-v86

Build a **Docker-enabled Buildroot Linux** for the **v86 x86-in-browser
emulator**, using free GitHub Actions cloud CI — and publish the images as a
**GitHub Release** with permanent, CORS-friendly URLs that the generator
fetches at runtime.

Docker 28.3.3 (`docker`), containerd, runc, libseccomp, iptables, and CA
certificates — as a 32-bit i686 Linux with a **separate `rootfs.cpio` initrd**
(v86 loads `bzimage` + `initrd` side by side; no initramfs embedding needed).

---

## What you do (once)

1. **Create a GitHub repo** and push this folder to it
   (e.g. `github.com/<you>/buildroot-docker-v86`).
2. **Actions** tab → enable workflows → run **"build-docker-v86"**
   (`workflow_dispatch`). The workflow needs **no secrets**; it auto-creates a
   Release when done.
3. Wait ~1.5–2.5h. On success, the Actions run creates a **GitHub Release**
   named `docker-v86-<n>` containing:
   - `bzImage` (~85MB) — the kernel
   - `rootfs.cpio` (~224MB) — the rootfs with containerd/runc/docker (raw cpio initrd)
   - `rootfs.cpio.lz4` (~92MB) — LZ4-compressed alternative

4. Open the Release → copy the `bzImage` and `rootfs.cpio` asset URLs
   (they look like `https://github.com/<you>/buildroot-docker-v86/releases/download/docker-v86-1/bzImage`).

5. In the **v86 generator's** `index.html`, fill in the `FILES` block:
   ```js
   bzimage: "https://github.com/<you>/buildroot-docker-v86/releases/download/docker-v86-1/bzImage",
   initrd:  "https://github.com/<you>/buildroot-docker-v86/releases/download/docker-v86-1/rootfs.cpio",
   ```
   (the generator auto-detects `initrd` and uses `root=/dev/ram0` + 512MB RAM).

---

## Why GitHub Release (not uploads.dev)

| | uploads.dev | GitHub Release |
|---|---|---|
| Max file | ~5 MB | **2 GB** |
| Total quota | 100 MB | unlimited (public) |
| CORS | ✓ | ✓ |
| Permanent URL | ✓ | ✓ |

The built images are ~85MB + ~224MB — far too big for uploads.dev or the
generator's `src/` storage. A GitHub Release on the same repo is the natural
home, and the workflow publishes it automatically.

---

## What the workflow does

| Step | Detail |
|------|--------|
| Free disk | removes runner bloat (needs ~20GB) |
| Download buildroot | `2024.02.5` (browser-buildroot pins this) |
| Merge config | `standard/.config` (glibc, kernel 6.6.37, i686) |
| Add Docker | `BR2_PACKAGE_DOCKER_ENGINE=y` + containerd/runc/docker-cli/libseccomp/CA-certs |
| Kernel options | `docker-kernel.config` merged into the kernel |
| Build | `make -j4` (toolchain + rootfs + kernel) |
| Release | `softprops/action-gh-release` uploads bzImage + rootfs.cpio |

---

## Notes / troubleshooting

- **Storage driver:** `S99docker` starts dockerd with `--storage-driver=vfs`
  (the only one that works on a RAM/initrd rootfs). Slow but functional.
- **cgroups:** buildroot 2024.02.5's docker-engine auto-selects
  `cgroupfs-mount` (cgroup v1). The `S99docker` script tries cgroup2 first,
  falls back to cgroup1.
- **Initrd size:** the raw `rootfs.cpio` is 224MB → v86 loads it into RAM at
  the 64MB boundary, so the VM needs ≥ 512MB. The generator sets this
  automatically when `initrd` is present.
- **Download time:** the generator must download ~310MB before boot. This is
  unavoidable for a Docker-in-browser setup.
- **Build time:** first run compiles the toolchain (slow). Re-runs rebuild
  only changed pieces; the disk-space triage frees ~20GB each run.

## Files

```
.github/workflows/build.yml   # CI: build + publish GitHub Release
standard/.config              # browser-buildroot "standard" config (glibc)
standard/board/browser_linux/ # board files: kernel config, overlay, S99docker
buildroot-docker.config       # the exact BR2_* Docker fragment (reference)
```

Credit: base config from [Darin755/browser-buildroot](https://github.com/Darin755/browser-buildroot)
(Buildroot 2024.02.5, kernel 6.6.37).
