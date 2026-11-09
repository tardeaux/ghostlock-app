# REVVL 7 V021 profile candidate

This is a candidate for the T-Mobile REVVL 7 5G (`TMRV075G`, `CypressPoint`),
not a built-in supported profile. The attached device reports
`5.10.236-android12-9-geb4f34fc554e` and build fingerprint
`T-Mobile/CypressPoint/CypressPoint:16/BQ2A.250925.001/523AA1_W_V021:user/release-keys`.
The matching stock `boot.img` has SHA-256
`0f7e97d9a8a10bb1452a17d6232eeb032d73a3d7e941682d7903cd2f18cbe3df`.
Its kernel banner matches `/proc/version` on the attached device.

The project's `ghostlock-extract` recovered the embedded kallsyms table and
the symbol offsets in the adjacent `.conf`. It reported the vulnerable
`remove_waiter` primitive present. The image contains no embedded BTF, the
device exposes no `/sys/kernel/btf`, and its config has
`# CONFIG_DEBUG_INFO_BTF is not set`. ADB shell cannot read `/proc/iomem` or
`/proc/kallsyms` on this unrooted device. Consequently, the extracted profile
retains `null` for the unverified physical addresses, task/credential and
waiter structure geometry, and multicast task/lock offsets. Some emitted
values are extractor heuristics and also require device validation.

To complete support, derive every required `null` field from matching kernel
source or independently verified binary evidence, establish the physical
address mapping, and validate the selected route on this exact build. Only
then move the profile into `app/src/main/assets/kernel_profiles/`, register
it in `index.conf`, and complete the project's real-device gate before listing
the device as supported. No exploit attempt was made with this candidate.
