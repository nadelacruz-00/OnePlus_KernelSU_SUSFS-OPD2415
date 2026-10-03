# FORK-SETUP — OnePlus Pad 3 (OPD2415) only

Derivative of [WildKernels/OnePlus_KernelSU_SUSFS](https://github.com/WildKernels/OnePlus_KernelSU_SUSFS).
All credit to the WildKernels project.

## What was changed, and why

| # | Change | Reason |
|---|---|---|
| 1 | `configs/` pruned to **1** file (`a16/OP-PAD-3-SM8750-6.6.89.json`) | The matrix is built by scanning `configs/**/*.json`. Upstream ships **158** configs (71 for A16) with no "single device" option in the dispatch form, so `op_model: A16` would queue **71 parallel builds**. |
| 2 | `manifests/` pruned to **1** file | Same reasoning; only the Pad 3 manifest is referenced. |
| 3 | `.github/workflows/oplus-kernel-monitor.yml` deleted | 12-hourly cron that pushes a `status-page` branch and opens issues — pure noise here. |
| 4 | Weekly `schedule:` removed from `mirror-toolchains.yml` | Avoids mirroring 150+ toolchains on a timer. Kept for manual use. |
| 5 | `patches/custom_defconfig.txt` + **Apply Fork Custom Patches** step | Declares the full Droidspaces GKI config explicitly, right before the build. |
| 6 | `patches/required_config.txt` + **Verify Droidspaces kernel config** step | **Fails the build** if any required option is missing from `out/.config`. |
| 7 | `kernel-source-sync` cache repo → upstream public repo | Fork's own `toolchain-cache` release does not exist, and the lookup uses unauthenticated `curl`/`aria2c`, so a private repo would 404 and abort. |

## Why the config guard exists

The kernel running on this device (built **2026-06-30**) had **no `CONFIG_USER_NS`** — the
Droidspaces step at that time did not write it, and the option defaults to `n`. Nothing failed:
the build was green and the flash worked. The breakage only surfaced later as
`docker-compose up` being unable to start any container.

Measured on the device:

```
busybox unshare -U id   ->  unshare(0x10000000): Invalid argument    (CLONE_NEWUSER)
/proc/self/uid_map      ->  No such file or directory
```

`uid_map` only exists when `CONFIG_USER_NS` is compiled in. Upstream fixed it on
**2026-07-21** (`c054bfe9` "Remove USER_NS hardening" — the hardening patch forced `-EPERM`
for non-root callers). The guard step means this class of silent regression cannot ship again.

## Running a build

1. **Actions** tab → *Build and Release OnePlus Kernels* → **Run workflow**
2. Inputs: leave **everything at defaults**.
   `op_model` defaults to `A14+15+16`, which with the pruned tree resolves to exactly one job.
   Set `make_release: true` only if you want a GitHub release.
3. Artifact: `AK3_OP-PAD-3-SM8750-6.6.89_A16_6.6_KSUN_..._SuSFS_...zip`

No toolchain-mirror step is needed — the cache is served from upstream.

## Flashing safely (tablet-only, no PC)

The Pad 3 is A/B. Flash to the **inactive** slot so the working kernel is never touched.

```bash
# working bootctl extracted from the Kernel Flasher APK
/data/local/tmp/kfb/bootctl get-active-boot-slot     # -> 1 (slot B)
```

1. Back up first (Kernel Flasher → Backups, or raw):
   `boot_b`, `init_boot_b`, `vendor_boot_b`, `dtbo_b`, `vbmeta_b`
2. Kernel Flasher → **Slot A** card → View → **Flash** → the AK3 zip
3. `/data/local/tmp/kfb/bootctl set-active-boot-slot 0`
4. Reboot, test
5. Revert: `set-active-boot-slot 1` then `mark-boot-successful`
6. Verify inside Android: `su -c droidspaces check`

Slot state lives in the **GPT attributes** (bit 54 = `successful_boot`, bits 48-49 priority,
bits 50-52 `tries_remaining`), so a non-booting kernel should revert on its own — see
`../shared/opd2415-wildkernel-fork/gpt-slot-analysis/`.

## Open item after build #1

`/proc/cgroups` on the device lists only `cpuset cpu cpuacct blkio memory freezer net_prio`.
The Droidspaces doc calls the `devices` and `pids` controllers fatal, but its **GKI** block
omits them by design ("do not enable anything beyond this block").

If `droidspaces check` reports them missing, uncomment these two lines in
`patches/custom_defconfig.txt` and rebuild:

```
CONFIG_CGROUP_DEVICE=y
CONFIG_CGROUP_PIDS=y
```
