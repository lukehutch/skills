---
name: colab-offload
description: How to offload builds, test runs and other batch work to Google Colab VMs with the `colab` CLI, paid for with the compute units of a Google AI Ultra (or Colab Pro) plan. Covers installation, authentication, the commands, the measured VM shapes with their cost and the account limits, running long jobs without losing them to dropped connections, a reusable VM pool that stops idle VMs, budget planning, privacy, and the failures met in practice with their fixes. Use whenever work should run on Colab instead of the local machine, or when choosing, starting, using or stopping Colab VMs from a script or an agent.
---

# Offloading work to Google Colab with the `colab` CLI

This guide records what was measured on 2026-11-22 while running a large Rust workspace's
builds and tests on Colab VMs paid for with a Google AI Ultra plan's monthly Colab compute
units. Every number below was measured on that date unless marked otherwise. Colab changes
its shapes, prices and limits without notice, so re-measure anything a decision depends on
(section 6 shows how).

**Check the balance or buy more units** at <https://colab.research.google.com/signup>. The
balance can also be read from the command line: `colab --auth=adc usage` (section 3).

## 1. Install

The CLI is the PyPI package `google-colab-cli` (version 0.8.0 when this was written). It
needs Python 3.12 or later and runs on Linux and macOS. Home page:
<https://github.com/googlecolab/google-colab-cli>.

```bash
command -v colab || uv tool install google-colab-cli    # or: pip install --user google-colab-cli
colab --version
```

The binary lands in `~/.local/bin/colab`; make sure that directory is on `PATH`.

The CLI ships its own guide: `colab skill` prints its agent guide and `colab readme` its
README. Read both once; this file adds what they do not say.

## 2. Authenticate

Use Application Default Credentials (ADC). The Colab backends need four scopes, and gcloud
refuses a scope list without `cloud-platform`:

```bash
gcloud auth application-default login \
  --scopes=openid,https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/userinfo.email,https://www.googleapis.com/auth/colaboratory
```

Then pass `--auth=adc` on every call, **before** the subcommand: `colab --auth=adc new -s x`.
`colab --help` says the default is `oauth2`, while the bundled guide says it is `adc`;
passing it explicitly removes the question. The oauth2 route opens a browser consent page,
so it needs a person; ADC does not.

Check the login with `colab --auth=adc sessions` (read-only). A 401 or 403 on any call is
almost always a missing scope: rerun the gcloud command above. `colab auth` is a different
thing: it puts Google Cloud credentials *inside the VM* for notebook code. It does not fix
the CLI's own login.

The ADC login is independent of anything else on the machine (an AI assistant's own login,
for example), so switching those accounts does not affect running VMs.

## 3. Commands

All take `-s <name>`. Always name sessions yourself; an omitted name becomes a random hex
string and later commands become ambiguous.

| Command | What it does |
|---|---|
| `colab new -s N [--high-mem] [--gpu T4\|L4\|G4\|H100\|A100] [--tpu V5E1\|V6E1]` | Allocates a VM (billing starts) |
| `colab exec -s N [-f file.py] [--timeout S] [--env K=V]` | Runs Python on the VM's kernel; code from `-f` or stdin only: a command given as an argument is refused |
| `colab upload -s N <local> <remote>` / `colab download -s N <remote> <local>` | Copies one file |
| `colab ls -s N [path]` / `colab rm -s N <path>` | Lists / deletes files on the VM (`ls` lists files, **not** sessions) |
| `colab sessions` | Lists your live VMs, from the server |
| `colab status -s N` | Hardware, idle or busy, last execution |
| `colab stop -s N` | Releases the VM (billing stops) |
| `colab usage` | Balance, current burn rate, number of live VMs |
| `colab run [--gpu ..] script.py args` | `new` + `exec` + `stop` in one call; stops the VM even on error unless `--keep` |
| `colab ssh -s N -- CMD` | One shell command over ssh (needs a local ssh key; not used here) |
| `colab install -s N pkgs…` | pip install on the VM |
| `colab log -s N` | Recent events of the session; `-o x.ipynb` exports it as a notebook |
| `colab restart-kernel -s N` | Resets the kernel, keeps the VM |
| `colab url -s N` | A browser link that attaches the Colab web page to this VM |

There is no `colab list`; use `colab sessions`.

Example outputs (account details removed):

```
$ colab --auth=adc usage
Current balance: 4998.58 compute units
Usage rate: 0.52/hr
Active assignments: 2

$ colab --auth=adc sessions
[hm3] m-hm-ase1a0-3fumrwt3y82w8 | Hardware: CPU | Shape: High-RAM | Variant: DEFAULT
[hm2] m-hm-use1c0-2z9jucqkt0k7e | Hardware: CPU | Shape: High-RAM | Variant: DEFAULT
```

The two VMs above appear to be in different regions (`ase1` and `use1` in their ids, an inference); you
do not choose the region.

**The balance in `colab usage` lags** by many minutes. Do not use it to measure the cost of
one job; use the rate table in section 5 and the VM's lifetime.

### Running shell commands through `exec`

`exec` runs Python, but a cell whose first line is `%%bash` runs as a bash script. Pipe it on
stdin:

```bash
printf '%%%%bash\nnproc; free -g; df -h /\n' | colab --auth=adc exec -s hm1 --timeout 100
```

(`%%%%` is `printf`'s escape for `%%`.) This was the method used throughout; `colab ssh`
needs a local ssh key and the ssh path was not tested.

Three properties of `exec` decide how jobs must be written:

- **`--timeout` is seconds without output, default 30.** A build that prints nothing for 30 s
  fails the call. `--timeout 0` removes the limit, but see the next point.
- **The connection can drop.** A call can fail with `RuntimeError: Connection was lost`,
  and the command it carried may or may not have run. A job that lasts more than a minute
  must therefore not run *inside* an `exec` call: start it detached and poll it (section 4).
- **The kernel persists between calls**, so Python variables and `cd` inside Python survive.
  Each `%%bash` cell is a fresh bash process starting in `/content`.

Wrap every call in the local `timeout` command as well, so a hung client cannot block the
script: `timeout 120 colab --auth=adc exec -s N --timeout 110`.

### What a VM provides

Measured on the default CPU shapes: Ubuntu 24.04.5, Python 3.13, gcc, git, curl and apt-get
present, open internet access (crates.io, static.rust-lang.org and PyPI reachable), 226 GB of
free disk on `/` and `/content`, running as root. Not present: Rust, and the `time` program
(`apt-get install -y time` gives `/usr/bin/time -v` for peak memory). Installing Rust with
`curl -sSf https://sh.rustup.rs | sh -s -- -y --profile minimal` took 17 s on a 2-core VM.
`awk` is mawk, not GNU awk: it rejects gawk extensions (`match()` with an array argument,
`gensub`, `PROCINFO`) with an error on stderr and prints nothing, so a script that parses its own
logs with awk silently produces empty results there. Install gawk (`apt-get install -y -qq gawk`)
in the one-time setup when your scripts need it.
There is no `ld.lld` (a std VM has `cc` and `ld`; a v5e VM was seen without `ld` too), so any test
that links a native binary with LLD fails on every VM until it is installed. Check and install
on every job, not only in the one-time setup, so that VMs set up earlier get it too:
`command -v cc > /dev/null && command -v ld > /dev/null && command -v ld.lld > /dev/null || apt-get
install -y -qq gcc binutils lld` (with an `apt-get update -qq` retry). Measured: 18 s, Ubuntu LLD 18.1.3.
Ubuntu's LLD writes `Linker: Ubuntu LLD 18.1.3` in an executable's `.comment`, not the
`Linker: LLD <version>` of an upstream build (Fedora): a test that tells LLD from GNU ld by that
string must allow one vendor word before `LLD`.

What a VM can and cannot do (probed on std2 and l42, 2026-11-22). A VM is a docker container:
root with the identity uid map, capabilities docker's default set plus SYS_PTRACE and SYS_ADMIN
(no NET_ADMIN), `/sys/fs/cgroup` and `/proc/sys` read-only, no user systemd bus
(`systemd-run --user`: "Failed to connect to bus: No medium found"), no `ping`.
- Works: `unshare -Urn` with veth pairs and `tc ... netem` inside; bind mounts; clang, qemu-user
  and gdb-multiarch from apt (Ubuntu's `gdb` has no aarch64 or riscv); podman rootless, as a
  user of its own, with `containers.conf` holding `default_sysctls = []`, `pidns = "host"`,
  `default_capabilities` = podman's eleven plus `NET_RAW` (for ping in a container; the eleven of
  the VM's podman 4.9.3, from Ubuntu's containers-common 0.57.4 `containers.conf`, are those of
  podman 5.8.9: CHOWN, DAC_OVERRIDE, FOWNER, FSETID, KILL, NET_BIND_SERVICE, SETFCAP, SETGID,
  SETPCAP, SETUID, SYS_CHROOT; read on l43 2026-11-22), and
  `cgroup_manager = "cgroupfs"`, after `mknod -m 666 /dev/net/tun c 10 200`. Measured: runs,
  volumes, two containers on a podman network, image builds.
- Fails: podman as root (netavark "Netlink error: Operation not permitted", crun
  "cgroup.subtree_control ... Read-only file system"); an overlay mount whose upper directory is on
  `/content` (itself overlayfs: "wrong fs type, bad option"), as root or in `unshare -Urm`. An
  overlay with its upper and work directories on a tmpfs mounts, `index=on` included; so does it
  inside `unshare -m --propagation private` as root, with the checkout bound at a fixed path and a
  tmpfs over its build directory (not over the clone directory itself: a mount point cannot be
  renamed, "Device or resource busy", and a tool that drops a stale clone by renaming it fails),
  and a second mount of the same upper directory is refused
  ("overlay already mounted"), as on Linux generally (l43, kernel 6.6, 2026-11-22). The namespace
  keeps these mounts from every other job on the VM and drops them when its process ends.
- Fails: `open(dir, O_TMPFILE | O_RDWR)` in the VM's temporary directory, which is on the overlay
  root: EOPNOTSUPP (Linux 6.6, 2026-11-22). A test that needs an unnamed file falls back to
  `memfd_create` on EOPNOTSUPP. For a test of `RLIMIT_DATA` the substitute is exact: the kernel
  counts a mapping against the data limit by its flags alone, not by the file behind it
  (`may_expand_vm` in mm/mmap.c). Another substitute may change what a test measures; check the
  kernel source before using one. A test that passes locally and fails only on a VM may be
  hitting a file system feature the overlay lacks.
`job.sh` below installs these tools once per VM (marker `/content/setup-full-3-done`).

## 4. Running a long job reliably

The pattern that worked:

1. **Pack the inputs locally** as one tarball each (`git ls-files -z | tar --null -T - -czf
   src.tgz`) and `colab upload` them. One archive uploads far faster and more reliably than
   many files; the Colab FAQ gives the same advice.
2. **Write the job as a script** (`run.sh`), upload it, and **start it detached** in its own
   process group, recording the group id:

   ```bash
   printf '%%%%bash\n[ -s /content/job/pgid ] || { setsid nohup bash /content/job/run.sh %s > /content/job/run.log 2>&1 & echo $! > /content/job/pgid; }\n' "$TOKEN" |
     timeout 120 colab --auth=adc exec -s "$VM" --timeout 110
   ```

   The `[ -s pgid ] ||` guard makes the launch safe to repeat. It would also block the next job
   on the same VM, so each job first clears `/content/job` the way `release` does (kill the old
   group, delete the directory); without that, a second job on a held VM never starts and is
   reported dead.
3. **Confirm the launch with a separate call**, and retry the launch up to three times if it
   is not confirmed. A launch call that loses its connection may never have run; without
   this check the poller below waits forever for a job that never started.

   ```bash
   printf '%%%%bash\n[ -s /content/job/pgid ] && echo "LAUNCH""ED"\n' | timeout 60 colab --auth=adc exec -s "$VM" --timeout 50 | grep -q LAUNCHED
   ```

4. **Poll for a done file named with a per-job token**, and detect a dead job:

   ```bash
   while :; do
     r=$(printf '%%%%bash\nif [ -e /content/job/DONE-%s ]; then echo "JOB""DONE"; elif kill -0 $(cat /content/job/pgid) 2>/dev/null; then echo "RUN""NING"; else echo "JOB""DEAD"; fi\n' "$TOKEN" |
         timeout 60 colab --auth=adc exec -s "$VM" --timeout 50)
     case $r in *JOBDONE*) break;; *JOBDEAD*) echo "job died"; exit 5;; esac
     sleep 20      # a failed poll (dropped connection) just tries again
   done
   ```

   - `"JOB""DONE"` is split in the source on purpose. `exec` may echo the cell's source in
     its output, and a plain `echo JOBDONE` in the source then matches `*JOBDONE*` at once,
     ending the wait before the job is done. Splitting the string means only the *printed*
     word matches.
   - The token (for example `date +%s%N`) makes the done file unique to this job, so a done
     file left by an earlier job on a reused VM cannot end this wait.
   - Without the dead-job check, a job that crashed before writing its done file makes the
     poller loop forever.
5. **Have the job pack its results** into one tarball, then write the done file last. Download
   the tarball and unpack it locally.
6. **Clean up per job, not per VM**: kill the job's session (`pkill -s $(cat pgid)`; the job
   was started with `setsid`), not only its process group: a `timeout` inside the job moves its
   command into a process group of its own, which `kill -- -$(cat pgid)` does not reach. Then
   delete `/content/job`. Keep the toolchain and build cache (for example
   `CARGO_TARGET_DIR=/content/target`) so the next job on the same VM builds incrementally.
   When jobs from different checkouts share that cache, never build in the freshly unpacked
   copy: `tar` restores the original mtimes, so a file edited before another checkout's last
   build looks older than that build, and cargo (like make) silently reuses the other
   checkout's code. Copy into a fixed build directory with `rsync -rlpc --delete` (contents
   compared by checksum, no `-t`), so that only changed files get a new mtime.
   Killing the process group matters: a job still running when its VM is handed on would write
   into the next job's directory and mix its results into the new job's.

### Memory-heavy test runs: back off on out-of-memory

