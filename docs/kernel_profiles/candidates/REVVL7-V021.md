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

To complete support, verify the remaining approximate fields and physical
address mapping, then validate the selected route on this exact build. Only
then move the profile into `app/src/main/assets/kernel_profiles/`, register
it in `index.conf`, and complete the project's real-device gate before listing
the device as supported. No exploit attempt was made with this candidate.
