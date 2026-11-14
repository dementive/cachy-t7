# cachy-t7

One script and one udev rules file that give every Samsung T7 running CachyOS
the same disk settings, so both systems behave the same.

## Use

```bash
./cachy-t7            # status: what each attached T7 has, OK or DIFF per setting
./cachy-t7 install    # copy the rules to /etc/udev/rules.d, apply them now, enable fstrim.timer,
                      # and on Limine add the XFS log buffer to the kernel command line (reboot)
./cachy-t7 trim       # trim every mounted filesystem now (once after the first install)
./cachy-t7 uninstall  # remove the rules; CachyOS's defaults apply again
```

`install`, `trim` and `uninstall` re-run themselves under `sudo`. Run the same
`install` on each system; the rules match any T7 by its USB id, not by serial,
so one file serves both drives. After a `git pull` of a changed rules file, run
`install` again: `status` says DIFF when the installed copy differs from the repo.

`install` first checks that every attached T7 reports TRIM (UNMAP) support: the
LBPU bit of the drive's Logical Block Provisioning page, which the kernel keeps
in `/sys/block/<dev>/device/vpd_pgb2`. It refuses otherwise. `install --no-trim`
installs everything but the TRIM line.

## What it sets, and why

| Setting | CachyOS / kernel default on a T7 | cachy-t7 | Why |
|---|---|---|---|
| `bdi/strict_limit` | 1 | 0 | CachyOS's `61-writeback-limits.rules` (cachyos-settings 1.4.4, 2026-11-11) caps every USB disk at 16 MB of unwritten data, enforced always. With the OS and builds on the T7, the kernel stalls writers: a 357 MB link took 336 s with it and 2.3 s without (measured 2026-11-14). |
| `bdi/max_ratio` | 1 (from the 16 MB cap) | 100 | The kernel default: under heavy writes the T7 may use the whole system dirty limit, like an internal SSD. |
| `queue/read_ahead_kb` | 65532 | 1024 | The T7's USB bridge reports a 32 MB optimal transfer size and the kernel reads ahead twice that on every file. Builds read thousands of small files. |
| `queue/scheduler` | mq-deadline | mq-deadline | CachyOS's own SSD choice, pinned so both systems match. |
| `provisioning_mode` (TRIM) | full (TRIM off) | unmap | Linux leaves TRIM off for USB disks, so `fstrim` never reaches the drive and sustained writes slow down as the SSD runs out of known-free blocks. |
| USB `power/control` | on | on | Never autosuspend the link of the disk the OS runs from. |
| `fstrim.timer` | enabled | enabled | The weekly trim, now that TRIM reaches the drive. |

The rules file is named `62-` so it runs after CachyOS's `61-` file and wins.
Nothing here touches other disks, other USB sticks, or CachyOS's files.

`status` also checks two mount options of `/` when it lives on a T7. Mount
options are `/etc/fstab`'s job and differ per system, so the script only
reports them:

- `noatime`: no write on every file read.
- `logbsize=256k` (XFS only): the log buffer. Builds create and delete thousands
  of small files, each a log entry; 256 KB buffers turn many small log writes
  into fewer large ones, which suits a USB disk.

  For `/`, putting it in `/etc/fstab` is not enough: the initramfs mounts `/`
  with the kernel's `rootflags` before fstab is read, and XFS cannot change
  `logbsize` on the later remount, so the fstab value is silently ignored. It
  goes on the kernel command line as `rootflags=logbsize=256k`.

  On Limine (CachyOS's default), `install` does this when `/` is XFS on a T7:
  it backs up `/etc/default/limine` (`.cachy-t7.<date>.bak` beside it),
  appends the flag inside the quotes of the `KERNEL_CMDLINE[default]` line,
  and runs `limine-update`; if that fails it restores the backup. It leaves
  the file alone when the flag is already there, or when other `rootflags`
  exist (add `logbsize=256k` to them by hand). Reboot to apply. `uninstall`
  removes the flag again. With another bootloader, `install` only prints what
  to add.

## Undo

`./cachy-t7 uninstall`, or delete `/etc/udev/rules.d/62-cachy-t7.rules` and
reboot.