Run each test command with a thread count, and when it dies of out-of-memory, run it again
with half the threads, down to 1. Record the peak memory with `/usr/bin/time -v`:

```bash
t=8
while :; do
  /usr/bin/time -v -o $n.time timeout 3600 cargo test $line -- --test-threads=$t > $n.log 2>&1; rc=$?
  if [ $rc -ne 0 ] && [ $rc -ne 124 ] && [ $t -gt 1 ] && grep -q -E 'SIGKILL|signal: 9|memory allocation of .* failed' $n.log; then
    t=$((t / 2)); continue
  fi
  break
done
awk '/Maximum resident/ {print $6}' $n.time     # peak RSS in kB
```

This exposed a real problem worth knowing about. One Rust test binary peaked at 40 GB on a
single thread, and out-of-memory at 51.5 GB with 8 threads, although no single test in it
used more than 3.0 GB. Measured so far: calling `malloc_trim(0)` at the end of each test
thread (in the test harness only) cut a two-test process from 4.17 GB to 2.49 GB, so glibc's
allocator does keep a finished test's freed memory, but that was not the main cause. The
cause, found 2026-11-22: a VM has no hardware instruction counter, so the scaling meter fell
back to processor time and ran each size 15 times, and the scaling test helpers built all 30
inputs (15 per size, two sizes) before the first run and kept every one to the end.
`fits_sig_takes_linear_time_in_the_depth` alone held 37.1 GB, `meets_of_shared_tuples` 32.5 GB.
Limiting glibc to one or two arenas (`MALLOC_ARENA_MAX=1` or `2`) left the peak unchanged, so
arenas were ruled out. Building the inputs in batches of 5 cut those two to 11.0 GB and
9.3 GB; the fix that landed builds each input just before its run, outside the measured time,
and drops it right after, and a test counts the inputs alive at once (with the fix disabled it
caught 10 against 1 allowed). A meter that repeats each size multiplies whatever is held per
repeat. If a test binary's peak is far above its largest test, record the resident memory
after each test in one process before buying a bigger VM: steady growth means a leak or a
cache, a step per new thread means arenas, a step per repeat means held inputs.

Two cautions when measuring memory. `Maximum resident` from `/usr/bin/time -v` is the peak of
the largest single process, not the run's total, and scaling checks often run in child
processes. A sampler or trace must follow the test process and its children: one that watched
the cargo parent read nothing useful, and children that leave through `std::process::exit`
are missed by a trace of the parent alone.

## 5. Shapes, cost and limits

Units per hour are what one VM of the shape costs: the account's total rate in `colab usage`
while that VM ran, minus the rate of the VMs already running (two probe VMs, 0.52/hr). Each
figure is a single reading. Billing is per VM, not per core: an 8-core VM costs the same
0.26/hr idle or with all 8 cores busy. Cores and RAM are what the VM reports (`nproc`,
`free -g`).

| Shape (`new` flags) | Cores | RAM | Units/hr | Units per core-hour | Notes |
|---|---|---|---|---|---|
| standard CPU (none) | 2 | 12 GB | 0.08 | 0.040 | Cheapest per hour, not per build: a cold workspace build takes about 4.6 times as long as on high-mem and costs more units (see "2 cores against 8 cores" below). Small test runs on an already built tree only |
| high-mem CPU (`--high-mem`) | 8 | 50 GB | 0.26 | 0.033 | Best value per core and per GB; the workhorse. AMD EPYC 7B12, 226 GB disk |
| T4 (`--gpu T4`) | 2 | 12 GB | 1.15 | 0.58 | Only worth it for GPU work |
| T4 high-mem (`--gpu T4 --high-mem`) | 8 | 50 GB | 1.57 | 0.20 | |
| L4 (`--gpu L4`) | 12 | 52 GB | 1.94 | 0.16 | Most cores among the GPU shapes per unit |
| TPU v5e-1 (`--tpu V5E1`) | 24 | 47 GB | 3.09 | 0.13 | Most cores per VM among the cheaper TPU/GPU shapes |
| TPU v6e-1 (`--tpu V6E1`) | 44 | 172 GB | 5.09 | 0.12 | AMD EPYC 9B14; took about 6 min to create |
| A100 (`--gpu A100`) | 12 | 83 GB | 6.73 | 0.56 | |
| A100 high-mem | 12 | 167 GB | 8.88 | 0.74 | |
| G4, H100 | — | — | — | — | `Service Unavailable` on every try |

For CPU-only work (builds, tests), the high-mem CPU VM is the cheapest compute by a factor of
4 or more; accelerator VMs are worth renting for their CPUs only when the CPU limit below is
used up and the budget would otherwise go unspent. Only the host CPU of a GPU or TPU VM is
used in that case.

**Account limits, measured** (the number of VMs of a kind that could be live at once on this
Ultra account; Colab does not publish them and says they vary):

| Kind | Limit |
|---|---|
| CPU VMs | 5 high-mem + 5 standard |
| L4 | 3 |
| T4 | 3 |
| T4 high-mem | 3 |
| TPU v5e-1 | 2, then `Service Unavailable` (looked like a capacity shortage, not a limit) |
| A100 | not measured |

GPU and TPU VMs did not count against the CPU limit.

**The limits above did not hold on 2026-11-26.** With 8 VMs live (3 high-mem, 1 standard,
2 L4, 2 TPU v5e-1), every `colab new --high-mem` failed with `Allocation refused
(precondition failed)`: 66 tries over 18 min by one client, and 15 or more by another at the
same time. That is below the 5 high-mem measured earlier. Earlier the same day the same error
cleared after 24 tries at 15 s intervals with fewer VMs live. So the limit may be on the
total number of VMs, or on something else Colab does not report. The message does not say
which limit was reached. Treat it as "full": stop idle VMs (`pool.sh reap 60`), retry every
15 s, and do not count on a fixed number of VMs.

### 2 cores against 8 cores

A Rust workspace build (`cargo test --workspace --no-run`, about 133 crates) measured on
standard (2-core) and high-mem (8-core) CPU VMs:

| VM | Build directory | Crates compiled | Time | Units |
|---|---|---|---|---|
| high-mem hm2, newly created, 2026-11-26 | empty | 133 | 852 s (14.2 min) | about 0.062 |
| standard std1, 2026-11-25 | empty | 115 | 61 min 43 s | about 0.082 |
| standard std2, 2026-11-26 | empty | 134 | 65 min 48 s | about 0.088 |
| standard, 2026-11-22 (smaller workspace) | empty | — | 3764 s (62.7 min) | about 0.084 |

On a cold build the 2-core VM is about 4.6 times slower (3948 s against 852 s for the same
crate count). That is more than the factor of 4 in cores. The probable reasons, not measured
apart: the high-mem VM also runs 8 rustc front-end threads (`-Zunstable-options
--jobs-frontend=8`, about 14% faster cold), and the 2-core VM has a quarter of the memory. Units are rate × time, so the
standard VM, at 0.08/hr against 0.26/hr, still costs about 40% more per cold build and keeps
the client waiting an hour. The units are computed from the rates, not read from `colab usage`.

- Use a high-mem VM, or an L4 (12 cores) or TPU v5e-1 (24 cores) VM for its host CPU, for any
  job that compiles much. Use a standard VM only for jobs whose build is already in place.
  A standard VM that is still compiling after an hour is normal, not a hang.
- The measured front-end speedup was at 8 threads. A 12- or 24-core VM gets 8 too, since
  more threads were never measured.
- A build on a VM that has built the workspace before is much faster. A TPU v5e-1 that had
  built earlier versions rebuilt all 134 crates in 3 min 58 s. Such numbers are not cold
  builds; compare cold with cold.

### What a kept build directory saves when the source changes

Measured on hm2 (high-mem, 8 cores, 8 front-end threads), 2026-11-26. One build directory was
kept throughout, and a git clone of the repository was moved from commit to commit with
`git checkout`. Each build was `cargo test --workspace --no-run`.

| Step | Files changed | Crates compiled | Time |
|---|---|---|---|
| build an old commit, empty build directory | — | 133 | 852 s |
| move to the head, 2 days later | 291 | 25 | 856 s |
| the same commit again, no change | 0 | 0 | 0.6 s |
| back one commit | 6, one of them in the bottom crate of the graph | 20 | 289 s |
| forward again | 6 | 20 | 293 s |

Running tests after a build recompiled nothing (0 crates, each run 0.5 to 13 s).

- Cargo recompiles only the crates whose source changed and the crates that depend on them.
  This holds across `git checkout`: cargo compares each local source file's modification time
  with the time of its last build, and a checkout rewrites only the files that differ. (So
  copy a tree onto the VM with a tool that leaves unchanged files untouched, such as
  `rsync -c`, never by unpacking it afresh, which gives every file a new time.)
- A large step gains nothing. The 25 crates of the 291-file step were the deepest ones, so
  the build took as long as a cold one.
- A small step is about 3 times faster than a cold build, even when it changes the crate at
  the bottom of the graph.
- So keep a VM's build directory, and send it jobs whose source is close to what it built
  last: the same client's next commit, or the current main branch plus one change. Section 6
  does this.

**Clone with `--depth 1`.** History is needed only to move between commits, as this
measurement did (a full clone took 34 s on the VM, with an 89 MB `.git`; a `--depth 1` clone of
the same repository took 4.4 s on a local machine, with a 19 MB `.git`). For one commit:

```bash
git clone --depth 1 <url> /content/repo
git -C /content/repo fetch --depth 1 origin <sha> && git -C /content/repo checkout -q --detach <sha>
```

GitHub served an older commit, not a branch tip, to `fetch --depth 1` by its full sha (1.8 s
for the fetch and checkout, 2026-11-26).

**Timings:** a CPU VM is created in 3 to 5 s; a TPU v6e-1 took about 6 min. A cold VM
needed about 1 min to install Rust (the pinned nightly of `rust-toolchain.toml` installed
without trouble) and 6 to 10.5 min to build the workspace with tests (8 cores; a release build
of the binaries alone took 5 min 45 s); later jobs on the same VM built incrementally. A 16 MB
source tarball uploaded without trouble. A binary built on the local machine (Fedora, glibc
2.43) ran on a VM (Ubuntu, glibc 2.39); that held for the one binary tried and is not
guaranteed, since a binary linked against a newer glibc can need symbols an older one lacks.

**Idle VMs are not stopped quickly.** A VM left idle stayed alive about 45 min, billing the
whole time. The Colab FAQ says idle VMs are deleted "after a period of time" and that a VM
lives "at most 12 hours" (24 h of continuous execution on Pro+). Stop VMs yourself.

### Budget planning

The Ultra plan used here deposits 5000 units a month (the FAQ: Google AI plan units are
deposited monthly, from the date the plan started). Units that are not spent are lost when
they expire, so plan the rate:

    target rate (units/hr) = units left / hours until the units expire

Example: 4999 units with 502 h left is about 9.95 units/hr. The CPU limit alone (5 high-mem
+ 5 standard) burns only 1.7 units/hr, so spending the whole balance on CPU work needs GPU or
TPU VMs for their CPUs too, for example 5 high-mem + 5 standard + 3 L4 + 1 or 2 v5e-1
(about 10.6 to 13.7 units/hr). A paid account whose balance runs out falls back to the free
tier's rules (FAQ).

## 6. A reusable VM pool

### The problem it solves

Three costs pull against each other when several clients (scripts, or several AI agents
working at once) send jobs to Colab:

1. **Setup is slow, so VMs should be reused.** A new VM has no toolchain and no build
   cache. Installing Rust took about 1 min and the first build of the workspace 6 to 10.5
   min; a later job on the same VM rebuilt only what changed. A VM per job pays the cold
   cost every time.
2. **Idle VMs cost units, so they should be stopped quickly.** Colab itself left an idle VM
   running (and billing) for about 45 min.
3. **Two clients must never use the same VM at once.** Both would write `/content/job`, and
   each would read the other's results. And the account limit (5 high-mem CPU VMs here)
   means clients cannot simply each create their own.

The pool answers all three: a VM is created once, handed from job to job with its
toolchain and build cache intact, and given to one client at a time. A VM of a costly shape
is stopped after one minute with no job. A VM of a kept shape (by default the 8-core high-mem
CPU VM, at 0.26 units/hr) is never stopped by the pool. It stays alive and is rebuilt at the
main branch while idle, so every job sent to it compiles only what its own change touches
(section 5 has the measurements).

### How it works

The pool is one text file on the local machine, `pool.tsv`, with one line per live VM:

    <name> <shape> <busy|idle> <owner> <epoch seconds of last change>
    hm1 hm busy agent-a 1795339365
    hm2 hm idle - 1795340176

Every change to the file is made while holding an exclusive `flock` on `pool.lock`, so two
clients that call `acquire` at the same moment cannot both get `hm2`.

A VM moves through these states:

    (none) --acquire, no idle VM of the shape--> colab new --> busy (owner = caller)
    idle   --acquire------------------------------------------> busy (owner = caller)
    busy   --release: kill the job, delete /content/job------> idle (time stamped, owner kept)
    idle   --reap: idle longer than the limit, shape not kept--> colab stop --> (none)
    idle   --claim by prewarm.sh------------------------------> busy (owner = prewarm)
    busy (prewarm) --acquire, no idle VM: stop the prewarm job--> busy (owner = caller)

- **`acquire <shape> <owner>`** prints the name of a VM that the caller now holds alone.
  It takes an idle VM of that shape if there is one, and otherwise creates one named
  `<shape><k>` with the lowest free `k`. Exit 3 means `colab new` failed (account limit
  reached, or `Service Unavailable`); the caller then tries another shape, or waits 15 s and
  tries again; it never falls back to running the work locally.
  Exit 4 means the owner already holds a busy VM: each client holds one VM at a time and runs
  its next job on it, so that a client taking a second VM never keeps the others waiting.
  Owners matching the extended regex `COLAB_POOL_MULTI` (for example a test runner that spreads
  its pieces over several VMs) are exempt.
