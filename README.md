# 20CRv3 on Cirrus — Quick Start

**Four-cycle October 2026 reference workflow **

This page reproduces the operational instructions and the **nine numbered steps** from the Version 2 Quick Start. Use a **Bash terminal on Cirrus**. Run one block at a time, in order; **stop at any FAIL**.

## Reference configuration

| Item                  | Reference value                                   |
| --------------------- | ------------------------------------------------- |
| Experiment            | `ensda_602_2026`                                  |
| Members / nodes       | 80 / 80                                           |
| Assimilation interval | 6 hours                                           |
| Initial analysis date | `2026101000`                                      |
| Four-cycle next date  | `2026101100`                                      |
| Initial restart       | 886 files, approximately 40.55 GB                 |
| Free space required   | 290 GiB before staging; 250 GiB before submission |

The validated shared installation is:

```
/epccfs/n02/n02-NEZ000203/shared/20crv3_cirrus_v1

```

**Files:** **How to use:** Copy one complete command block at a time into the **same Bash session** on Cirrus. Stop on any `FAIL`. If you reconnect, repeat Step 1 to restore your variables. The steps below are specific to the October 2026 reference inputs; they do not automatically configure a different date.

## Step 1 — Define your own paths (B0)

Defines private paths using your Cirrus login. Use **Bash**, and choose unused directories. `DATADIR/` must be no longer than 40 characters.

**Copy, paste and run this entire block:**

```bash
SHARED=/epccfs/n02/n02-NEZ000203/shared/20crv3_cirrus_v1
SRC_WF=$SHARED/run/coriII_home/20CRv3_scripts/ensda_602_2026
ME=$(id -un)
WORK=/epccfs/n02/n02-NEZ000203/$ME/20crv3_work/run1/ensda_602_2026
DATADIR=/epccfs/n02/n02-NEZ000203/$ME/L
DRIVER=gfsenkf_20crV3_sing_archer2_V47_cirrus_test4.csh
WF_FILES="dateutils3.py get_biascorr3.py $DRIVER incdate rewrite2.py"
WF_FILES+=" run_enkf_ps_h5_biascor_sing_archer2"
WF_FILES+=" run_fg_gfs_ens_sing_archer2 run_gefs_sa2"
DP=$DATADIR/gfsenkf_20crV3_archer2/ensda_602_2026
die() { echo "FAIL: $*" >&2; exit 1; } # called only inside parenthesised blocks
echo "PASS B0: variables defined for $ME"
```

**Expected:** `PASS B0: variables defined for <your login>`\
&#x20;**PASS:** Continue to Step 2.

**FAIL:** If variables are missing, repeat Step 1 in the same Bash session.

## Step 2 — Check access, paths and free space (B1)

Verifies project membership, unused directories, observations, SST/ice files, and at least **290 GiB** available before staging.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  case "$WORK" in *[[:space:]]*|*\#*) die "WORK must not contain whitespace or #";; esac
  case "$WORK/" in "/epccfs/n02/n02-NEZ000203/$(id -un)/"*) ;; *) die "WORK must be under /epccfs/n02/n02-NEZ000203/$(id -un)/";; esac
  case "$DATADIR/" in "/epccfs/n02/n02-NEZ000203/$(id -un)/"*) ;; *) die "DATADIR must be under /epccfs/n02/n02-NEZ000203/$(id -un)/";; esac
  id -nG | tr ' ' '\n' | grep -qx n02-nez000203 || die "not in group n02-nez000203"
  [ "$(basename "$WORK")" = ensda_602_2026 ] || die "workflow directory must end in ensda_602_2026"
  [ ! -e "$WORK" ] || die "$WORK already exists"
  [ ! -e "$DATADIR" ] || die "$DATADIR already exists"
  case "$DATADIR" in *[[:space:]]*|*\#*) die "DATADIR must not contain whitespace or #";; esac
  n=$(printf '%s' "$DATADIR/" | wc -c)
  [ "$n" -le 40 ] || die "datadir is $n characters including the slash; limit is 40"
  [ "$n" -eq 37 ] || echo "NOTE: datadir is $n characters; the validated length is 37"
  [ -r "$SHARED/MANIFEST_OPERATIONAL.sha256" ] || die "cannot read the shared installation"
  [ -r "$SHARED/inputs/restart/ensda_602_2026/analdate.csh" ] || die "cannot read the start state"
  for c in 2026101000 2026101006 2026101012 2026101018; do
    [ -r "$SHARED/inputs/obs/NRT/$c/psobfile" ] || die "observations missing for $c"
  done
  [ "$(ls "$SHARED/inputs/sst/SSTsNRT_CF/2026" | wc -l)" -eq 160 ] || die "SST/ice files incomplete"
  avail=$(df -Pk "$(dirname "$DATADIR")" | awk 'NR==2{print $4}')
  [ "$avail" -ge 304087040 ] || die "less than 290 GiB free before staging under $(dirname "$DATADIR")"
  echo "PASS B1 (datadir $n characters)"
)
```

**Expected:** `PASS B1 (datadir 37 characters)` for the default layout.\
&#x20;**PASS:** Continue to Step 3.

**FAIL:** Stop. Investigate access, paths, missing inputs, or insufficient free space. Existing directories are deliberately rejected.

## Step 3 — Verify shared executables and scripts (B2)

Checks SHA-256 hashes of 17 essential files in the shared installation.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  cd "$SHARED"
  RE='  \./(run/coriII_home/20CRv3_scripts/ensda_602_2026/[^/]+'
  RE+='|build/bin/(getsfcensmeanp\.x|getsigensmeanp\.x|global_postgpp|gribmeanp\.x|psop_gfs_biascor2be\.x)'
  RE+='|build/enkf/trunk/src/global_enkf_new'
  RE+='|build/ext_New2_bin/global_sighdr'
  RE+='|build/gfs/global_fcst\.fd\.v14\.0\.1/global_fcst_cmip5'
  RE+='|tree_B/gfs/global_cycle\.fd/global_cyclep)$'
  [ "$(grep -cE "$RE" MANIFEST_OPERATIONAL.sha256)" -eq 17 ] || die "manifest selection is not 17 files"
  grep -E "$RE" MANIFEST_OPERATIONAL.sha256 | sha256sum --quiet -c - || die "a shared runtime file has changed: stop and contact the installation owner"
  echo "PASS B2"
)
```

