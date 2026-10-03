# patches/

Files here are consumed by two custom steps added to
`.github/actions/build-kernel/action.yml`:

| Step | Consumes | When |
|---|---|---|
| **Apply Fork Custom Patches** | `*.patch`, `custom_defconfig.txt` | Before `Build Kernel` |
| **Verify Droidspaces kernel config** | `required_config.txt` | After `Build Kernel` |

## `custom_defconfig.txt`

Appended to `kernel_platform/common/arch/arm64/configs/gki_defconfig`,
immediately before `make gki_defconfig`. Because Kconfig honours the last
assignment, these values win over anything an earlier CI step wrote.

Contents: the full GKI block from the Droidspaces kernel configuration guide
(https://github.com/ravindu644/Droidspaces-OSS/blob/main/Documentation/Kernel-Configuration.md),
including the optional UFW / Fail2ban / tmpfs sections.

## `required_config.txt`

The build is failed if any option listed here is not `=y` in the generated
`out/.config`. This is the guard against silently shipping a kernel that is
missing a required feature.

## `*.patch`

Applied in filename order with:
```
patch -p1 --forward --fuzz=3 --no-backup-if-mismatch
```
against `kernel_platform/common`, right before the build runs. Patches must be
`-p1` diffs (`a/arch/arm64/...` style). Expect upstream drift - when a patch
stops applying, the build fails loudly and prints the `.rej` files on purpose.