- **`release <name>`** is called by the client when its job is finished, also when the job
  failed. On the VM it kills the job's process group (the job wrote its process group id to
  `/content/job/pgid`) and deletes `/content/job`. It does **not** delete the toolchain
  (`~/.cargo`, `~/.rustup`) or the build cache (`/content/target`); those are what make the
  next job fast. Then it marks the VM idle with the current time.
- **`reap <seconds>`** stops every VM that has been idle longer than `<seconds>` and removes
  its line. One background loop runs it every 15 s (`reaper.sh` below). A busy VM is never
  reaped.
- **`list`** prints the file.

So a VM of a shape that is not kept is created by the first `acquire` of its shape, reused by
every later `acquire` that comes within a minute of a `release`, and stopped one minute after
its last release. A VM of a kept shape lives until Colab ends it (at most 12 hours):

- **`acquire` prefers the VM the owner used last.** Its build directory holds that owner's
  tree, so the owner's next commit rebuilds only what that commit changed.
- **`keepalive [seconds]`** runs a trivial cell on each idle kept VM that nothing has reached
  for that long. Colab ended VMs after about 45 min with no cell. That keepalive cells prevent
  this is inferred, not measured.
- **`prewarm.sh`** (below) rebuilds an idle kept VM at the main branch after 5 min idle. Then
  a job from any branch close to main rebuilds only its own change. A prewarm build is
  stopped at once when a client needs the VM.

The cost is 0.26 units/hr for each kept high-mem VM, whether busy or idle: 6.2 units a day
each.

### Limits of this design

