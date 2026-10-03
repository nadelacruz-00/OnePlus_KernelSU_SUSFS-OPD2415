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
| 8 | `retry_fetch_archive` handles commit SHAs; SUSFS fetch falls back to a GitHub mirror | GitLab's archive endpoint intermittently refuses GitHub runner IPs, which failed build #1. See below. |

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

## SUSFS source: GitLab first, GitHub mirror as fallback

Build #1 died after 2.5 minutes with:

```
 Attempt 1/3: High-speed fetch for susfs4ksu (a0f9c59e...)
##[error]All attempts failed for https://gitlab.com/simonpunk/susfs4ksu.git
```

The URL itself is fine — 200, valid tar — so this is GitLab's CDN intermittently refusing
GitHub runner IPs. A kernel build should not depend on that.

**Two changes:**

1. `retry_fetch_archive` now distinguishes a commit SHA from a branch. GitHub serves
   `/archive/<sha>.tar.gz` but **not** `/archive/refs/heads/<sha>.tar.gz` (404) — and the
   SUSFS ref is normally a 40-char SHA.
2. If GitLab fails, the step retries against a read-only **GitHub mirror**:
   <https://github.com/nadelacruz-00/susfs4ksu>

**Why a mirror at all, and why a single branch.** `git clone --mirror` of susfs4ksu is
**2.2 GB** — impractical. One branch is enough:

```bash
git clone --single-branch --branch gki-android15-6.6 --no-checkout \
    https://gitlab.com/simonpunk/susfs4ksu.git susfs4ksu   # 28.7 MB
cd susfs4ksu
git remote add gh https://github.com/nadelacruz-00/susfs4ksu.git
git push gh gki-android15-6.6
```

The pinned commit `a0f9c59e2243f8a5db955f4ad1686d5e0ad26e1a` is present in that branch, and
GitHub serves its archive because the commit is reachable from a ref. Verified:
`archive/<sha>.tar.gz` → 200.

**Provenance:** GitLab stays the *primary* source — it is the SUSFS author's own repo. The
mirror is only consulted after GitLab has failed outright. To drop the fallback, delete the
single `retry_fetch_archive "https://github.com/nadelacruz-00/susfs4ksu.git"` branch in
`action.yml`; nothing else depends on it.

To refresh the mirror later:

```bash
cd susfs4ksu && git fetch origin gki-android15-6.6 && git push gh gki-android15-6.6
```

## cgroup `devices` / `pids` — checked, NOT needed

`/proc/cgroups` on the device lists only `cpuset cpu cpuacct blkio memory freezer net_prio`,
so the `devices` and `pids` controllers are genuinely absent. The Droidspaces doc's
non-GKI table calls those fatal — but running the tool settles it:

```
$ su -c /data/local/Droidspaces/bin/droidspaces check
Droidspaces v6.6.0 - Checking system requirements...
  [MUST HAVE]     ... all present ...
  [RECOMMENDED]   ... [✓] Cgroup v2 support   [✓] Cgroup namespace ...
  [OPTIONAL]      ... [✗] Sandboxing (user namespaces)
                        CONFIG_USER_NS; enable per container with --allow-sandboxing.
                        Needed by unprivileged Docker and Podman, ...
Summary:
  [✓] All required features found!
```

**The only failing feature is `CONFIG_USER_NS`** — which is exactly what the build fixes.
Droidspaces v6.6.0 does not check for the cgroup controllers at all, and the GKI omission
in the doc is deliberate.

So `CONFIG_CGROUP_DEVICE` / `CONFIG_CGROUP_PIDS` are **not needed** and are not enabled.
Do not add them speculatively.

### After flashing: one per-container flag

The check output notes user namespaces are enabled **per container** with
`--allow-sandboxing`. So if `docker-compose up` still complains after the kernel is updated,
start the container with that flag.