**Expected:** `PASS B2`\
&#x20;**PASS:** Continue to Step 4.

**FAIL:** Stop and contact the installation owner if files are missing or hashes differ.

## Step 4 — Copy eight workflow files (B3)

Creates your private workflow and copies exactly eight small scripts. The shared installation remains untouched.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  [ ! -e "$WORK" ] || die "$WORK already exists"
  mkdir -p "$(dirname "$WORK")"
  mkdir "$WORK"
  for f in $WF_FILES; do cp -p "$SRC_WF/$f" "$WORK/$f"; done
  [ "$(ls -A "$WORK" | wc -l)" -eq 8 ] || die "expected exactly 8 files in $WORK"
  for f in $WF_FILES; do cmp -s "$SRC_WF/$f" "$WORK/$f" || die "copy differs: $f"; done
  echo "PASS B3"
)
```

**Expected:** `PASS B3`\
&#x20;**PASS:** Continue to Step 5.

**FAIL:** Stop and inspect the partially created private directory. Do not rerun blindly.

## Step 5 — Configure the driver (B4)

Patches **only line 205** of the private driver, verifies unchanged lines and the corrected surface-cycle executable.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  cd "$WORK"
  [ "$(sha256sum < "$DRIVER" | cut -d' ' -f1)" = 7e5d9bef4c61f9d69af80ffddb128ecdca5b65c7b5d5f13fed5f71320ec85bd2 ] \
    || die "driver is not the validated original (already edited?): redo B3"
  LC_ALL=C sed -i "s#^set datadir=/epccfs/n02/n02-NEZ000203/shared/edr/\$#set datadir=$DATADIR/#" "$DRIVER"
  [ "$(sed -n 205p "$DRIVER")" = "set datadir=$DATADIR/" ] || die "line 205 is not the new datadir"
  diff <(sed 205d "$SRC_WF/$DRIVER") <(sed 205d "$DRIVER") >/dev/null || die "a line other than 205 changed"
  if [ "$DATADIR" = /epccfs/n02/n02-NEZ000203/ehawkins/L ]; then
    [ "$(sha256sum < "$DRIVER" | cut -d' ' -f1)" = f24a4fd7eba6155079a4811ed7e1b894d2b69691a602e6e5439cf8dbbc14f513 ] || die "edited driver hash differs from the audited value"
  fi
  for f in $WF_FILES; do [ "$f" = "$DRIVER" ] || cmp -s "$SRC_WF/$f" "$f" || die "$f differs from the shared copy"; done
  csh -n "$DRIVER" || die "csh syntax: $DRIVER"
  csh -n run_enkf_ps_h5_biascor_sing_archer2 || die "csh syntax: run_enkf_ps_h5_biascor_sing_archer2"
  csh -n run_fg_gfs_ens_sing_archer2 || die "csh syntax: run_fg_gfs_ens_sing_archer2"
  sh -n run_gefs_sa2 || die "sh syntax: run_gefs_sa2"
  hits=$(grep -nE 'j_bilbao|shared/edr|shared/x2c|shared/x2d|shared/edh' $WF_FILES | grep -vE '^[^:]+:[0-9]+:[[:space:]]*#' | grep -vxF "$DRIVER:205:set datadir=$DATADIR/" || true)
  [ -z "$hits" ] || die "active reference to a private or evidence path: $hits"
  grep -qx 'setenv CYCLEXEC /epccfs/n02/n02-NEZ000203/shared/20crv3_cirrus_v1/tree_B/gfs/global_cycle.fd/global_cyclep' "$DRIVER" \
    || die "CYCLEXEC is not the corrected X5c-B binary"
  echo "PASS B4"
)
```

