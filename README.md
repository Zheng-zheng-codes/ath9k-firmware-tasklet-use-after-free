# KASAN: slab-use-after-free in ath9k WMI event tasklet (tasklet_action_common, v7.4-rc2)

Syzkaller-found bug on Linux v7.4-rc2, reproduced and verified on the same
kernel build (fuzzing pipeline reproducer: confirmed=true, status=verified).

## Summary

A slab-use-after-free read fired in the ath9k WMI event tasklet softirq
path:

```
BUG: KASAN: slab-use-after-free in tasklet_action_common+0x2ec/0x3b0 kernel/softirq.c:988
Allocated by task 1259:
 ath9k_init_wmi+0x49/0x320 drivers/net/wireless/ath/ath9k/wmi.c:95
 ath9k_htc_probe_device+0x203/0x2650 drivers/net/wireless/ath/ath9k/htc_drv_init.c:956
Freed by task 1259:
 ath9k_htc_probe_device+0x292/0x2650 drivers/net/wireless/ath/ath9k/htc_drv_init.c:978
```

The freed object is the WMI control structure allocated by
`ath9k_init_wmi()` (drivers/net/wireless/ath/ath9k/wmi.c:95) during
`ath9k_htc_probe_device()` (htc_drv_init.c:956). The WMI event tasklet
(`wmi_event_tasklet`, registered by `ath9k_init_wmi()`) was scheduled by
an incoming USB message during probe. Subsequently, a later init step
failed and the error path (`err_init` -> `ath9k_destroy_wmi()`,
htc_drv_init.c:978) freed the WMI structure. The already-queued tasklet
fired after the free, triggering the use-after-free.

## How to reproduce

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, booted with `oops=panic` and
`panic_on_warn=1`. See `environment.txt` for the exact QEMU command line
and `kernel.config` for the configuration. The bug involves the ath9k_htc probe
error path: after `ath9k_init_wmi()` registers the WMI event tasklet, a
USB message schedules it, then the init failure path frees the WMI
structure before the tasklet fires.

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

- The freed object is the WMI control structure (allocated by
  `ath9k_init_wmi()`, wmi.c:95); the tasklet is `wmi_event_tasklet`
  (registered in the same function). The race is between the probe
  error-path cleanup (`ath9k_destroy_wmi()`) and an already-scheduled
  WMI event tasklet from an incoming USB message during probe.
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
