# HANDOFF — nightly backup job keeps flaking (originSessionId: 7f3a…, host: workstation)

## goal
Make the nightly rsync backup of the photo library reliable again. It has failed 3 of the last 7 nights.

## closed (don't reopen)
- It is NOT a disk-space problem: the target has 1.8 TB free, checked on 2026-09-10.
- Switching from rsync to restic was considered and rejected (restore drills would need rewriting).

## open
- Failures correlate with the NAS going to sleep around 03:00; the job starts at 03:15.
- The `--timeout` flag was added last week but the job still hangs on some nights.

## traps
- Do not run the job by hand during the day; it saturates the uplink and the household notices.
- The log rotates at 04:00, so read yesterday's log before then or it is gone.

## wake-condition
Next failed night (check the log).

## first-action-on-wake
Read the last failed log, confirm whether the hang is on the SSH connect or mid-transfer, then either move the schedule to 02:30 or add a wake-on-LAN step before the job.