**Expected:** `PASS B4`\
&#x20;**PASS:** Continue to Step 6.

**FAIL:** Stop. Never patch the shared driver.

## Step 6 — Copy and verify the initial restart (B5)

Stages the **886-file, \~40.55 GB** initial restart into your private data directory and verifies hashes.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  [ ! -e "$DATADIR" ] || die "$DATADIR already exists"
  [ "$(sed -n 205p "$WORK/$DRIVER")" = "set datadir=$DATADIR/" ] || die "driver not pointed at $DATADIR: complete B4 first"
  mkdir "$DATADIR"
  mkdir "$DATADIR/gfsenkf_20crV3_archer2" "$DATADIR/logs"
  rsync -rlpt "$SHARED"/inputs/restart/ensda_602_2026 "$DATADIR"/gfsenkf_20crV3_archer2/
  [ "$(ls -A "$DATADIR/gfsenkf_20crV3_archer2")" = ensda_602_2026 ] || die "unexpected layout under gfsenkf_20crV3_archer2"
  [ "$(find "$DP" -type f | wc -l)" -eq 886 ] || die "expected 886 staged files in $DP"
  [ "$(find "$DP" -type l | wc -l)" -eq 0 ] || die "unexpected symlinks in $DP"
  [ "$(grep -c '  \./inputs/restart/ensda_602_2026/' "$SHARED/MANIFEST_OPERATIONAL.sha256")" -eq 886 ] || die "manifest selection is not 886"
  ( cd "$DATADIR/gfsenkf_20crV3_archer2" &&
    grep '  \./inputs/restart/ensda_602_2026/' "$SHARED/MANIFEST_OPERATIONAL.sha256" |
    sed 's#  \./inputs/restart/#  ./#' |
    sha256sum --quiet -c -
  ) || die "start state differs from the manifest"
  [ "$(cat "$DP/analdate.csh")" = "setenv analdate 2026101000" ] || die "analdate.csh is not 2026101000"
  [ "$(cat "$DP/fg_only.csh")" = "setenv fg_only false" ] || die "fg_only.csh is not false"
  [ ! -L "$DATADIR" ] || die "$DATADIR is a symlink"
  echo "PASS B5"
)
```

**Expected:** `PASS B5`\
&#x20;**PASS:** Continue to Step 7.

**FAIL:** Stop and inspect partial staging. Do not repeat blindly or modify the shared restart.

## Step 7 — Check the login environment (C1)

Checks that the clean login module environment matches the validated configuration. The driver loads its own modules.

**Copy, paste and run this entire block:**

```bash
( set -euo pipefail
  for f in ~/.cshrc ~/.tcshrc; do [ ! -e "$f" ] || die "$f exists: show it to the reviewer before submitting"; done
  ref='craype-x86-turin:libfabric/2.3.1:craype-network-ofi:'
  ref+='perftools-base/25.03.0:xpmem/1.0.1-1.5_1_gfb6998056825:'
  ref+='cce/19.0.0:cse_env/0.2:craype/2.7.34:cray-dsmml/0.3.1:'
  ref+='cray-mpich/8.1.32:cray-libsci/25.03.0:'
  ref+='PrgEnv-cray/8.6.0:epcc-setup-env:load-epcc-module'
  got=$(env -i HOME=$HOME USER=$USER LOGNAME=$USER TERM=dumb bash --login -c 'echo "$LOADEDMODULES"' | tail -n 1)
  [ "$got" = "$ref" ] || die "clean-login modules differ from the validated account: $got"
  echo "PASS C1"
)
```

**Expected:** `PASS C1`\
&#x20;**PASS:** Continue to Step 8.

**FAIL:** Stop and investigate the module environment; Ed’s own login configuration has not been independently tested.

## Step 8 — Submit the 80-node job (D1)

**WARNING: This step submits a real 80-node exclusive Slurm job.** Obtain allocation approval, confirm the prerequisites, and execute only once. Do not run Step 8 merely to test the guide.

**Copy, paste and run this entire block:**

```bash
JOB=
if ( set -euo pipefail
  queue=$(squeue -h -u "$USER" -n 2026_602_gfsenkfs) || die "squeue failed: do not submit"
  [ -z "$queue" ] || die "a 20CRv3 job is already queued or running"
  [ "$(sed -n 205p "$WORK/$DRIVER")" = "set datadir=$DATADIR/" ] || die "driver datadir is not $DATADIR/"
  [ "$(cat "$DP/analdate.csh")" = "setenv analdate 2026101000" ] || die "not a fresh start state: restage (B5)"
  [ ! -e "$DP/2026101006" ] || die "$DATADIR has already been used: use a new datadir"
  [ -z "$(ls -A "$DATADIR/logs")" ] || die "$DATADIR/logs is not empty"
  [ ! -e "$WORK/walltime.out" ] || die "$WORK has already been used: use a new workflow directory"
  avail=$(df -Pk "$(dirname "$DATADIR")" | awk 'NR==2{print $4}')
  [ "$avail" -ge 262144000 ] || die "less than 250 GiB free: do not submit"
  echo "PASS D1-precheck"
); then
  submit="cd $WORK && sbatch --parsable"
  submit+=" -o gfsenkf_cirrus_%j.out"
  submit+=" -e gfsenkf_cirrus_%j.err $DRIVER"
  if JOB=$(env -i HOME="$HOME" USER="$USER" LOGNAME="$USER" TERM=dumb bash --login -c "$submit"); then
    JOB=${JOB%%;*}
    case "$JOB" in
      ''|*[!0-9]*) echo "FAIL: invalid job ID. Check 'squeue -u $USER' before retrying"; JOB=; false ;;
      *) echo "JOB=$JOB" ;;
    esac
  else
    JOB=
    echo "FAIL: sbatch failed. Check 'squeue -u $USER' before retrying" >&2
    false
  fi