- **The pool file can disagree with the server.** A VM can be deleted by Colab (idle
  timeout, maximum lifetime, or a crash) while the file still lists it, or a VM made by
  hand can be missing from the file. `acquire`, `release` and `reap` therefore read the VM
  names in `colab --auth=adc sessions` (names only; the list also holds each VM's endpoint)
  and delete an idle line whose VM is not there, and `release` deletes its own line in that
  case instead of marking it idle. Before this, a VM that left the list was marked idle at
  its release and handed to the next client, whose job failed 3 times at the clearing
  (2026-11-22). When the list cannot be read, nothing is deleted. Every `exec` to a deleted
  VM prints `[colab] Session '<name>' not found.` and exits 1 at once; `job.sh` below ends the
  job with exit 5 on it. A VM made by hand that is missing from the file is still yours to
  find: compare with `colab --auth=adc sessions` from time to time.
- **A client that dies while holding a VM leaves it busy forever,** so the reaper never
  stops it. Look for busy lines whose epoch is hours old and whose owner is gone, release
  them, and let the reaper stop them.
- **`acquire` holds the lock while `colab new` runs.** That is 3 to 5 s for a CPU VM but
  about 6 min for a TPU v6e-1, during which every other client waits. Keep slow shapes out
  of the pool, or create them before adding them to the file by hand.
- **Never `colab stop` a pooled VM by hand,** and never `colab new` outside the pool: the
  file then counts wrongly against the account limit.

To measure a shape's limit, acquire VMs of that shape until `new` fails, and read
`colab usage` with each running alone to get its rate. Stop all of them afterward.

### When the pool is too small: a local core semaphore beside it

When Colab refuses more VMs, clients wait in `acquire` for a long time (30+ minutes with 3 VMs
and 12 clients, 2026-11-27). A fixed set of local cores, given out as a bounded semaphore, takes
the overflow. `local-job.sh <label> <worktree> <out dir> <tests file>` reads the same tests file
as `job.sh` and writes the same output layout, so a client can send one job to either.

- **State:** `holders.tsv` (label, pid, CPUs, epoch) and `queue.tsv` (label, pid, start epoch),
  both changed under one flock. A line whose pid is dead is dropped at the next look, so a killed
  client never keeps its cores.
- **Grants per line, not per job:** before each line of the tests file the client queues, ordered
  by job start time, and at the head takes max(1, N / (holders + waiters)) cores once that many
  are free; it gives them back when the line ends. So the shares follow the demand at every line
  boundary. A running line is never stopped, so long jobs should be split into lines.
- **Each line runs pinned** (`taskset -c <cpus>`), with `CARGO_BUILD_JOBS` and the test threads
  set to the share, and the share exported for the line's own use.
- **Differences from a VM job:** the job reads the worktree in place, so the worktree must not
  change until it ends; the build directory must be seeded first; tool paths print relative to
  the worktree; and tools the VM image has (valgrind, perf) may be missing. Never compare an
  instruction count taken locally with one taken on a VM.

### `pool.sh`

```bash
#!/bin/bash
# A pool of Colab VMs shared by several local clients. State: $D/pool.tsv, one line per VM:
#   <name> <shape> <busy|idle> <owner> <epoch of last change>
# Every change holds an flock on pool.lock, so two agents never get the same VM.
# Usage:
#   pool.sh acquire <shape> <owner>   prints a VM name; exit 3 when no VM of that shape can be had
#                                     exit 4 when <owner> already holds a busy VM (one VM per client;
#                                     owners matching $COLAB_POOL_MULTI, an extended regex, may hold several)
#   pool.sh release <name> [<owner>]  kills the job, deletes /content/job on the VM, marks it idle;
#                                     with <owner>, only when <owner> still holds it (prewarm.sh)
#   pool.sh reap [seconds]            stops every VM idle longer than <seconds> (default 60), except
#                                     the kept shapes (see kept)
#   pool.sh claim <name> <owner>      marks the idle VM <name> busy for <owner> (prewarm.sh); exit 3
#                                     when it is not idle
#   pool.sh keepalive [seconds]       runs a trivial cell on each idle kept VM not reached for
#                                     <seconds> (default 600), so Colab does not end it as idle
#   pool.sh list
# acquire, release and reap first delete each VM gone from the Colab session list (see live).
# Kept VMs: a VM of a kept shape (POOL_KEEP, an extended regex, default ^hm$; empty keeps none) is
# never stopped by reap, so its build directory stays and the next job compiles only what changed.
# release keeps the last owner in column 4, and acquire gives an owner the idle VM it used last
# before any other. An idle kept VM can be rebuilt at the main branch by prewarm.sh (owner
# "prewarm"); acquire takes a VM from a prewarm job when no VM of the shape is idle (stop_prewarm).
# Every value is checked before use; an empty or malformed one stops the script with exit 2.
# An empty VM name would make the sed patterns below match every line, and an empty owner would shift the file's columns.
set -u -o pipefail
die() { echo "pool.sh: $*" >&2; exit 2; }
# colab output goes through scrub: no lines with an @, and each URL replaced, since a URL can carry
# a runtime-proxy token in its query.
scrub() { command grep -a -v '@' | sed -E 's#[a-z]+://[^[:space:]]*#<url removed>#g'; }
[ -n "${HOME-}" ] || die "HOME is empty"
D=${COLAB_POOL_DIR:-$HOME/.colab-pool}
[[ $D == /?* ]] || die "COLAB_POOL_DIR must be an absolute path, got '$D'"
mkdir -p "$D" || die "cannot create $D"
P=$D/pool.tsv
K=$D/pool.lock
touch "$P" || die "cannot create $P"
flag() { case $1 in hm) echo --high-mem;; std) echo;; l4) echo --gpu L4;; v5e) echo --tpu V5E1;; *) return 1;; esac; }
name_ok() { [[ ${1-} =~ ^(hm|std|l4|v5e)[0-9]+$ ]]; }
now() { local x; x=$(date +%s) && [[ $x =~ ^[0-9]+$ ]] || die "date +%s gave '${x-}'"; echo "$x"; }
KEEP_RE=${POOL_KEEP-^hm$}
# An empty regex matches every string, so an empty POOL_KEEP is tested first.
kept() { [ -n "$KEEP_RE" ] && [[ ${1-} =~ $KEEP_RE ]]; }
# stop_prewarm <vm>: stops the local prewarm job on <vm>, so that acquire can hand the VM on.
# prewarm-run.sh writes its pid, which is its session id (it runs under setsid), to
# $D/prewarm/<vm>.sid. The whole session is stopped, job.sh with it: a job.sh left polling would
# read the next job's /content/job/pgid as its own and kill that job at its limit. By session, not
# by process group: GNU timeout moves each colab call into a process group of its own. A session
# is stopped only when one of its processes is this VM's prewarm-run.sh or job.sh, since a session
# id can be reused once its processes end. Returns 0 when no prewarm process is left.
stop_prewarm() {
  local v=$1 f=$D/prewarm/$1.sid sid p hit=0 i
  [ -s "$f" ] || return 0
  sid=$(cat "$f")
  [[ $sid =~ ^[0-9]+$ ]] && [ "$sid" -gt 1 ] || return 0
  for p in $(pgrep -s "$sid"); do
    tr '\0' ' ' < "/proc/$p/cmdline" 2>/dev/null | command grep -a -q -E "(prewarm-run|job)\.sh $v " && hit=1
  done
  [ $hit = 1 ] || return 0
  pkill -TERM -s "$sid"
  for i in 1 2 3 4 5; do pgrep -s "$sid" > /dev/null || return 0; sleep 1; done
  pkill -KILL -s "$sid"; sleep 1
  ! pgrep -s "$sid" > /dev/null
}
# A VM can drop out of the Colab session list while pool.tsv still holds it. live prints the
# names in `colab sessions`, one a line, and nothing else of that list (it holds the endpoints);
# it fails when the list could not be read, and then no line is deleted for want of a name. An
# empty list reads "No active sessions found".
live() {
  local t
  t=$(colab --auth=adc sessions 2>&1) || return 1
  command grep -a -q -E '^\[[a-z0-9]+\] |No active sessions found' <<<"$t" || return 1
  command grep -a -o -E '^\[[a-z0-9]+\] ' <<<"$t" | tr -d '[] ' | command grep -a -E '^(hm|std|l4|v5e)[0-9]+$'
  return 0
}
# prune <names>: deletes each idle line whose VM is not in <names> (the output of live). A busy line
# is left to its owner's release, which deletes it the same way.
prune() {
  local n s st o t
  while read -r n s st o t; do
    name_ok "$n" && [ "$st" = idle ] || continue
    command grep -a -q -x "$n" <<<"$1" && continue
    sed -i "/^$n /d" "$P"
    echo "pool.sh: $n is gone from the Colab session list; deleted from $P" >&2
  done < <(cat "$P")
}
reap() {
  local lim=$1 now n s st o t names
  [[ $lim =~ ^[0-9]+$ ]] || die "reap: seconds must be a number, got '$lim'"
  now=$(now) || exit 2
  if names=$(live); then prune "$names"; else echo "pool.sh: cannot read the Colab session list; no VM deleted for want of it" >&2; fi
  while read -r n s st o t; do
    if ! name_ok "$n" || ! [[ ${t-} =~ ^[0-9]+$ ]]; then
      echo "pool.sh: skipping malformed line: $n $s $st $o $t" >&2; continue
    fi
    if [ "$st" = idle ] && ! kept "$s" && [ $((now - t)) -gt "$lim" ]; then
      colab --auth=adc stop -s "$n" > /dev/null 2>&1
      sed -i "/^$n /d" "$P"
      echo "stopped $n (idle $((now - t)) s)" >&2
    fi
  done < <(cat "$P")
}
case ${1-} in
acquire)
  shape=${2-}; owner=${3-}
  [ -n "$shape" ] || die "acquire: empty shape"
  [[ $owner =~ ^[A-Za-z0-9._-]+$ ]] || die "acquire: owner must be a non-empty label without spaces, got '$owner'"
  f=$(flag "$shape") || die "unknown shape '$shape'"
  exec 9>"$K"; flock 9 || die "cannot lock $K"
  # One VM per client: a client that takes a second VM while it holds one keeps others waiting.
  # A test runner that spreads pieces over several VMs is named in COLAB_POOL_MULTI (an extended
  # regex matched against the whole owner; empty, the default, exempts no one).
  multi=${COLAB_POOL_MULTI-}
  if [ -z "$multi" ] || ! [[ $owner =~ ^($multi)$ ]]; then
    h=$(awk -v o="$owner" '$3 == "busy" && $4 == o {print $1; exit}' "$P")
    [ -z "$h" ] || { echo "pool.sh: $owner already holds $h; run the next job on it, or release it first" >&2; exit 4; }
  fi
  ts=$(now) || exit 2
  # No VM gone from the session list is handed out. When the list cannot be read, an idle VM is
  # still handed out; job.sh then ends with exit 5 if it is gone.
  if names=$(live); then prune "$names"; else echo "pool.sh: cannot read the Colab session list" >&2; fi
  # First the idle VM this owner used last (its tree and build are there), then any idle VM, then a
  # VM running a prewarm build (stopped first, see stop_prewarm), then a new VM.
  n=$(awk -v s="$shape" -v o="$owner" '$2 == s && $3 == "idle" && $4 == o {print $1; exit}' "$P")
  [ -n "$n" ] || n=$(awk -v s="$shape" '$2 == s && $3 == "idle" {print $1; exit}' "$P")
  if [ -z "$n" ] && [ "$owner" != prewarm ]; then
    for v in $(awk -v s="$shape" '$2 == s && $3 == "busy" && $4 == "prewarm" {print $1}' "$P"); do
      name_ok "$v" || continue
      if stop_prewarm "$v"; then n=$v; echo "pool.sh: stopped the prewarm build on $v for $owner" >&2; break; fi
    done
  fi
  if [ -n "$n" ]; then
    name_ok "$n" || die "malformed VM name '$n' in $P"
    sed -i "s/^$n .*/$n $shape busy $owner $ts/" "$P"; echo "$n"; exit 0
  fi
  k=1; while command grep -a -q "^$shape$k " "$P"; do k=$((k + 1)); done; n=$shape$k
  name_ok "$n" || die "made a malformed VM name '$n'"
  # $f is empty for std on purpose (no flag), so it is left unquoted.
  # The log goes through scrub (see job.sh): no @ lines, no URLs, which can carry a token.
  if colab --auth=adc new -s "$n" $f 2>&1 | scrub > "$D/new_$n.log"; [ "${PIPESTATUS[0]}" = 0 ]; then
    echo "$n $shape busy $owner $ts" >> "$P"; echo "$n"; exit 0
  fi
  tail -1 "$D/new_$n.log" >&2; exit 3;;
release)
  n=${2-}; want=${3-}
  name_ok "$n" || die "release: bad VM name '$n'"
  [ -z "$want" ] || [[ $want =~ ^[A-Za-z0-9._-]+$ ]] || die "release: bad owner '$want'"
  command grep -a -q "^$n [a-z0-9]* busy " "$P" || die "release: $n is not a busy VM in $P"
  # held <owner>: 0 when <owner> holds <name>. Checked before the clearing and again under the lock.
  held() { awk -v n="$n" -v o="$1" '$1 == n && $3 == "busy" && $4 == o {f = 1} END {exit !f}' "$P"; }
  if [ -n "$want" ] && ! held "$want"; then echo "pool.sh: $n is no longer held by $want; left as it is" >&2; exit 0; fi
  # A VM gone from the session list is deleted, not marked idle.
  if names=$(live) && ! command grep -a -q -x "$n" <<<"$names"; then
    exec 9>"$K"; flock 9 || die "cannot lock $K"
    sed -i "/^$n /d" "$P"
    echo "pool.sh: $n is gone from the Colab session list; deleted from $P" >&2; exit 0
  fi
  # Stop the job first (run.sh's pid, in /content/job/pgid, is the session of every process of the
  # job: job.sh starts it with setsid), so that a job still running cannot write into the next
  # job's directory. By session, not by process group: run.sh runs each line under `timeout`,
  # which moves the line into a process group of its own, so `kill -- -<pgid>` left it running
  # (2026-11-22). The id must be a number above 1: `pkill -s 0` is this cell's own session.
  printf '%%%%bash\np=$(cat /content/job/pgid 2>/dev/null)\nif [[ $p =~ ^[0-9]+$ ]] && [ "$p" -gt 1 ]; then pkill -s $p; fi\nsleep 2\nrm -rf --one-file-system --preserve-root=all /content/job\n' |
    timeout 120 colab --auth=adc exec -s "$n" --timeout 100 > /dev/null 2>&1
  exec 9>"$K"; flock 9 || die "cannot lock $K"
  if [ -n "$want" ] && ! held "$want"; then echo "pool.sh: $n is no longer held by $want; left as it is" >&2; exit 0; fi
  ts=$(now) || exit 2
  # The owner stays in column 4 for acquire's choice of the VM an owner used last.
  sed -i "s/^$n \([a-z0-9]*\) busy \([^ ]*\) .*/$n \1 idle \2 $ts/" "$P";;
reap) exec 9>"$K"; flock 9 || die "cannot lock $K"; reap "${2:-60}";;
claim)
  n=${2-}; owner=${3-}
  name_ok "$n" || die "claim: bad VM name '$n'"
  [[ $owner =~ ^[A-Za-z0-9._-]+$ ]] || die "claim: bad owner '$owner'"
  exec 9>"$K"; flock 9 || die "cannot lock $K"
  command grep -a -q "^$n [a-z0-9]* idle " "$P" || exit 3
  ts=$(now) || exit 2
  sed -i "s/^$n \([a-z0-9]*\) idle .*/$n \1 busy $owner $ts/" "$P";;
keepalive)
  # Colab ends a VM that runs no cell for about 45 min. Its status reads IDLE even while a
  # detached job runs, so the timer is taken to count from the last exec call (inferred, not
  # measured: every VM a job.sh polls has stayed up). Busy VMs are reached by their job's polls.
  # The time each VM was last reached is in alive.tsv (<name> <epoch>), apart from pool.tsv's
  # column 5, which prewarm.sh reads as the time the VM went idle.
  lim=${2:-600}
  [[ $lim =~ ^[0-9]+$ ]] || die "keepalive: seconds must be a number, got '$lim'"
  A=$D/alive.tsv; touch "$A" || die "cannot create $A"
  now=$(now) || exit 2
  while read -r n s st o t; do
    name_ok "$n" && [ "$st" = idle ] && kept "$s" && [[ ${t-} =~ ^[0-9]+$ ]] || continue
    last=$(awk -v n="$n" '$1 == n {print $2}' "$A" | tail -1)
    [[ $last =~ ^[0-9]+$ ]] && [ "$last" -gt "$t" ] || last=$t
    [ $((now - last)) -gt "$lim" ] || continue
    if printf '%%%%bash\ntrue\n' | timeout -k 5 60 colab --auth=adc exec -s "$n" --timeout 45 > /dev/null 2>&1; then
      { flock 9; sed -i "/^$n /d" "$A"; echo "$n $(now)" >> "$A"; } 9>"$K"
    else
      echo "pool.sh: keepalive cell on $n failed" >&2
    fi
  done < <(cat "$P");;
list) cat "$P";;
*) sed -n '2,23p' "$0"; exit 2;;
esac
```

### `reaper.sh`

Start it once, in the background, and leave it running while the pool is in use:

```bash
#!/bin/bash
# Every 15 s: stops pooled VMs idle for more than 60 s (except the kept shapes), runs a trivial
# cell on each idle kept VM not reached for 10 min, and starts a prewarm build if one is due.
set -u
d=$(dirname "$0") && [ -x "$d/pool.sh" ] || { echo "reaper.sh: no pool.sh next to $0" >&2; exit 2; }
while :; do
  "$d/pool.sh" reap 60
  "$d/pool.sh" keepalive 600
  [ -x "$d/prewarm.sh" ] && "$d/prewarm.sh"
  sleep 15
done
```

### `prewarm.sh` and `prewarm-run.sh`: keep idle kept VMs built at the main branch

`prewarm.sh` runs from `reaper.sh`. It starts at most one prewarm job a round and returns at
once. It takes an idle VM of shape `hm` that went idle at least `PREWARM_IDLE` seconds ago
(default 300, so a client's next job within 5 min still finds its own tree) and is not already
built at the current main. It moves a clean worktree to that commit and runs `job.sh` there
with one small test line, so that job.sh's prebuild builds every test binary. The source is
the worktree `PREWARM_TREE` of the repository `PREWARM_REPO`, moved with
`git switch --detach`, which refuses to overwrite a local change. Nothing else may write to
that worktree. The ref read is `origin/main`, as last fetched; fetch it from time to time if
pushes come from elsewhere.

```bash
#!/bin/bash
set -u -o pipefail
D=${COLAB_POOL_DIR:-$HOME/.colab-pool}; PW=$D/prewarm
R=${PREWARM_REPO:?absolute path of the repository}; T=${PREWARM_TREE:?absolute path of the prewarm worktree}
J=$(dirname "$0")
mkdir -p "$PW" || exit 2
# One launch at a time: the worktree is moved only after the last launch packed its source.
exec 8>"$PW/lock"; flock -n 8 || exit 0
sha=$(git -C "$R" rev-parse --verify -q origin/main) && [[ $sha =~ ^[0-9a-f]{40}$ ]] || exit 0
now=$(date +%s); [[ $now =~ ^[0-9]+$ ]] || exit 2
touch "$PW/built.tsv"
vm=$(awk -v now="$now" -v sha="$sha" -v idle="${PREWARM_IDLE:-300}" -v f="$PW/built.tsv" '
  BEGIN { while ((getline l < f) > 0) { split(l, a, " "); b[a[1]] = a[2] } }
  $2 == "hm" && $3 == "idle" && now - $5 >= idle && !($4 == "prewarm" && b[$1] == sha) { print $1; exit }' "$D/pool.tsv")
[ -n "$vm" ] || exit 0
[[ $vm =~ ^hm[0-9]+$ ]] || exit 2
"$J/pool.sh" claim "$vm" prewarm || exit 0
if [ -d "$T" ]; then git -C "$T" switch -q --detach "$sha"; else git -C "$R" worktree add -q --detach "$T" "$sha"; fi ||
  { echo "prewarm: cannot move $T to $sha" >&2; "$J/pool.sh" release "$vm" prewarm; exit 2; }
[ "$(git -C "$T" rev-parse HEAD)" = "$sha" ] && [ -z "$(git -C "$T" status --porcelain)" ] ||
  { echo "prewarm: $T is not a clean checkout of $sha" >&2; "$J/pool.sh" release "$vm" prewarm; exit 2; }
# Any one quick test line of the workspace; job.sh builds the whole workspace before it.
printf '%s\n' "${PREWARM_TEST:?one cargo test line, e.g. -p mycrate --test smoke}" > "$PW/tests.txt"
out=$PW/out-$vm-$now
: > "$PW/empty"
setsid bash "$J/prewarm-run.sh" "$vm" "$sha" "$out" "$T" > "$PW/run-$vm.log" 2>&1 < "$PW/empty" &
# Hold the lock until job.sh packed the worktree: it writes extra.tgz (empty when there is no
# extra checkout) only after src.tgz is complete.
for i in $(seq 1 120); do
  [ -s "$out/extra.tgz" ] && break
  kill -0 $! 2> /dev/null || break
  sleep 1
done
echo "prewarm: $vm building $sha (out $out)"
```

`prewarm-run.sh` writes its own pid to `$D/prewarm/<vm>.sid` (under `setsid` the pid is the
session id, which `stop_prewarm` stops), runs the job, records the result so that a broken
main is not rebuilt every round, and releases the VM only if the prewarm still holds it:

```bash
#!/bin/bash
set -u -o pipefail
D=${COLAB_POOL_DIR:-$HOME/.colab-pool}; PW=$D/prewarm; J=$(dirname "$0")
vm=${1-}; sha=${2-}; out=${3-}; T=${4-}
[[ $vm =~ ^hm[0-9]+$ ]] && [[ $sha =~ ^[0-9a-f]{40}$ ]] && [[ $out == "$PW"/out-* ]] && [ -d "$T" ] ||
  { echo "prewarm-run: bad arguments" >&2; exit 2; }
echo $$ > "$PW/$vm.sid"
"$J/job.sh" "$vm" "$T" "$out" "$PW/tests.txt"; rc=$?
{ flock 7; sed -i "/^$vm /d" "$PW/built.tsv"; echo "$vm $sha $(date +%s) $rc" >> "$PW/built.tsv"; } 7>"$PW/built.lock"
echo "prewarm-run: $vm at $sha, job.sh exit $rc"
"$J/pool.sh" release "$vm" prewarm
```

`nohup ./reaper.sh >> "$HOME/.colab-pool/reaper.log" 2>&1 &`. When all work is done, stop
the reaper, then check `colab --auth=adc sessions` and stop anything still listed.

### `job.sh`: one test job on a pooled VM

This is the complete job runner used here, generalized. It sends a git checkout (tracked
and untracked files, not ignored ones) and an optional second directory to the VM, builds,
runs each line of a tests file with the out-of-memory back-off of section 4, and brings back
the logs. A tests-file line that starts with `!` is run as a shell command instead (clippy, a
benchmark with its own flags, a script), once, with no back-off. It must exit with its build
command's code (`cargo ...; r=$?; echo ...; exit $r`): the dylib-retry runs only on a nonzero
exit, so a line ending in `echo` hides the failure and is never retried. Adapt the build and test
commands in `run.sh` to your project.

Every argument is checked before it is used, and an empty or malformed one stops the script
with exit 2 before anything is created or sent. This matters because an empty variable in a
path or a pattern does not fail: `sed -i "/^$n /d"` with an empty `n` deletes every line of
the pool file, `kill -- -$p` with `p` equal to 1 kills every process on the VM, and
`pkill -s $p` with `p` equal to 0 kills the caller's own session.
`pool.sh` and `job.sh` therefore start with `set -u -o pipefail`, check each name against a
pattern, require absolute paths, and accept a process-group id only when it is a number
greater than 1.

```bash
#!/bin/bash
# Runs cargo builds and tests of one git checkout on a pooled Colab VM and brings the logs back.
# Usage: job.sh <vm> <checkout> <out dir> <tests file> [<extra dir>]
#   <vm>          a name from `pool.sh acquire`
#   <checkout>    absolute path of the checkout to test (tracked and untracked, non-ignored files;
#                 on the VM git tracks only the tracked ones, see tracked.lst below)
#   <out dir>     absolute path of a new local directory for the logs
#   <tests file>  one line per run: the arguments after `cargo test`, e.g. `-p mycrate --test foo`,
#                 or a shell command after a !, e.g. `!cargo clippy --all-targets -- -D warnings`
#   <extra dir>   optional absolute path of a second git checkout the tests read, given to them
#                 as EXTRA_DIR
# Each cargo test line runs with --test-threads=8, and after an out-of-memory kill again at 4, 2,
# then 1. When every line is a ! command, the build of every test binary before the lines is left
# out: such commands build what they run themselves, often with other features or flags, so the
# prebuild would be a second, unused build of the workspace. The VM keeps the toolchain and the
# build directory /content/target between jobs, and the last job's files as /content/src and
# /content/extra (see run.sh), so the next job on the same VM builds incrementally; the job's own
# files are under /content/job, which `pool.sh release` deletes. Output: <out dir>/summary.tsv (line, exit code, threads, seconds,
# max RSS kB, [oom-retry or dylib-retry]) and <out dir>/out/<n>.log per line. Exit 0 when every line passed;
# 4 the clearing or the launch failed; 5 the job died or its VM left the Colab session list; 6 an
# upload failed; 7 the job passed JOB_LIMIT or stalled for JOB_STALL seconds (see the wait below).
# Every argument and computed value is checked before use; an empty or malformed one stops the
# script with exit 2.
set -u -o pipefail
die() { echo "job.sh: $*" >&2; exit 2; }
[ $# -ge 4 ] && [ $# -le 5 ] || die "$# arguments; usage: job.sh <vm> <checkout> <out dir> <tests file> [<extra dir>]"
vm=${1-}; wt=${2-}; out=${3-}; tests=${4-}; extra=${5-}
[[ $vm =~ ^(hm|std|l4|v5e)[0-9]+$ ]] || die "bad VM name '$vm'; usage: job.sh <vm> <checkout> <out dir> <tests file> [<extra dir>]"
[[ $wt == /?* ]] && [ -d "$wt" ] || die "checkout '$wt' is not an absolute path of a directory"
[[ $out == /?* ]] && [ "$out" != / ] || die "out dir '$out' is not an absolute path below /"
[ -s "$tests" ] || die "tests file '$tests' is missing or empty"
[ -z "$extra" ] || { [[ $extra == /?* ]] && [ -d "$extra" ]; } || die "extra dir '$extra' is not an absolute path of a directory"
# The source sent is the files `git ls-files -co --exclude-standard` lists in the checkout. In a
# directory git does not know (a snapshot made with `git archive`, which has no .git) it lists
# none, and tar of an empty list still writes a non-empty archive: the VM got an empty source,
# every test line failed with rustup's "no default is configured", and the job reported nothing
# wrong (FARM-job-empty-file-list, 2026-11-23).
[ -n "$(git -C "$wt" ls-files -co --exclude-standard 2>/dev/null | head -c 1)" ] ||
  die "checkout '$wt' has no files git lists (not a git checkout, or none tracked or untracked): the VM would get an empty source"
# 0 when every non-empty line is a ! command: no prebuild (see the head).
prebuild=1
command grep -a -q -v -E '^(!|$)' "$tests" || prebuild=0
mkdir -p "$out" || die "cannot create $out"
# Everything colab prints goes through scrub: it drops lines with an @ (account names) and replaces
# each URL by <url removed>, since a failed upload prints a URL whose query carries a runtime-proxy
# token.
scrub() { command grep -a -v '@' | sed -E 's#[a-z]+://[^[:space:]]*#<url removed>#g'; }
# One exec call; the cell comes on stdin; $2 is the local timeout in seconds (default 120). -k 5
# sends KILL 5 s after the TERM, so a colab that ignores the TERM cannot outlive its limit.
c() { local s=${2:-120}; timeout -k 5 "$s" colab --auth=adc exec -s "$vm" --timeout "$((s - 10))" 2>&1 | scrub; }
# A VM that dropped out of the Colab session list never answers again: colab prints
# "[colab] Session '<name>' not found." (colab_cli/common.py get_session) or "... appears to be
# lost (404/401). Cleaning up." (commands/execution.py exec_command) and exits 1 at once. The wait
# below took that for a lost connection and polled every 20 s with no end, once for about 10 hours
# (2026-11-22). gone <text> is true for either message; the clearing, the launch, the progress
# copy and the poll all end the job with exit 5 on it.
gone() { command grep -a -q -E "Session '[^']*' (not found|appears to be lost)" <<<"$1"; }
vanished() { echo "job died on $vm: job.sh: the VM is gone from the Colab session list ($1)"; exit 5; }
pushd "$wt" > /dev/null && { git ls-files -z -co --exclude-standard | tar --null -T - -czf "$out/src.tgz" ; } ; popd > /dev/null
# The untracked files are sent for the build and for ! lines that read them, but the VM's git
# tracks only what the checkout's git tracks: run.sh adds the paths of tracked.lst, not all files.
# With `git add -A` there, a test that reads `git ls-files` also read the untracked scratch files
# on the VM (a citation check failed on an untracked tar of an old tree, 2026-11-26) and not on
# the local machine. Tracked files deleted from the checkout are left out of the list, since
# `git add` of a path that is not there fails.
pushd "$wt" > /dev/null && { git ls-files -z | while IFS= read -r -d '' f; do
  [ -e "$f" ] || [ -L "$f" ] && printf '%s\0' "$f"; done > "$out/tracked.lst" ; } ; popd > /dev/null
[ -f "$out/tracked.lst" ] || die "writing the list of tracked files failed"
if [ -n "$extra" ]; then
  pushd "$extra" > /dev/null && { git ls-files -z | tar --null -T - -czf "$out/extra.tgz" ; } ; popd > /dev/null
  [ -s "$out/extra.tgz" ] || die "packing $extra failed"
else
  tar czf "$out/extra.tgz" -T /dev/null || die "cannot write $out/extra.tgz"
fi
[ -s "$out/src.tgz" ] || die "packing $wt failed"
cp "$tests" "$out/tests.txt" || die "cannot copy $tests"
cat > "$out/run.sh" <<'R'
#!/bin/bash
set -u
J=/content/job; TOKEN=${1-}; PREBUILD=${2-}
[[ $TOKEN =~ ^[0-9]+$ ]] || { echo "run.sh: bad token '$TOKEN'" >&2; exit 2; }
[[ $PREBUILD =~ ^[01]$ ]] || { echo "run.sh: bad prebuild flag '$PREBUILD'" >&2; exit 2; }
[ -n "${HOME-}" ] || { echo "run.sh: HOME is empty" >&2; exit 2; }
# The marker is made only when each step succeeded: made whatever their results, one failed
# rustup download left a VM on which every later job died at `source`, with no word of the cause.
# A failed step is named on stderr, which run.log keeps and the wait below prints on exit 5.
if [ ! -e /content/setup-done ] || [ ! -f "$HOME/.cargo/env" ]; then
  step=
  curl -sSf https://sh.rustup.rs > $J/rustup-init.sh 2> $J/setup.log || step="the download of rustup"
  [ -n "$step" ] || sh $J/rustup-init.sh -y --profile minimal --default-toolchain none >> $J/setup.log 2>&1 || step=rustup
  [ -n "$step" ] || command -v /usr/bin/time > /dev/null || apt-get install -y -qq time >> $J/setup.log 2>&1 ||
    { apt-get update -qq && apt-get install -y -qq time; } >> $J/setup.log 2>&1 || step="apt-get install time"
  if [ -n "$step" ]; then
    echo "run.sh: setup failed: $step (setup.log)" >&2
    tail -5 $J/setup.log | command grep -a -v '@' >&2
    exit 2
  fi
  touch /content/setup-done
fi
source "$HOME/.cargo/env" || { echo "run.sh: setup failed: no $HOME/.cargo/env" >&2; exit 2; }
mkdir -p $J/src $J/extra $J/out /content/src /content/extra
tar xzf $J/src.tgz -C $J/src; tar xzf $J/extra.tgz -C $J/extra
pushd $J/src > /dev/null || exit 1
# The tarball holds no .git, and tests that call git (git ls-files) need a repository. Only the
# checkout's tracked files are added (see tracked.lst); -f adds a tracked file .gitignore matches.
git init -q && git add -f --pathspec-from-file=$J/tracked.lst --pathspec-file-nul && git -c user.name=job -c user.email=job commit -q --allow-empty -m snapshot
popd > /dev/null
# Never build in the unpacked copy: tar restores each file's original mtime, so with a target
# dir shared by several checkouts' jobs, a file edited before another checkout's last build
# looks older than that build, and cargo silently reuses the other checkout's code. rsync -c
# without -t rewrites only files whose contents changed, with the current time as their mtime:
# changed files rebuild, unchanged ones stay incremental.
command -v rsync > /dev/null || apt-get install -y -qq rsync >> $J/setup.log 2>&1
# Native tests link with cc, ld and ld.lld; a VM lacks ld.lld (and a v5e VM lacked ld). 18 s.
command -v cc > /dev/null && command -v ld > /dev/null && command -v ld.lld > /dev/null ||
  { apt-get install -y -qq gcc binutils lld || { apt-get update -qq && apt-get install -y -qq gcc binutils lld; }; } >> $J/setup.log 2>&1
# Tools for tests that need podman, qemu, gdb for other architectures and ping (see section 3 for
# what failed without each). The marker's number goes up with each change of this list.
if [ ! -e /content/setup-full-3-done ]; then
  { { apt-get install -y -qq clang qemu-user gdb-multiarch podman uidmap passt slirp4netns fuse-overlayfs zstd xz-utils iputils-ping ||
      { apt-get update -qq && apt-get install -y -qq clang qemu-user gdb-multiarch podman uidmap passt slirp4netns fuse-overlayfs zstd xz-utils iputils-ping; }; } &&
    mkdir -p /dev/net && { [ -e /dev/net/tun ] || mknod -m 666 /dev/net/tun c 10 200; } &&
    { id zp > /dev/null 2>&1 || useradd -m -s /bin/bash zp; } &&
    mkdir -p /home/zp/.config/containers &&
    printf '%s\n' '[containers]' 'default_sysctls = []' 'pidns = "host"' \
      'default_capabilities = ["CHOWN", "DAC_OVERRIDE", "FOWNER", "FSETID", "KILL", "NET_BIND_SERVICE", "SETFCAP", "SETGID", "SETPCAP", "SETUID", "SYS_CHROOT", "NET_RAW"]' \
      '[engine]' 'cgroup_manager = "cgroupfs"' > /home/zp/.config/containers/containers.conf &&
    chown -R -P zp: /home/zp/.config && touch /content/setup-full-3-done; } >> $J/setup.log 2>&1 ||
    echo "run.sh: the setup of the full tests' tools failed (setup.log)" > $J/out/setup-full.log
fi
rsync -rlpc --delete $J/src/ /content/src/ || { echo "run.sh: rsync of the source failed" >&2; exit 2; }
rsync -rlpc --delete $J/extra/ /content/extra/ || { echo "run.sh: rsync of the extra files failed" >&2; exit 2; }
pushd /content/src > /dev/null || exit 1
nc=$(nproc); [[ $nc =~ ^[0-9]+$ ]] || nc=1
# The compiler's front end (nightly --jobs-frontend) runs on one thread per core on a VM of 8 cores
# or more, and on 1 below that: measured 2:14 at 1 thread, 1:56 at 8 on a cold test build, about
# 14% faster (zed-impl, 2026-11-18). A VM runs one job, so the threads do not compete with other
# builds. RUSTFLAGS replaces a project's build.rustflags in .cargo/config.toml: copy any flags the
# project sets there into this line.
fe=1; [ "$nc" -ge 8 ] && fe=8   # 8 is the count measured; a larger VM also gets 8
export RUSTFLAGS="-Zunstable-options --jobs-frontend=$fe"
export CARGO_TARGET_DIR=/content/target CARGO_BUILD_JOBS=$nc EXTRA_DIR=/content/extra
# hm and std Colab VMs have no hardware counter of instructions (l4 and v5e not checked; dmesg: "no PMU driver, software events
# only"; perf_event_open of the instructions counter fails). A checkout that ships a DynamoRIO
# client at tools/icount (as zed does: build.sh, install-dynamorio.sh, zed_icount.c) gets
# DynamoRIO 11.3.0 at /opt/dynamorio and the client built at /content/icount, once per VM and
# again when the client's source changes, and the two variables that name them; its test
# binaries then count instructions in software. Measured 2026-11-22 on an hm VM: 1.12x slower
# than native at 0.9 s of work (plus about 0.08 s of start-up), the same count on every run, and
# ratios within 0.1% of the hardware counter's. valgrind is for tests that run it. Other
# checkouts skip this. Rejected: DynamoRIO's own sample counter (1.73x, two runs gave
# different counts) and valgrind callgrind (21x). The software count is about 133,000
# instructions below the hardware count at both sizes measured (process start and exit,
# inferred), which leaves ratios unchanged. Without the client the meter measures processor
# time: never report such a ratio as an instruction count. A runner that reads the job's out/
# must fail a benchmark run when out/icount.log exists (written below when the client's build
# failed) or a `bench` line says `work=` where a count says `instr=` (an old branch writes no
# icount.log): a ratio of processor time is not an instruction count, even though zed's scaling
# check now holds every meter to the limit itself (10.5, at or over fails; until 2026-11-23
# processor time had twice the limit, 21: FAIL-scaling-check-passes-over-limit), and it takes
# the fastest of 5 runs in 3 tries, so such a run can pass where instructions would fail. zed's tools/farm_run.sh does this under --bench, with a
# line `farm: benchmarks measured processor time, not instructions, on <VM> ...` and exit 1
# (FARM-icount-fallback-silent, 2026-11-23). A branch older than tools/icount
# gets processor time too, and the meter's own tests (scaling_tests.rs) fail on a VM; merge
# main into the branch. The meter reruns a test under DynamoRIO only when the test binary's
# arguments include `--exact` (zed-impl crates/zed-meter-settings/src/lib.rs, the check on
# `--exact` in start_under_dynamorio): a job line that names a benchmark without `--exact`
# measures processor time even on a VM with the client (found 2026-11-22 by a job that did so).
# A job line that names a benchmark also needs `--nocapture`: libtest keeps the output of a
# passing test, so without it the log has no `bench ... instr=` line to read the ratio from
# (b22-checkrest10, 2026-11-23; FARM-bench-nocapture). zed's tools/run_tests.sh --bench passes
# both flags to each benchmark piece.
if [ -e tools/icount/build.sh ]; then
  if ! cmp -s tools/icount/zed_icount.c /content/icount/zed_icount.c; then
    { { apt-get install -y -qq cmake g++ valgrind || { apt-get update -qq && apt-get install -y -qq cmake g++ valgrind; }; } &&
      bash tools/icount/install-dynamorio.sh /opt/dynamorio &&
      bash tools/icount/build.sh /opt/dynamorio /content/icount &&
      cp tools/icount/zed_icount.c /content/icount/zed_icount.c; } >> $J/setup.log 2>&1 ||
      echo "run.sh: building zed_icount failed (setup.log); benchmarks measure processor time" > $J/out/icount.log
  fi
  # Only a client built from this tree's source: after a failed build the stored copy differs
  # (it is written last), so a half-built .so is never used (b22-scale1, 2026-11-26).
  cmp -s tools/icount/zed_icount.c /content/icount/zed_icount.c &&
    export ZED_METER_DRRUN=/opt/dynamorio/bin64/drrun ZED_METER_ICOUNT=/content/icount/libzed_icount.so
fi
# A crate that is also a dylib (crate-type = ["rlib", "dylib"]) keeps one file name whatever its
# features. A build that resolves other features (a `cargo run -p` line, or a checkout whose
# manifests differ from the last one built here) overwrites dylibs that the other build's
# fingerprints still take as fresh, and every later build that links them fails with "cannot
# satisfy dependencies so `std` only shows up once". A step that fails so cleans the build
# directory, by its literal path, and runs once more, marked dylib-retry; the cost is one cold
# build, against a VM that would fail every later build.
# cargo clean refuses a directory without cargo's CACHEDIR.TAG ("missing or invalid
# `CACHEDIR.TAG` file"), which cargo writes only into a target directory it made itself; one made
# first by mkdir has none, and the retry then ran on the same stale build and failed again
# (2026-11-22). The tag, with cargo's own text, is written first.
cleaned=0
mixed() { [ $cleaned = 0 ] && command grep -a -q 'cannot satisfy dependencies so' "$1"; }
reclean() {
  cleaned=1
  [ ! -d /content/target ] || [ -e /content/target/CACHEDIR.TAG ] ||
    printf 'Signature: 8a477f597d28d172789f06886806bc55\n' > /content/target/CACHEDIR.TAG
  cargo clean --target-dir /content/target >> $J/out/clean.log 2>&1
}
# The clean does not cure a ! line that runs two builds of other features itself, or a -p build
# after a workspace build: its retry mixes the dylibs again (FARM-dylib-mix-bang-lines, b22-tests7
# 2026-11-23, l43, fresh target directories: `cargo test -p zedc-presburger --no-run` then
# `cargo test -p zed-tests --test l01_38_omega_test --no-run` failed, rc 101, 52 such errors in
# zedc-preimage; `cargo build -p zed-tools --bin plan --bin sync` after the workspace build in
# /content/target failed the same way; with cargo's hashed dylib names (__CARGO_DEFAULT_LIB_METADATA,
# as wt/zrun sets) the first pair still failed, and with CARGO_RESOLVER_FEATURE_UNIFICATION=workspace
# too). The error comes from std linked statically into each dylib: with -C prefer-dynamic, as wt/zrun
# builds locally, every dylib and binary loads the one libstd, and the same pair passed (l43, fresh
# directory). So such a ! line runs once more with prefer-dynamic in its own build directory,
# /content/target-pd, kept across jobs; the rpaths let a binary there run without cargo setting
# the library path. /content/target keeps static std, so the benchmarks and the prebuild measure
# what they measured before; a plain line after a mixed ! line still gets the clean above.
pdflags="-C prefer-dynamic -C link-arg=-Wl,-rpath,$(rustc --print target-libdir) -C link-arg=-Wl,-rpath,\$ORIGIN -C link-arg=-Wl,-rpath,\$ORIGIN/deps -Zunstable-options --jobs-frontend=$fe"
if [ "$PREBUILD" = 0 ]; then
  echo "build skipped: every line is a ! command" > $J/out/build.log
else
  s=$(date +%s); cargo test --workspace --no-run > $J/out/build.log 2>&1; rc=$?
  if [ $rc -ne 0 ] && mixed $J/out/build.log; then
    echo "build rc=$rc secs=$(( $(date +%s) - s )) dylib-retry" >> $J/out/clean.log; reclean
    s=$(date +%s); cargo test --workspace --no-run > $J/out/build.log 2>&1; rc=$?
  fi
  echo "build rc=$rc secs=$(( $(date +%s) - s ))" >> $J/out/build.log
fi
n=0
while IFS= read -r line; do
  [ -z "$line" ] && continue; n=$((n + 1)); t=8
  # A line that holds its own -- (cargo test X -- --skip Y) gets the thread count after it; a second
  # -- made libtest take "--test-threads=8" as a name filter, so the line ran 0 tests and passed
  # (b22-checkrest5, 2026-11-22).
  sep=--; case " $line " in *" -- "*) sep=;; esac
  # A line starting with ! is a shell command run in the checkout's root (clippy, a benchmark
  # with its own flags, a script), once, with no out-of-memory retry.
  if [ "${line#!}" != "$line" ]; then
    [ -n "${line#!}" ] || { printf '%s\t2\t0\t0\t\n' "$line" >> $J/out/summary.tsv; continue; }
    s=$(date +%s)
    /usr/bin/time -v -o $J/out/$n.time timeout 3600 bash -c "${line#!}" < /dev/null > $J/out/$n.log 2>&1; rc=$?
    secs=$(( $(date +%s) - s )); rss=$(awk '/Maximum resident/ {print $6}' $J/out/$n.time)
    if [ $rc -ne 0 ] && command grep -a -q 'cannot satisfy dependencies so' $J/out/$n.log; then
      printf '%s\t%s\t%s\t%s\t%s\tdylib-retry\n' "$line" $rc 0 $secs "$rss" >> $J/out/summary.tsv
      s=$(date +%s)
      CARGO_TARGET_DIR=/content/target-pd RUSTFLAGS="$pdflags" /usr/bin/time -v -o $J/out/$n.time timeout 3600 bash -c "${line#!}" < /dev/null > $J/out/$n.log 2>&1; rc=$?
      secs=$(( $(date +%s) - s )); rss=$(awk '/Maximum resident/ {print $6}' $J/out/$n.time)
    fi
    printf '%s\t%s\t%s\t%s\t%s\n' "$line" $rc 0 $secs "$rss" >> $J/out/summary.tsv; continue
  fi
  while :; do
    s=$(date +%s)
    /usr/bin/time -v -o $J/out/$n.time timeout 3600 cargo test $line $sep --test-threads=$t < /dev/null > $J/out/$n.log 2>&1; rc=$?
    secs=$(( $(date +%s) - s )); rss=$(awk '/Maximum resident/ {print $6}' $J/out/$n.time)
    # A timeout (exit 124) is not retried: one test binary killed inside a line that used its whole hour would otherwise cost another hour at half the threads (found by b22-checkwalk5).
    if [ $rc -ne 0 ] && [ $rc -ne 124 ] && [ $t -gt 1 ] && command grep -a -q -E 'SIGKILL|signal: 9|memory allocation of .* failed|SIGABRT' $J/out/$n.log; then
      printf '%s\t%s\t%s\t%s\t%s\toom-retry\n' "$line" $rc $t $secs "$rss" >> $J/out/summary.tsv; t=$((t / 2)); continue
    fi
    if [ $rc -ne 0 ] && mixed $J/out/$n.log; then
      printf '%s\t%s\t%s\t%s\t%s\tdylib-retry\n' "$line" $rc $t $secs "$rss" >> $J/out/summary.tsv; reclean; continue
    fi
    printf '%s\t%s\t%s\t%s\t%s\n' "$line" $rc $t $secs "$rss" >> $J/out/summary.tsv; break
  done
done < $J/tests.txt
popd > /dev/null
# setup.log is written outside out/; a copy goes in out.tgz so a failed setup can be read.
[ ! -f $J/setup.log ] || cp $J/setup.log $J/out/setup.log
tar czf $J/out.tgz -C $J out; touch $J/DONE-$TOKEN
R
# A job left on this VM by an earlier job.sh run is cleared first, as pool.sh release does: its
# pgid file would stop the launch below ("[ -s pgid ] ||"), and the wait loop would then see the
# old dead group and report "job died" (seen twice, then reproduced, 2026-11-22). The clearing is
# confirmed by a separate call and tried 3 times: a lost clearing call (another exec call held the
# same kernel) left the old files and gave the same "job died" with no run.
for try in 1 2 3; do
  r=$(printf '%%%%bash\np=$(cat /content/job/pgid 2>/dev/null)\nif [[ $p =~ ^[0-9]+$ ]] && [ "$p" -gt 1 ]; then pkill -s $p; sleep 2; fi\nrm -rf --one-file-system --preserve-root=all /content/job\nmkdir -p /content/job\n' | c)
  gone "$r" && vanished "while clearing the previous job"
  r=$(printf '%%%%bash\n[ -d /content/job ] && [ ! -e /content/job/pgid ] && echo "CLEA""RED"\n' | c 60)
  gone "$r" && vanished "while clearing the previous job"
  command grep -a -q CLEARED <<<"$r" && break
  [ $try = 3 ] && { echo "job.sh: clearing the previous job on $vm failed 3 times" >&2; exit 4; }
  sleep 10
done
# An upload that fails is tried 3 times, 10 s apart, then the job stops with exit 6 before a launch
# without its files. colab exits 1 on an HTTP error (an unhealthy VM can answer every upload with
# HTTP 500); without this check the job went on and failed later as "launch failed".
# PIPESTATUS[0] is the upload's own exit, not grep's or tail's.
for f in src.tgz extra.tgz tracked.lst tests.txt run.sh; do
  for try in 1 2 3; do
    timeout 600 colab --auth=adc upload -s "$vm" "$out/$f" "/content/job/$f" 2>&1 | scrub | tail -1
    [ "${PIPESTATUS[0]}" = 0 ] && break
    [ $try = 3 ] && { echo "job.sh: upload of $f to $vm failed 3 times" >&2; exit 6; }
    sleep 10
  done
done
# The uploaded copies are emptied, not deleted (no rm from a variable path): each job keeps a
# copy of the worktree and the extra dir, and many jobs can fill a per-user /tmp quota. A status
# script can warn before that: `quota --show-mntpoint -w` prints "<fs> <mount point> <blocks>
# <quota> <limit> ..." in KiB (a `*` follows the blocks once past the quota), so warn when the
# blocks of the /tmp line pass 80% of the hard limit, or of the soft one when there is no hard one.
: > "$out/src.tgz"; : > "$out/extra.tgz"
# Each job has its own token, so a DONE file left by an earlier job cannot end this one's wait.
# setsid gives run.sh its own process group, whose id `pool.sh release` kills.
tok=$(date +%s%N)
[[ $tok =~ ^[0-9]+$ ]] || die "bad token '$tok'"
# A dropped connection can lose the launch command (2026-11-22, t3: "Connection was lost"), so the
# launch is confirmed by a separate call that sees the pgid file, and retried up to 3 times.
for try in 1 2 3; do
  r=$(printf '%%%%bash\n[ -s /content/job/pgid ] || { setsid nohup bash /content/job/run.sh %s %s > /content/job/run.log 2>&1 & echo $! > /content/job/pgid; }\n' "$tok" "$prebuild" | c)
  gone "$r" && vanished "at the launch"
  r=$(printf '%%%%bash\n[ -s /content/job/pgid ] && echo "LAUNCH""ED"\n' | c 60)
  gone "$r" && vanished "at the launch"
  command grep -a -q LAUNCHED <<<"$r" && break
  [ $try = 3 ] && { echo "launch failed on $vm"; exit 4; }
  sleep 10
done
# Wait for DONE. A failed poll (connection lost) is retried; a job process that is gone with no
# DONE file ends the wait with exit 5, so a dead job never makes this loop run forever. An empty
# or non-numeric pgid counts as dead: `kill -0` with no argument would only print an error.
# Once 180 s or more have passed since the last copy, the job's progress is copied to
# $out/progress.txt, overwritten each time: the time, out/summary.tsv so far, and the last 20 lines
# of out/build.log and of the newest line log, so a long job can be followed without waiting for its
# end. Counted with $SECONDS, not in polls: an exec call can return only when its timeout ends it
# (60 s on l43, 2026-11-22), so a poll can take 80 s and a count of polls says little about time.
# Two limits end a job that would otherwise be waited on for ever, each with exit 7, after the
# job's process group on the VM is killed (one call; the VM may not answer):
# - the job limit, JOB_LIMIT seconds from the launch: by default 3600 for the setup and the build
#   (a cold build of a large workspace took 3764 s on a 2-core std VM and about 6 min on hm)
#   plus 3600 for each line, the limit run.sh puts on one try of a line;
# - the stall limit, JOB_STALL seconds (default 1800) in which no poll got an answer, or neither
#   the bytes under /content/job/out nor the processor time of the job's processes changed. The
#   bytes alone are not enough: a ! line that builds with its output in a log of its own left
#   out/ at 1521 bytes for over 1700 s of a cold build on a 2-core std VM (2026-11-22). A build
#   or a test that runs uses processor time; a process waiting for ever, or a VM that no longer
#   answers, does not, and a process that loops for ever is ended by the job limit. The time is
#   the sum over the processes of the session (`ps -s`), which falls when a process ends, so any
#   change counts. A poll that fails for any other reason than a gone session (a lost
#   connection) is still retried, up to this limit.
# Each copy of the progress also prints one line, so the caller's log of this script grows while
# the job runs. Both limits are read once and checked as numbers; a test with a fake colab (a
# script named colab first on PATH) sets them small.
lines=$(command grep -a -c . "$tests")
[[ $lines =~ ^[0-9]+$ ]] || die "cannot count the lines of $tests"
limit=${JOB_LIMIT:-$((3600 + 3600 * lines))}
stall=${JOB_STALL:-1800}
[[ $limit =~ ^[0-9]+$ ]] && [[ $stall =~ ^[0-9]+$ ]] || die "JOB_LIMIT '$limit' or JOB_STALL '$stall' is not a number of seconds"
# The job is stopped by its session, not its process group: run.sh runs each line under
# `timeout`, which puts itself and the line in a process group of its own, so `kill -- -<pgid>`
# left the line running (2026-11-22), while setsid makes run.sh's pid the session of every
# process of the job. The id must be above 1: `pkill -s 0` is the caller's own session.
stop_job() {
  printf '%%%%bash\np=$(cat /content/job/pgid 2>/dev/null)\nif [[ $p =~ ^[0-9]+$ ]] && [ "$p" -gt 1 ]; then pkill -s $p; fi\n' | c 60 > /dev/null
  echo "job.sh: $1; the job on $vm was stopped"; exit 7
}
started=$SECONDS
copied=$SECONDS
moved=$SECONDS
bytes= activity=
while :; do
  if [ $((SECONDS - copied)) -ge 180 ]; then
    copied=$SECONDS
    printf '%%%%bash\ncd /content/job/out 2>/dev/null || exit 0\ndate -u +%%FT%%TZ\necho "== summary.tsv"; cat summary.tsv 2>/dev/null\necho "== build.log"; tail -20 build.log 2>/dev/null\nl=$(ls -t [0-9]*.log 2>/dev/null | head -1)\n[ -z "$l" ] || { echo "== $l"; tail -20 "$l"; }\n' |
      c 60 > "$out/progress.txt"
    gone "$(cat "$out/progress.txt")" && vanished "at a progress copy"
    echo "job.sh: $(date -u +%FT%TZ) running on $vm for $((SECONDS - started)) s, out/ ${bytes:-?} bytes"
  fi
  # The answer is the state, the byte count of /content/job/out and the processor seconds of the
  # job's session, as "RUNNING 12345 678".
  r=$(printf '%%%%bash\np=$(cat /content/job/pgid 2>/dev/null)\nb=$(du -sb /content/job/out 2>/dev/null | cut -f1)\nt=$(ps -s "$p" -o times= 2>/dev/null | awk "{ s += \\$1 } END { print s + 0 }")\nif [ -e /content/job/DONE-%s ]; then echo "JOB""DONE"; elif [[ $p =~ ^[0-9]+$ ]] && kill -0 "$p" 2>/dev/null; then echo "RUN""NING ${b:-0} ${t:-0}"; elif [ -e /content/job/DONE-%s ]; then echo "JOB""DONE"; else echo "JOB""DEAD"; fi\n' "$tok" "$tok" | c 60)
  gone "$r" && vanished "at a poll"
  case $r in *JOBDONE*) break;; *JOBDEAD*) echo "job died on $vm"; printf '%%%%bash\ntail -20 /content/job/run.log\n' | c; exit 5;; esac
  b=$(command grep -a -o -E 'RUNNING [0-9]+ [0-9]+' <<<"$r")
  if [ -n "$b" ] && [ "${b#RUNNING }" != "$activity" ]; then activity=${b#RUNNING }; bytes=${activity% *}; moved=$SECONDS; fi
  [ $((SECONDS - started)) -lt "$limit" ] || stop_job "the job passed its limit of $limit s (JOB_LIMIT)"
  [ $((SECONDS - moved)) -lt "$stall" ] || stop_job "no answer from the VM, or no new output and no processor time used, for $stall s (JOB_STALL)"
  sleep 20
done
timeout 600 colab --auth=adc download -s "$vm" /content/job/out.tgz "$out/out.tgz" 2>&1 | scrub | tail -1
tar xzf "$out/out.tgz" -C "$out" && cp "$out/out/summary.tsv" "$out/summary.tsv" || die "no results in $out/out.tgz"
# out.tgz is emptied once unpacked, like src.tgz and spec.tgz: a 0-byte out.tgz after a job is
# expected, and its contents are in $out/out (1.log, summary.tsv, setup.log). One that could not
# be unpacked stops the job above with "no results".
: > "$out/out.tgz"
cat "$out/summary.tsv"; tail -1 "$out/out/build.log"
awk -F'\t' '$6 == "" && $2 != 0 {bad = 1} END {exit bad}' "$out/summary.tsv"
```

The job runs as root on the VM, so the `rm -rf` in `pool.sh release` is written with a
literal path and `--one-file-system --preserve-root=all`; keep it that way if you edit it.

### One job, start to finish

```bash
O=$HOME/colab-runs/run-1; mkdir -p "$O"
printf '%s\n' '-p mycrate --test big_tests' '-p othercrate' '!cargo clippy --all-targets -- -D warnings' > "$O.tests"
vm=
while [ -z "$vm" ]; do           # exit 3: none free, retry every 15 s; exit 4: you hold one already
  for s in hm l4 v5e std; do
    vm=$(./pool.sh acquire $s me); rc=$?
    [ $rc = 0 ] && break
    vm=; [ $rc = 3 ] || { echo "pool.sh acquire $s: exit $rc (4: reuse or release the VM you hold)"; exit 1; }
  done
  [ -n "$vm" ] || sleep 15
done
[[ $vm =~ ^(hm|std|l4|v5e)[0-9]+$ ]] || { echo "pool.sh printed no VM name"; exit 1; }
./job.sh "$vm" "$PWD" "$O" "$O.tests" > "$O.log" 2>&1; echo "job rc=$?"
./pool.sh release "$vm"          # always, also after a failure
cat "$O/summary.tsv"
```

### Instructions to give AI agents that use the pool

When several AI agents share the pool, each one needs these rules in its instructions (they
are what was given to the agents here):

- Run every build, test, clippy run and benchmark on a VM. Only editing, formatting, git and
  the push scripts run on the local machine. A job already running locally may finish there,
  but stop at once every loop that would start new local runs (a watcher such as main-watch),
  or switch it to the VMs: one left running started a new local benchmark run after the order.
- Cap every test piece at 60 s yourself: `job.sh` runs each line under `timeout 3600`, so a
  plain line (`-p <crate> --test <t>`) has no 60 s cap. Write `!timeout 60 cargo test -p <crate>
  --test <t> <filter>` for each test piece and `!timeout <limit> cargo test ...` for each
  benchmark, with its own limit. A red run (the old code, expected to fail) gets the same
  limits; a timeout there is the failure it should show (a quadratic benchmark on the base
  commit ran 337 s uncapped, 2026-11-22).
- How the build is done depends on the lines. If any line is plain, `job.sh` first runs
  `cargo test --workspace --no-run` with no limit of its own (only `JOB_LIMIT`, by default
  3600 s plus 3600 s a line). If every line starts with `!`, there is no such build, so put
  `!cargo test --no-run -p <crate> --test <t> ...` first; that line has the 3600 s limit of a
  line. A cold build on a 2-core std VM takes about 3800 s and is killed there, so do a cold
  build on hm, l4 or v5e, or add one plain line so the workspace build runs first. The target
  directory is kept between jobs on one VM, so later builds there are incremental. `JOB_STALL`
  need not be raised for a long build: since aed21ad the stall check counts the processor time
  of the job's processes, which a compile uses.
- If no VM can be had (every shape exits 3), wait 15 seconds and try again (the reaper stops a
  VM idle for 60 s, so a longer wait lets a freed VM be stopped instead of reused); do not fall back
  to running the job locally, or the local machine is overloaded again. Exit 4 means you
  already hold a busy VM: reuse it or release it, and never retry on exit 4. Never hide
  acquire's exit code; a script that did retried for ever.
- A test piece that times out in a full run (one piece per core, so heavy pieces share the
  VM) is run once more, alone, on the same VM, and both results are reported. It passes only
  if it finishes within its normal limit alone; one that fails alone is really slow. (A full
  run had 61 of 451 benchmarks hit 20 s at a load of about 10 on 8 cores.)
- A pattern that selects tests for a farm run or a merge check names them with their group
  prefix (`l01_38_presburger::`) and is anchored, `^...$`. An unanchored one also matched
  `short-zed_kernel`, a piece that runs only in a separate sandbox, and the runner stopped.
- Do not start helper agents. Each counts against the agent limit and needs its own VM; an
  agent whose work is too big writes a handoff instead.
- **Benchmarks: run a whole scaling series in one job on one VM.** A Colab VM shares its host
  with other users, so a ratio measured across two VMs, or against a local number, means
  nothing. A ratio that fails is run once more on the same VM before it is analyzed; report
  both runs.
- Get VMs only through `pool.sh acquire`, never `colab new`; return them only through
  `pool.sh release`, never `colab stop`. Release also after a failure. Hold one VM at a time.
- Run `job.sh` in the background and keep working; do not poll it. Do not end the turn while
  a job is still running: an agent that stops then sits idle until someone resumes it.
- Never wait for a job with a loop on `pgrep -f <script name>`: the waiting shell's own command
  line holds that name, so pgrep always matches it and the loop never ends (a waiter hung 12 h
  this way, 2026-11-26). Wait for the background task's completion notice or the `job rc=` line.

### Waiting for an agent's jobs (orchestrator)

An agent that ended its turn with jobs still running can be resumed once they end.
`wait-agent-jobs.sh <label>` exits when no `job.sh` or `farm_run.sh` process names the label
(in its worktree or output path; a job on another worktree is missed, see section 9). Run it in the background; when it reports, send the
agent a message to continue. It is a backup: an agent that started its own background watch on
a job is resumed by that watch when the job ends, and reports so. When the wait ends while the
agent still says a job is running, the job is on a worktree whose name lacks the label.

```bash
#!/bin/bash
# Usage: wait-agent-jobs.sh <agent label>
# Its own command line holds the label but neither job.sh nor farm_run.sh, so it never counts itself.
A="${1:-}"
[ -n "$A" ] || { echo "usage: wait-agent-jobs.sh <label>" >&2; exit 2; }
while :; do
  n=$(ps -eo args | command grep -a -E '(job\.sh|farm_run\.sh)' | command grep -a -c -F "$A")
  [ -n "$n" ] || n=0
  [ "$n" -gt 0 ] || break
  sleep 30
done
echo "$(date -u +%FT%TZ) jobs of $A ended"
```

A label that is a prefix of another (`b22-misc7` and `b22-misc70`) matches both; pick labels
that are not prefixes of each other.
- Read `summary.tsv` (a line's exit is on its final row, field 6 empty; a row ending in
  `oom-retry` or `dylib-retry` is an attempt run again); analyze a failure on the VM like a local one, including failures that
  do not happen locally (memory, thread count, a host path, a missing tool).
- Never put a credential, token or email address on a VM, in a tests file or in a kept log;
  filter `@` lines and URLs (a failed upload prints a URL whose query carries a token) from
  anything `colab` prints; the scripts do this with `scrub`.

## 7. Privacy

- Never put credentials, tokens, API keys or the account's email address on a VM, in an
  uploaded tarball, or in a job script. Build tarballs from `git ls-files`, which leaves out
  untracked secrets and ignored files.
- Some CLI output contains the account's email address. Filter it out of anything logged or
  shown to others: `colab … 2>&1 | grep -v '@'`.
- `~/.config/colab-cli/sessions.json` holds session tokens. Never print, upload or share it.
  Use `colab --config <path>` to give a separate client its own state file.

## 8. From the Colab FAQ

<https://research.google.com/colaboratory/faq.html>, read 2026-11-22. The points that matter
for offloading:

- VMs have a maximum lifetime and are deleted when idle; Colab "prioritizes interactive
  compute." Long unattended runs are better split into jobs that each finish in a few hours.
- Limits on VM counts and accelerators are not published and vary over time.
- Paid units are spent by the hour by the shape chosen; do not pick a GPU unless it is
  needed. Prices per shape are shown at the signup page, not in the FAQ.
- Not allowed: file hosting, proxies, cryptocurrency mining, several accounts to get around
  limits, and containers used to get around the abuse policies. SSH, remote desktops and
  distributed workers are disallowed only on free runtimes or paid runtimes with no balance,
  so batch builds and tests on a paid balance are within the terms.
- Copy data in as archives, not as many small files.

## 9. What failed, and why

| What was tried | Result | Fix |
|---|---|---|
| Running the whole build inside one `exec` call | A call fails after 30 s without output, and a long call is exposed to dropped connections | Detached job + polling (section 4) |
| Launching without confirming | `RuntimeError: Connection was lost` on the launch call; `run.sh` never started and the poller waited forever | Confirm with a second call; retry 3 times; detect a dead job |
| `grep JOBDONE` on the poll output | Matched the echoed source, so the wait ended at once | Split the word in the source: `"JOB""DONE"` |
| A fixed done-file name on a reused VM | A file left by the previous job ended the new job's wait | A token per job in the done-file name |
| Reusing a VM whose last job was abandoned | The old job, if still running, writes into the new job's directory | Kill the process group on release |
| `colab list` | No such command | `colab sessions` |
| `--gpu G4`, `--gpu H100` | `Service Unavailable` | Use L4, T4 or A100 |
| A misspelled `--gpu` value | Silently gives an A100 (the most expensive shape) | Check the shape with `colab sessions` after `new` |
| Reading the cost of a job from `colab usage` | The balance had not moved yet | Rate table × lifetime |
| Leaving a VM idle "to stop by itself" | Still billing after 45 min | The reaper (section 6) |
| 8 test threads on a 50 GB VM | Out-of-memory | Thread back-off. The cause was the scaling helpers holding all 30 inputs while the meter repeated each size 15 times (section 4); fixed by building each input just before its run |
| Editing `job.sh` in place while jobs ran it | bash reads a script as it runs, so a running job can read the new text at its old offset and run half a command | Write `job.sh.new`, test it, then `mv` it over the old one: a running job keeps the old file (`/proc/<pid>/fd/255` shows it as deleted) |
| `cp job.sh.next job.sh` (2026-11-22 16:07) | `cp` onto an existing file writes into the same inode, so it is an edit in place: a job running the old text failed with `syntax error near (` at line 337 and returned no results | Only `mv job.sh.next job.sh`; check afterward that `stat -c %i job.sh` changed |
| `wait-agent-jobs.sh <label>` with jobs on another worktree | An agent ran a job on a second worktree (`b22-presburger3` for agent `b22-presburger4`); its command line did not hold the label, so the wait ended while the job ran | Ask the agent which worktrees its jobs use, and run one wait per worktree name |
| A cold build on std3 (2 cores) under the `job.sh` from before aed21ad (2026-11-22) | The progress log stopped at `Compiling zed-tests` after 3668 s; with no new output for 1800 s the stall check stopped the job before any test ran. A cold workspace build on std takes about 3764 s | The stall check now also counts processor time (aed21ad), so a compile no longer looks stalled; do cold builds on hm, l4 or v5e when one is free (see section 6) |
| All 15 pool VMs held by long-running agents and their merge checks (2026-11-22) | An agent got exit 3 from `pool.sh acquire` on hm, l4 and v5e for about 30 minutes; std was the only shape it could get | Expected at the pool limit: keep retrying every 15 s and take any shape; an agent that holds a VM while waiting for its own results keeps the others waiting, so release a VM once its job's results are read and no next job is ready |
| Jobs from different checkouts sharing one VM's build directory (FARM-jobsh-p-builds-mix-dylibs, 2026-11-22) | A crate built as both rlib and dylib keeps one file name whatever its features. A build from another checkout, with different manifests, overwrote dylibs that the first build's fingerprints still took as fresh, and every later build that linked them failed with "cannot satisfy dependencies so \`std\` only shows up once" (b22-preimage5 j2-j4; b22-presburger4 colab-7 and colab-8 on hm5). The VM failed every later build | When the workspace build or any line fails with that message, `job.sh` runs `cargo clean` on the fixed build directory (`/content/target`) and runs the step once more. At most one clean per job, so a step that still fails is reported failed, not retried for ever; the failed try stays in `summary.tsv` tagged `dylib-retry`. Cost: one cold build (about 6 min on hm). merge_check and main-watch run through `job.sh` and get the same repair. Tested with a fake cargo: corrupt build, corrupt single line, error that persists |
| A `!` line that builds other features itself: two `-p` builds, or `cargo run --bin plan`/`--bin sync` after the workspace build (FARM-dylib-mix-bang-lines, FARM-job-prefer-dynamic, 2026-11-23) | The clean above does not cure it: the line's second build mixes the dylibs again even in a fresh directory, since std is linked statically into each dylib (l43: `cargo test -p zedc-presburger --no-run` then the omega test binary, rc 101; `cargo build -p zed-tools --bin plan --bin sync` after the workspace build, rc 101). Hashed dylib names (`__CARGO_DEFAULT_LIB_METADATA`) and feature unification do not cure it | A `!` line that fails with that message runs once more with `RUSTFLAGS="-C prefer-dynamic"` plus rpaths to the sysroot lib dir, `$ORIGIN` and `$ORIGIN/deps`, in its own build directory `/content/target-pd` (b22-tests7 job 6 on l43: plan, `sync --check` and the presburger/omega pair each failed, were retried and passed, rc 0). `/content/target` keeps static std, so benchmarks measure what they did. A line that pipes into `echo` hides the rc and is never retried: end the line with the command whose rc counts |
| A test opening an unnamed file with `O_TMPFILE` (zed-rt `which_mappings`, 2026-11-22) | The VM's temporary directory is on overlayfs, whose `open(O_TMPFILE)` fails with EOPNOTSUPP; the child process panicked at its first assertion and the parent reported only "exit 101" | On EOPNOTSUPP, map a `memfd_create` file instead (zed-impl 007be491d; 13 of 13 limits tests pass on l41). Correct for a data-limit test because `may_expand_vm` (mm/mmap.c) applies `RLIMIT_DATA` by the mapping's flags, not by the file behind it, checked against the kernel source. Make a parent test print the child's standard error, so a failure on a VM shows its cause (section 3) |
| A second `job.sh` run on a VM still held (2026-11-22) | The launch runs only if `/content/job/pgid` does not exist (`[ -s pgid ] ||`, so a retried launch cannot start the run twice). After the first job that file still held the finished job's group, so the second job never started; the wait loop saw a dead group and reported "job died". Reproduced on std5: first job rc 0, second rc 5 | Each job first clears `/content/job` as `pool.sh release` does (kill the old session, delete the directory) and confirms `pgid` is gone before the launch (section 2, step 2; `job.sh` "CLEARED" check) |
| `timeout 3600` on each line of `job.sh` taken as the test limit | Two test lines ran with no 60 s cap, and a red benchmark ran 337 s | Put `timeout 60` (or the benchmark's own limit) in each `!` line (section 6) |
| `pgrep -f '<pattern>'` to find a job | Matched the shell running the `pgrep` command itself, whose command line contains the pattern | Record the job's pid when it starts, or look a pid up with `ps -o pid,args -p <pid>`; or wait for an output file the job writes when it ends; or use the bracket form `pgrep -f '[b]in plan'`, which the loop's own text does not match. Such a loop never ends: seven were found 13 to 22 hours old (2026-11-22), so list wait loops older than 10 minutes and stop them by pid |
| Native tests that link with `ld.lld` (build_binary, l01_36b_link_identity, 2026-11-22) | A std VM has `cc` and `ld` but no `ld.lld`; a v5e VM had no `ld` either, so a merge check failed both tests | `job.sh` installs `gcc binutils lld` when any of the three is missing (section 3; 18 s, Ubuntu LLD 18.1.3) |
| A tool run in the dev profile in the same job as tests (`plan --fix`, 2026-11-22) | The test build's target directory holds test-profile artifacts; a `cargo run` in the dev profile there mixed the two, and the job deleted the 19.5 GiB target and rebuilt it from nothing | Run such a tool with `cargo run --profile test …`, the profile the tests built, so it reuses their artifacts |
| Profiling a benchmark with `perf` in a job (2026-11-23) | There is no `perf` package for the VM's kernel 6.6, and `job.sh` sets `ZED_METER_DRRUN`, so the scaling harness reruns itself under DynamoRIO: `perf` then sees 82% "[JIT]" and libdynamorio, never the program's functions | In a profiling job, `unset ZED_METER_DRRUN ZED_METER_ICOUNT` before the run, so the program runs natively; measure the ratio in a separate run with the counter on |
| `perf` after `apt-get install linux-tools-generic` (b22-tests9, 2026-11-24) | The install puts the wrapper `/usr/bin/perf` on PATH, which looks for the running kernel's tools and fails ("perf not found for kernel 6.6.159+", rc 2), so a script that takes `command -v perf` works on the first job and fails on every later one | Call the generic binary directly: `P=$(ls /usr/lib/linux-tools/*/perf \| sort \| sed -n '$p')`, never `command -v perf` |
| `vm.overcommit_memory = 1` on the VMs (2026-11-22) | A build asked whether a huge allocation would fit; locally (mode 0) the kernel refuses such a request, on the VM every request is granted, so the build ran until the VM killed it at about 52 GB, with no message | Refuse any request larger than `MemTotal + SwapTotal` before asking the allocator. A program that relies on an allocation failing locally is killed silently on a VM |
| A farm share whose VM setup failed (2026-11-22) | It reported "pieces: 0, failed: 0", which reads as a pass, so broken code could land | Fail the merge check whenever a share ran fewer pieces than it was sent |
| All 4 parts of a merge check's own tests on std4 (2026-11-22) | Failed once with empty output after 284 s; passed on the next run on std5. The logs were not kept, so the cause is unknown | Open. Keep the logs of any such failure |
| The local /tmp quota (51.5 GB per user) full, 2026-11-22 | Every shell command failed before it ran (EDQUOT), `df` and deletions included; two agents and the orchestrator stopped mid-task. Most of the space (34 GB) was job folders in one session's scratchpad, each keeping its tarballs (about 49 MB a job) | The user frees space from their own terminal. Empty each tarball after its upload (section 6) and keep scratch build copies off /tmp |
| `run_tests.sh` on a VM | Every piece starts under `systemd-run --user`, and a VM has no user systemd bus | On a VM the farm runs each piece with a 112 GiB memory limit of its own instead |
| A change to `farm_run.sh` while main-watch runs | main-watch copies the runner when it starts, so it kept the old one | Restart main-watch after a runner change. `farm_run.sh` reruns a part on another VM when `job.sh` exits with anything other than 0 or 1 |
| A farm share longer than the hour of its `!` line (MAIN-WATCH-sighup, 2026-11-24) | `job.sh` stops each `!` line at 3600 s (`timeout 3600`, exit 124). main-watch's run of every benchmark, about 1,100 pieces a share on 3 VMs, each counted under DynamoRIO, took longer: every run from 2026-11-23 16:48Z on ended with each share's line at 124, its `1.log` holding only the share line (the VM side printed the pieces' lines only at its end), and `farm_run.sh` exited 129, which main-watch read as a crashed runner. It was not a SIGHUP: 129 is also farm_run.sh's exit for a lost share | `farm_run.sh` now writes a share's lines to the job's `out/` as each piece ends (`farm-r1`, `farm-r2`; `job.sh` returns `out/` after a timeout too), and a share stopped at the hour runs on in a new job that leaves out the pieces already settled (a benchmark that failed there runs again alone); a job that settles nothing still loses the share (129). Pieces are matched across jobs by their position in the share's list, which is the same on every VM, not by name: a build hash in a name may differ between VMs, and one share may hold two binaries of one library that differ only in that hash (matching by the name without the hash failed them as one piece reported twice, 2026-11-24). Restart main-watch after it lands (row above). Test: tools/test_farm_run.sh cases hour-resume, hour-resume-bench and hour-no-progress; the first two exit 129 on the runner before |
| Reading a line's exit from row k of `summary.tsv` (FARM-retry-rows-read-as-lines and FARM-share-exit1-pieces-pass, 2026-11-23) | A `!` line run again after `cannot satisfy dependencies so` has a `dylib-retry` row with the first attempt's rc 1 before its final row; `farm_run.sh` read row 1, so a share whose pieces all passed exited 1 and a merge check reported tests/full FAILED, and `--commands` lost the job (the row count disagreed) | Take a line's exit only from the rows whose field 6 is empty, line k's from the k-th of them, as `job.sh`'s own last check does (`$6 == ""`); `farm_run.sh`'s `final_rows` does this |
| A step that needs git history in a job (FAIL-merge-check-doc-skipped-on-vm, 2026-11-23) | The job's copy is a tar of the tracked and untracked files with no `.git`, so `git merge-base`, `git diff` and `git ls-files` fail there: `tools/check_changed_docs.sh` in merge_check's steps printed "The doc check of changed packages is skipped" and checked nothing | Find what needs the history on this machine and send it with the job: `farm_run.sh --commands` sends `check_changed_docs.sh --changes` with each command, and the check reads it from the file `ZED_DOC_CHANGES` names |
| The doc check in each share of a farm run (MERGECHECK-doc-check-skipped-without-origin, 2026-11-24) | The share path sent no changes, and `check_changed_docs.sh` exited 0 with "skipped" when it had no `origin/main`, so every share's `run_tests.sh` passed a doc check it never ran | With no `origin/main` the check now fails ("DOC CHECK FAILED") unless `ZED_DOC_CHANGES` gives the changes or `ZED_DOC_CHECKED` names the run that did the check; `farm_run.sh` gives share 1 the changes and names share 1 in the others, and `merge_check.sh` sets `ZED_DOC_CHECKED` for its shares, since it checks before the VMs |
| `merge_check.sh` running the doc check on this machine (TOOL-doc-check-on-vm, 2026-11-26) | Before its VM shares it ran `check_changed_docs.sh`, which runs `cargo metadata`, `cargo tree` and `cargo doc` of every changed package twice: a local build, against the rule that builds run only on VMs | The changes are found here with git alone (`check_changed_docs.sh --changes` exits before any cargo call), and the check is a second command of the steps' `farm_run.sh --commands` job, which sends them as `ZED_DOC_CHANGES`; the shares get `ZED_DOC_CHECKED` naming that job. A share checks docs only in its `--only` run of short-addresses pieces: `run_tests.sh --list` exits before the check and `--pieces` leaves it to the caller, so "share 1 does it" in the row above holds only when share 1 has such a piece (FARM-share-doc-check-only-with-short) |
| `valgrind --tool=callgrind` on a metered benchmark (RADIX-read-radix-vs-decimal, 2026-11-26) | The meter's allocator reserves 64 TiB of address space at its first count (zed-alloc-count meter.rs, `ready`), which valgrind refuses: the bench panicked at once, "reserving 70368748371968 bytes ... Invalid argument (os error 22)", and callgrind counted 0 | Profile a metered benchmark with the meter's own counts (DynamoRIO, `ZED_METER_DRRUN`), or callgrind a plain test or program that calls the same function without the meter |
| A job sent from a snapshot made with `git archive` (FARM-job-empty-file-list, 2026-11-23) | The snapshot has no `.git`, so `git ls-files` lists nothing, the packed source was empty, every test line failed with rustup's "no default is configured", and the job reported nothing wrong | `job.sh` stops with exit 2 when git lists no file in the checkout; send jobs from a git checkout or worktree |
| A merge check in a fresh worktree | It did nothing: the worktree had no pre-built dependencies | Run `seed-build.sh` first |
| VM results from before the stale-build fix (11:15Z, 2026-11-22) | Jobs that built in the unpacked copy could reuse another checkout's code (section 4) | Rerun any VM result from before that time that a conclusion rests on |
| An empty variable in `sed -i "/^$n /d"` | Would delete every line of the pool file | Check every value against a pattern first; `set -u` (section 6) |
| `kill -- -$(cat pgid)` | A pgid file holding 1 would kill every process on the VM; and it misses a command a `timeout` inside the job moved into its own process group | Kill the session, `pkill -s`, and accept only a number greater than 1 (`pkill -s 0` is the caller's session) |
| `--tpu v5e1` | Not in `colab new --help`, which lists `V5E1` and `V6E1` | Use the spelling the help lists |
| `colab new --high-mem` with 8 VMs of mixed shapes live (2026-11-26) | `Allocation refused (precondition failed)` for 18 min, though only 3 high-mem VMs were live and 5 had been measured as the limit | Treat the message as "full, for some limit": reap idle VMs, retry every 15 s, keep one retry loop per client (section 5) |
| `pool.sh release hm5; pool.sh reap 0; pool.sh acquire hm x` to get a blank VM (2026-11-26) | `reap` stops VMs idle *longer* than the limit, and hm5 had been idle 0 s, so it was not stopped; `acquire` then handed back the same VM, with 23 h of build directories and a toolchain on it | To get a blank VM: release, wait 2 s or more, `reap 1`, check the "stopped" line, then acquire. Confirm with `uptime; ls /content` on the new VM (a new VM shows only `sample_data`) |
| A VM a client released, thought by its client to be still running a job (2026-11-26) | An agent waited on a job that had never started: its launcher was still retrying `acquire` against the full account. `pool.sh list` showed no VM for that owner | When a client reports waiting on a job, check `pool.sh list` and the job's out directory before believing it |
| Starting a detached job with `... < /dev/null &` from a local Claude Code session with a hook that blocks `/dev` paths | The hook refused the whole command, so nothing ran | Close stdin with `0<&-` instead: `setsid nohup bash job.sh > log 2>&1 0<&- &` |
| `colab exec -s N 'ls /content/job; ps -eo pid,args'` (the command as an argument, as with `ssh host cmd`) | Refused before anything runs: `Got unexpected extra argument(s) (...)` and the usage text, exit 2; a loop over VMs prints only usage text, which can be misread as "the VMs are down" (2026-11-23) | Pipe the cell on stdin: `printf '%%%%bash\nls /content/job\n' \| colab --auth=adc exec -s N --timeout 90`. Every literal `%` in the body is doubled too (`date +%%H`, `--time-style=+%%H:%%M`), since the body is `printf`'s format string; or write the cell to a file and pass `-f` |
