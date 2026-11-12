# KASAN: slab-use-after-free Read in tasklet_action_common (kernel/softirq.c:988, ath9k firmware release, v7.4-rc2)

Syzkaller-found bug on Linux v7.4-rc2, reproduced and verified on the same
kernel build (fuzzing pipeline reproducer: confirmed=true, status=verified).

## Summary

A slab-use-after-free read fired in the tasklet softirq path while the
ath9k wireless driver released its firmware:

```
BUG: KASAN: slab-use-after-free in tasklet_action_common+0x2ec/0x3b0 kernel/softirq.c:988
Call Trace:
 tasklet_action_common+0x2ec/0x3b0 kernel/softirq.c:988
 ...
 free_pages_prepare+0x3db/0xe70 mm/page_alloc.c:1448
 vm_area_free_pages+0x165/0x280 mm/vmalloc.c:3498
 vfree+0x11f/0x870 mm/vmalloc.c:3547
 free_fw_priv+0x3e1/0x4e0 drivers/base/firmware_loader/main.c:229
 release_firmware+0xa5/0xd0 drivers/base/firmware_loader/main.c:1122
 ath9k_hif_usb_firmware_cb+0x333/0x620 drivers/net/wireless/ath/ath9k/hif_usb.c:1276
 request_firmware_work_func+0x13b/0x440 drivers/base/firmware_loader/main.c:1164
```

The ath9k USB firmware callback (`ath9k_hif_usb_firmware_cb()`,
drivers/net/wireless/ath/ath9k/hif_usb.c:1276) called
`release_firmware()`; freeing the firmware memory (vfree path) raced with
a tasklet still referencing freed data — the KASAN read fired from
`tasklet_action_common()` (kernel/softirq.c:988) as the USB rx/tx tasklet
associated with the ath9k device ran concurrently with (or after) the
firmware teardown.

## How to reproduce

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, booted with `oops=panic` and
`panic_on_warn=1`. See `environment.txt` for the exact QEMU command line
and `kernel.config` for the configuration. The bug involves the emulated
USB device stack (the reproducer drives syz-usb connect/ disconnect
sequences against ath9k_hif_usb).

The crash depends on device/firmware state built up by the preceding
programs, so the reproducer is an ordered program set (50 programs):

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

Verified twice: (1) by the fuzzing pipeline's reproducer machinery on a
fresh VM (reproducer pipeline outcome: confirmed=true, status=verified);
(2) by an external standalone chunked-replay run on a fresh VM — the
KASAN use-after-free report fired 216.5 seconds after guest boot and
panic_on_warn escalated it to a kernel panic (full console:
`logs/external-verification/external-verification-serial.log`). A C
reproducer has not been included yet; it will be added once syzkaller's
C-repro stage passes for this bug.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (50-program ordered set + triggering testcase)
- `reproducer/repro.prog` — minimal triggering testcase (state-dependent; does not reproduce alone)
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports from the fuzzing pipeline
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics

## Notes for maintainers

- The freeing side is `release_firmware()` from
  `ath9k_hif_usb_firmware_cb()`; the reading side is the tasklet softirq.
  Confirming whether the freed object is the `firmware_priv` vm area or
  the usbdev-backed buffers requires the full KASAN report (included in
  the pipeline artifacts on request).
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