else
  echo "FAIL: D1 prechecks failed; no submission attempted" >&2
  false
fi
```

**Expected:** `PASS D1-precheck`, followed by `JOB=<numeric Slurm ID>`\
&#x20;**PASS:** Continue to Step 9.

**FAIL:** Do not retry until checking the queue and logs. A precheck failure means no submission was attempted.

## Step 9 — Monitor the job (D2)

After Step 8 prints `JOB=<number>`, check its state. **Do not run Step 9 if Step 8 failed.** If you reconnect, repeat Step 1 in Bash and then set `JOB=<actual numeric job ID>` (replace the placeholder with the real number reported by Slurm).

**Copy, paste and run this status block:**

```bash
( set -euo pipefail
  [[ "${JOB:-}" =~ ^[0-9]+$ ]] || die "JOB must be the numeric Slurm job ID from Step 8"
  echo "Checking Slurm job $JOB"
  squeue -u "$USER"
  sacct -j "$JOB" --format=JobID,JobName%24,Elapsed,State,ExitCode
)
```

**Expected:** A queued or running job appears in `squeue`; after it finishes, check `sacct` for the **main job** reporting `COMPLETED` and `0:0`. A pending job can take time to start. If the job is `FAILED`, `CANCELLED`, `TIMEOUT`, or has a nonzero exit code, stop and investigate; do not resubmit into the same directories.

**Optional: watch standard output** (only after Step 8 returned a job ID):

```bash
[ -n "${JOB:-}" ] && [[ "$JOB" =~ ^[0-9]+$ ]] && tail -F "$WORK/gfsenkf_cirrus_$JOB.out"
```

Press **Ctrl-C** to stop following the file; this **does not cancel the job**. The output file may not exist until the job starts. If `tail` reports a missing file, check `squeue` and retry later. `tail -F` intentionally waits for new output; it is not a job deadlock.

**Optional: list cycle logs** (these may appear only after the job starts):

```bash
ls "$DATADIR/logs/ensda_602_2026/"
```

**This is the end of the quick start.** The guide gets the reference job submitted and monitored; a `COMPLETED|0:0` Slurm result is **not** by itself scientific validation. For detailed output and science checks, use the project manual on Cirrus:

```text
/epccfs/n02/n02-NEZ000203/shared/20crv3_cirrus_v1/docs/20CRv3_CIRRUS_USER_GUIDE_FINAL.md
```

## Important notes

- **Reference case only:** Initial analysis `2026101000`, four six-hour cycles, 80 members and 80 nodes. To run a different case, input data, dates, scripts, configuration and acceptance criteria must be reviewed; do not just change the date.
- **Storage:** Step 2 requires **290 GiB free** before staging; Step 8 requires **250 GiB free** before submission. The previously observed filesystem capacity was insufficient; Ed must recheck it. This is filesystem space, not RAM.
- **Fresh directories:** Steps 2, 4 and 6 deliberately refuse pre-existing work/data directories. Never delete an existing run to make a check pass without first identifying its contents.
- **Environment:** Step 7 compares against the validated login modules. Ed's login environment has not been tested independently; a mismatch requires investigation, not bypassing the check.
- **If a step fails:** Stop, record the exact command and output, and do not edit shared files or blindly retry `sbatch`.
