# GitHub Actions plan: Build ZealOS ISO + attach artifacts + (optional) PR automation

This repo builds ZealOS ISOs using `build/build-iso.sh`, which relies on QEMU, NBD, xorriso, and builds Limine + ZealBooter.

The goal of this GitHub Actions workflow is to:

1. Build both ISO variants on every PR / push to `master`
2. Upload the generated ISOs as workflow artifacts
3. (Optional) Add a manual workflow (`workflow_dispatch`) to run builds on demand
4. (Optional later) Add release automation to publish artifacts on tag

> Note: the ISO build script uses `sudo` + `qemu-nbd` (NBD kernel module). GitHub-hosted Ubuntu runners support `sudo`. Loading `nbd` generally works, but if GitHub runner restrictions change, we can switch to a loopback mount strategy or a container with privileged mode.

---

## Workflow 1: `build-iso.yml`

### Triggers
- `pull_request` (opened/synchronize)
- `push` on `master`
- `workflow_dispatch`

### Runner
- `ubuntu-latest`

### Steps

#### 1) Checkout
- `actions/checkout@v4`

#### 2) Install dependencies
Install the system packages required by `build/build-iso.sh`:

- build tools: `make`, `gcc`, `binutils`
- virtualization: `qemu-system-x86`, `qemu-utils`
- disk/mount: `nbd-client` (or `qemu-nbd` from qemu-utils), `parted`, `kmod`, `util-linux`
- ISO tooling: `xorriso`
- misc: `git`, `curl`

Example apt list:

- `build-essential`
- `make`
- `qemu-system-x86`
- `qemu-utils`
- `xorriso`
- `parted`
- `kmod`
- `util-linux`
- `git`
- `curl`

#### 3) Build ISO
Run:

```bash
cd build
./build-iso.sh --headless
```

Notes:
- The script produces two outputs in `build/`:
  - `ZealOS-PublicDomain-BIOS-YYYY-...iso`
  - `ZealOS-BSD2-UEFI-YYYY-...iso`

#### 4) Upload artifacts
Use `actions/upload-artifact@v4` to upload:
- `build/ZealOS-PublicDomain-BIOS-*.iso`
- `build/ZealOS-BSD2-UEFI-*.iso`

Suggested artifact names:
- `zealos-bios-iso`
- `zealos-uefi-iso`

#### 5) (Optional) Save build logs
Capture `build-iso.sh` output to a log file and upload it.

---

## Workflow 2 (optional later): `release.yml`

### Trigger
- `push` tags like `v*`

### Steps
- Run the same build
- Upload ISOs as GitHub Release assets (via `softprops/action-gh-release`)

---

## Workflow 3 (optional): PR comment with artifact links

On PR builds, post a comment containing:
- Run URL
- Artifact names

This can be done with `actions/github-script` using the built-in `GITHUB_TOKEN`.

---

## Security posture

- Do **not** store PATs in repo secrets for normal PR builds.
- Use the built-in `GITHUB_TOKEN` for commenting / PR metadata.
- If you later want cross-repo pushes or more permissions, prefer a GitHub App over PATs.

---

## Known risks / mitigations

1) **NBD / mount permissions**
- Risk: `modprobe nbd` might fail.
- Mitigation: Use `sudo modprobe nbd max_part=8` and ensure runner kernel supports it.
- Fallback: replace qemu-nbd approach with `guestmount`/libguestfs (heavier) or loop device if possible.

2) **QEMU acceleration**
- CI will run without KVM acceleration. The build uses QEMU to boot AUTO.ISO twice.
- Mitigation: keep QEMU memory/CPU settings moderate. Consider caching if build times become too long.

3) **Limine clone**
- Script clones limine binary branch `v10.x-binary`.
- Mitigation: can cache `build/limine` directory between runs.

