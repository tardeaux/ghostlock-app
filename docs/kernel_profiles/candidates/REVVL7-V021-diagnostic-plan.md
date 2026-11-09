# REVVL 7 V021 leak diagnostic plan (approval pending)

## Motivation

The first device gate accepted the profile and found futex collisions, but all
four KernelSnitch scans returned no `mm_struct`. The selected route never ran.
The exact V021 kernel allocates a 960-byte `mm_struct`, consistent with the
profile's 1024-byte stride. The available log does not distinguish a false
collision result, wrong futex-hash assumptions, wrong scan geometry, or
tagged-address behavior.

## Scope and behavior

1. In `src/core/support/util.cpp`, enable the existing KernelSnitch
   diagnostic output for a diagnostic build. Do not change the profile,
   collision selection, candidate matching, scan bounds, waiter/race/payload
   code, or retry counts. Keep the diagnostic change separate from any fix.
2. Build and run only after reviewing the output volume and obtaining approval
   for this attack-preparation-path change. The diagnostic build is not a
   release or supported-device build. Revert the diagnostic switch after the
   gate, retaining the evidence and analysis.

## Verification

Before another device run, run host tests, NDK build, lint, and the project's
`cmp_disasm.py` comparison of the eight attack functions against a clean
baseline build from the same source commit. Review any binary differences
caused by logging. For a device gate,
use a cold start, one explicit route and CPU pair, retain `profile.conf`,
`profile.bin`, and the native log, and record whether any kernel write or
panic occurred. Stop if the diagnostic output is excessive or the machine-code
review cannot establish that the attack sequence is preserved.

Use the diagnostic evidence to select a single, separately reviewed correction.
The failed gate alone does not justify changing the profile or registering it
as supported.
