# REVVL 7 V021 profile candidate

This is a candidate for the T-Mobile REVVL 7 5G (`TMRV075G`, `CypressPoint`),
not a built-in supported profile. The attached device reports
`5.10.236-android12-9-geb4f34fc554e` and build fingerprint
`T-Mobile/CypressPoint/CypressPoint:16/BQ2A.250925.001/523AA1_W_V021:user/release-keys`.
The matching stock `boot.img` has SHA-256
`0f7e97d9a8a10bb1452a17d6232eeb032d73a3d7e941682d7903cd2f18cbe3df`.
Its kernel banner matches `/proc/version` on the attached device.

The project's `ghostlock-extract` recovered 172,743 embedded kallsyms and
symbol offsets in the adjacent `.conf`. It reported the vulnerable
`remove_waiter` primitive present. The V021 image contains no embedded BTF,
the device exposes no `/sys/kernel/btf`, and its config has
`# CONFIG_DEBUG_INFO_BTF is not set`. ADB shell cannot read `/proc/iomem` or
`/proc/kallsyms` on this unrooted device.

For an approximation, the [Google GKI `android12-5.10-2025-05_r7` debug
`vmlinux`](https://ci.android.com/builds/submitted/14427738/kernel_debug_aarch64/latest/raw/vmlinux)
(build 14427738, SHA-256
`bc0bc5ea241c81fe95cac5a80303ba91996c7b64f1442b69fa7c60c4f98457bb`)
supplied DWARF structure layouts. It is **not** the REVVL kernel build: for
example, its `init_task` image offset is `0x279be80`, whereas V021 has
`0x219e000`. V021 disassembly independently confirms the following transferred
offsets: `task_struct.prio` `0x84`, `normal_prio` `0x8c`, `pi_lock` `0x86c`,
`pi_waiters` `0x880`, `pi_top_task` `0x890`, `pi_blocked_on` `0x898`,
`real_cred` `0x778`, `cred` `0x780`, `comm` `0x790`, `seccomp` `0x848`, and
`rt_mutex_waiter.task` / `.lock` `0x30` / `0x38`. The remaining task offsets
come from GKI DWARF alone and may differ on this vendor build.

V021 `prepare_kernel_cred` copies `0xa8` bytes. Its `init_cred` at
`0xffffffc00a1b4598` contains pointers at `0x80`, `0x88`, and `0x90` to
`root_user`, `init_user_ns`, and `init_groups`, respectively; each pointer
matches V021's own kallsyms. The credential capability and usage fields are
inferred from the GKI layout and the existing validated 5.10 profile. These
values still need safe device validation.

The vendor boot DTBs reserve physical memory starting at `0x80000000`, but
their `/memory/reg` entries are zero placeholders. They do not establish the
actual kernel load address. Both physical fields remain `null`; the current
native runtime therefore falls back to Qualcomm `kernel_phys_load = 0xa8000000`
and `kernel_phys_offset = 0x80000000`. The vendor boot reserved-memory map does
not overlap `0xa8000000`, but it cannot confirm the load address. The
`multicast_waiter.waiter_off` and `buffer_size` values were emitted as 5.x
extractor heuristics, not measured for this device. The selected route has
not passed a device gate.

On 2026-11-09, the device owner imported this candidate into the installed
GhostLock 1.2 app and reported **Profile ready**. That confirms the app's
configuration validation, not the physical mapping or exploit behavior.

## First device gate: FAIL (2026-11-09)

The owner ran GhostLock 1.2+591 once with this imported profile, using the
direct app entry point on the attached V021 device. The native process started
as UID 10254 in `untrusted_app` (seccomp mode 2). The selected CPU pair was
main 4 / consumer 5. The active profile was accepted (`invalid=0`), and the
runtime used the Qualcomm physical-address fallbacks `0xa8000000` and
`0x80000000`. This run tested commit `928fc466`'s profile candidate; the
installed app binary was not rebuilt from that commit.

The single W1 attempt stopped during heap preparation. All four preparation
tries reported futex collisions, but KernelSnitch returned
`0xffffffffffffffff` instead of an `mm_struct` address. The selected
`multicast_waiter` route did not execute, and there was no kernel write, W2,
W3, root child, or module handoff. Native exited with code 1 after about 13
seconds. The device remained connected through ADB with the same kernel; no
panic or reboot was observed. Repeated `sched_setaffinity(...): Invalid
argument` warnings appeared before the fourth preparation try, after three
leak failures. The log's `cpu=8` is the online CPU *count*, not a selected
core. ADB reports possible and online CPUs `0-7`. The warnings' cause and
effect are unresolved.

The full gate artifacts are retained privately at
`Revvl7-Recovery/ghostlock-device-gate/20261109-020542/` in the local
workspace, matching the device's
`Download/ghostlock-debug-log/20261109-020542/` directory:

| File | SHA-256 |
| --- | --- |
| `ghostlock-direct-0.log.txt` | `6ce5260c88e36e7c8f1cc7614bff58e92800605c4dd8ca75742d6e2f8f307d90` |
| `profile.conf` | `59c4951b7309956146c985e18db40586ebb1ae7c79cbe5d25da12bf11574e7d4` |
| `profile.bin` | `048e2690cf0c3848b32c6789e35191acd91c06cfa5cf31547aedb83ab8dab95c` |

Read-only disassembly of the exact V021 kernel's `mm_cache_init` indicates a
`0x3c0` (960-byte) `mm_struct` allocation, consistent with a 1024-byte slab
stride. The device kernel config enables hardware-tagged KASAN and SLUB
freelist hardening; their effect on this scan has not been established. This
does not identify why the leak failed. The physical mapping,
futex-hash assumptions, scan geometry, and other profile approximations
remain unverified. No profile value was changed in response to this one run.

To complete support, determine the leak failure with evidence, verify the
remaining approximate fields and physical mapping, then pass the selected
route on this exact build. Only then move the profile into
`app/src/main/assets/kernel_profiles/`, register it in `index.conf`, and
list the device as supported.
