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
| `colab exec -s N [-f file.py] [--timeout S] [--env K=V]` | Runs Python on the VM's kernel; code from `-f` or stdin |
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
6. **Clean up per job, not per VM**: kill the job's process group (`kill -- -$(cat pgid)`),
   then delete `/content/job`. Keep the toolchain and build cache (for example
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
allocator does keep a finished test's freed memory. Not yet shown: that this explains 40 GB.
With one test thread the tests run one after another and should reuse the same freed memory,
so a peak 13 times the largest test points also to a leak, an unbounded cache, or many
allocator arenas from threads the tests start. If a test binary's peak is far above its
largest test, record the resident memory after each test in one process before buying a
bigger VM: steady growth means a leak or a cache, a step per new thread means arenas.

## 5. Shapes, cost and limits

Units per hour are what one VM of the shape costs, read from `colab usage` with only that VM
running. Cores and RAM are what the VM reports (`nproc`, `free -g`).

| Shape (`new` flags) | Cores | RAM | Units/hr | Units per core-hour | Notes |
|---|---|---|---|---|---|
| standard CPU (none) | 2 | 12 GB | 0.08 | 0.040 | Cheapest per core; small test runs only: a cold build of a large Rust workspace on 2 cores ran 43 min with no result |
| high-mem CPU (`--high-mem`) | 8 | 50 GB | 0.26 | 0.033 | Best value per core and per GB; the workhorse |
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

**Timings:** a CPU VM is created in 3 to 5 s; a TPU v6e-1 took about 6 min. A cold VM
needed about 1 min to install Rust and 6 to 10.5 min to build the workspace (8 cores);
later jobs on the same VM built incrementally.

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
toolchain and build cache intact, given to one client at a time, and stopped after one
minute with no job.

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
    busy   --release: kill the job, delete /content/job------> idle (time stamped)
    idle   --reap: idle longer than the limit----------------> colab stop --> (none)

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

So a VM's life is: created by the first `acquire` of its shape; reused by every later
`acquire` that comes within a minute of a `release`; stopped one minute after its last
release. To keep the build cache warm, queue work so that each released VM is acquired
again within that minute.

### Limits of this design

- **The pool file can disagree with the server.** A VM can be deleted by Colab (idle
  timeout, maximum lifetime, or a crash) while the file still lists it, or a VM made by
  hand can be missing from the file. Compare with `colab --auth=adc sessions` from time to
  time and fix the file. A job sent to a deleted VM fails at its first `exec`.
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
#   pool.sh release <name>            kills the job, deletes /content/job on the VM, marks it idle
#   pool.sh reap [seconds]            stops every VM idle longer than <seconds> (default 60)
#   pool.sh list
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
reap() {
  local lim=$1 now n s st o t
  [[ $lim =~ ^[0-9]+$ ]] || die "reap: seconds must be a number, got '$lim'"
  now=$(now) || exit 2
  while read -r n s st o t; do
    if ! name_ok "$n" || ! [[ ${t-} =~ ^[0-9]+$ ]]; then
      echo "pool.sh: skipping malformed line: $n $s $st $o $t" >&2; continue
    fi
    if [ "$st" = idle ] && [ $((now - t)) -gt "$lim" ]; then
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
  n=$(awk -v s="$shape" '$2 == s && $3 == "idle" {print $1; exit}' "$P")
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
  n=${2-}
  name_ok "$n" || die "release: bad VM name '$n'"
  command grep -a -q "^$n [a-z0-9]* busy " "$P" || die "release: $n is not a busy VM in $P"
  # Stop the job's process group first (run.sh writes it to /content/job/pgid), so that a job
  # still running cannot write into the next job's directory. The id must be a number above 1:
  # `kill -- -1` would kill every process on the VM.
  printf '%%%%bash\np=$(cat /content/job/pgid 2>/dev/null)\nif [[ $p =~ ^[0-9]+$ ]] && [ "$p" -gt 1 ]; then kill -- -$p; fi\nsleep 2\nrm -rf --one-file-system --preserve-root=all /content/job\n' |
    timeout 120 colab --auth=adc exec -s "$n" --timeout 100 > /dev/null 2>&1
  exec 9>"$K"; flock 9 || die "cannot lock $K"
  ts=$(now) || exit 2
  sed -i "s/^$n \([a-z0-9]*\) busy .*/$n \1 idle - $ts/" "$P";;
reap) exec 9>"$K"; flock 9 || die "cannot lock $K"; reap "${2:-60}";;
list) cat "$P";;
*) sed -n '2,9p' "$0"; exit 2;;
esac
```

### `reaper.sh`

Start it once, in the background, and leave it running while the pool is in use:

```bash
#!/bin/bash
# Stops pooled VMs idle for more than 60 s, checking every 15 s.
set -u
d=$(dirname "$0") && [ -x "$d/pool.sh" ] || { echo "reaper.sh: no pool.sh next to $0" >&2; exit 2; }
while :; do "$d/pool.sh" reap 60; sleep 15; done
```

`nohup ./reaper.sh >> "$HOME/.colab-pool/reaper.log" 2>&1 &`. When all work is done, stop
the reaper, then check `colab --auth=adc sessions` and stop anything still listed.

### `job.sh`: one test job on a pooled VM

This is the complete job runner used here, generalized. It sends a git checkout (tracked
and untracked files, not ignored ones) and an optional second directory to the VM, builds,
runs each line of a tests file with the out-of-memory back-off of section 4, and brings back
the logs. A tests-file line that starts with `!` is run as a shell command instead (clippy, a
benchmark with its own flags, a script), once, with no back-off. Adapt the build and test
commands in `run.sh` to your project.

Every argument is checked before it is used, and an empty or malformed one stops the script
with exit 2 before anything is created or sent. This matters because an empty variable in a
path or a pattern does not fail: `sed -i "/^$n /d"` with an empty `n` deletes every line of
the pool file, and `kill -- -$p` with `p` equal to 1 kills every process on the VM.
`pool.sh` and `job.sh` therefore start with `set -u -o pipefail`, check each name against a
pattern, require absolute paths, and accept a process-group id only when it is a number
greater than 1.

```bash
#!/bin/bash
# Runs cargo builds and tests of one git checkout on a pooled Colab VM and brings the logs back.
# Usage: job.sh <vm> <checkout> <out dir> <tests file> [<extra dir>]
#   <vm>          a name from `pool.sh acquire`
#   <checkout>    absolute path of the checkout to test (tracked and untracked, non-ignored files)
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
# max RSS kB, [oom-retry or dylib-retry]) and <out dir>/out/<n>.log per line. Exit 0 when every line passed.
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
# 0 when every non-empty line is a ! command: no prebuild (see the head).
prebuild=1
command grep -a -q -v -E '^(!|$)' "$tests" || prebuild=0
mkdir -p "$out" || die "cannot create $out"
# Everything colab prints goes through scrub: it drops lines with an @ (account names) and replaces
# each URL by <url removed>, since a failed upload prints a URL whose query carries a runtime-proxy
# token.
scrub() { command grep -a -v '@' | sed -E 's#[a-z]+://[^[:space:]]*#<url removed>#g'; }
# One exec call; the cell comes on stdin; $2 is the local timeout in seconds (default 120).
c() { local s=${2:-120}; timeout "$s" colab --auth=adc exec -s "$vm" --timeout "$((s - 10))" 2>&1 | scrub; }
pushd "$wt" > /dev/null && { git ls-files -z -co --exclude-standard | tar --null -T - -czf "$out/src.tgz" ; } ; popd > /dev/null
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
if [ ! -e /content/setup-done ]; then
  curl -sSf https://sh.rustup.rs | sh -s -- -y --profile minimal --default-toolchain none > $J/setup.log 2>&1
  apt-get install -y -qq time >> $J/setup.log 2>&1
  touch /content/setup-done
fi
source "$HOME/.cargo/env" || { echo "run.sh: no $HOME/.cargo/env" >&2; exit 2; }
mkdir -p $J/src $J/extra $J/out /content/src /content/extra
tar xzf $J/src.tgz -C $J/src; tar xzf $J/extra.tgz -C $J/extra
pushd $J/src > /dev/null || exit 1
git init -q && git add -A && git -c user.name=job -c user.email=job commit -qm snapshot
popd > /dev/null
# Never build in the unpacked copy: tar restores each file's original mtime, so with a target
# dir shared by several checkouts' jobs, a file edited before another checkout's last build
# looks older than that build, and cargo silently reuses the other checkout's code. rsync -c
# without -t rewrites only files whose contents changed, with the current time as their mtime:
# changed files rebuild, unchanged ones stay incremental.
command -v rsync > /dev/null || apt-get install -y -qq rsync >> $J/setup.log 2>&1
rsync -rlpc --delete $J/src/ /content/src/ || { echo "run.sh: rsync of the source failed" >&2; exit 2; }
rsync -rlpc --delete $J/extra/ /content/extra/ || { echo "run.sh: rsync of the extra files failed" >&2; exit 2; }
pushd /content/src > /dev/null || exit 1
nc=$(nproc); [[ $nc =~ ^[0-9]+$ ]] || nc=1
export CARGO_TARGET_DIR=/content/target CARGO_BUILD_JOBS=$nc EXTRA_DIR=/content/extra
# Colab VMs have no hardware counter of instructions (dmesg: "no PMU driver, software events
# only"; perf_event_open of the instructions counter fails). A checkout that ships a DynamoRIO
# client at tools/icount (as zed does: build.sh, install-dynamorio.sh, zed_icount.c) gets
# DynamoRIO 11.3.0 at /opt/dynamorio and the client built at /content/icount, once per VM and
# again when the client's source changes, and the two variables that name them; its test
# binaries then count instructions in software. Measured 2026-11-22 on an hm VM: 1.12x slower
# than native at 0.9 s of work (plus about 0.08 s of start-up), the same count on every run, and
# ratios within 0.1% of the hardware counter's. valgrind is for tests that run it. Other
# checkouts skip this.
if [ -e tools/icount/build.sh ]; then
  if ! cmp -s tools/icount/zed_icount.c /content/icount/zed_icount.c; then
    { { apt-get install -y -qq cmake g++ valgrind || { apt-get update -qq && apt-get install -y -qq cmake g++ valgrind; }; } &&
      bash tools/icount/install-dynamorio.sh /opt/dynamorio &&
      bash tools/icount/build.sh /opt/dynamorio /content/icount &&
      cp tools/icount/zed_icount.c /content/icount/zed_icount.c; } >> $J/setup.log 2>&1 ||
      echo "run.sh: building zed_icount failed (setup.log); benchmarks measure processor time" > $J/out/icount.log
  fi
  [ -e /content/icount/zed_icount.c ] &&
    export ZED_METER_DRRUN=/opt/dynamorio/bin64/drrun ZED_METER_ICOUNT=/content/icount/libzed_icount.so
fi
# A crate that is also a dylib (crate-type = ["rlib", "dylib"]) keeps one file name whatever its
# features. A build that resolves other features (a `cargo run -p` line, or a checkout whose
# manifests differ from the last one built here) overwrites dylibs that the other build's
# fingerprints still take as fresh, and every later build that links them fails with "cannot
# satisfy dependencies so `std` only shows up once". A step that fails so cleans the build
# directory, by its literal path, and runs once more, marked dylib-retry; the cost is one cold
# build, against a VM that would fail every later build.
cleaned=0
mixed() { [ $cleaned = 0 ] && command grep -a -q 'cannot satisfy dependencies so' "$1"; }
reclean() { cleaned=1; cargo clean --target-dir /content/target >> $J/out/clean.log 2>&1; }
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
    if [ $rc -ne 0 ] && mixed $J/out/$n.log; then
      printf '%s\t%s\t%s\t%s\t%s\tdylib-retry\n' "$line" $rc 0 $secs "$rss" >> $J/out/summary.tsv; reclean
      s=$(date +%s)
      /usr/bin/time -v -o $J/out/$n.time timeout 3600 bash -c "${line#!}" < /dev/null > $J/out/$n.log 2>&1; rc=$?
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
tar czf $J/out.tgz -C $J out; touch $J/DONE-$TOKEN
R
# A job left on this VM by an earlier job.sh run is cleared first, as pool.sh release does: its
# pgid file would stop the launch below ("[ -s pgid ] ||"), and the wait loop would then see the
# old dead group and report "job died" (seen twice, then reproduced, 2026-11-22).
printf '%%%%bash\np=$(cat /content/job/pgid 2>/dev/null)\nif [[ $p =~ ^[0-9]+$ ]] && [ "$p" -gt 1 ]; then kill -- -$p 2>/dev/null; sleep 2; fi\nrm -rf --one-file-system --preserve-root=all /content/job\nmkdir -p /content/job\n' | c > /dev/null
# An upload that fails is tried 3 times, 10 s apart, then the job stops with exit 6 before a launch
# without its files. colab exits 1 on an HTTP error (an unhealthy VM can answer every upload with
# HTTP 500); without this check the job went on and failed later as "launch failed".
# PIPESTATUS[0] is the upload's own exit, not grep's or tail's.
for f in src.tgz extra.tgz tests.txt run.sh; do
  for try in 1 2 3; do
    timeout 600 colab --auth=adc upload -s "$vm" "$out/$f" "/content/job/$f" 2>&1 | scrub | tail -1
    [ "${PIPESTATUS[0]}" = 0 ] && break
    [ $try = 3 ] && { echo "job.sh: upload of $f to $vm failed 3 times" >&2; exit 6; }
    sleep 10
  done
done
# The uploaded copies are emptied, not deleted (no rm from a variable path): each job keeps a
# copy of the worktree and the extra dir, and many jobs can fill a per-user /tmp quota.
: > "$out/src.tgz"; : > "$out/extra.tgz"
# Each job has its own token, so a DONE file left by an earlier job cannot end this one's wait.
# setsid gives run.sh its own process group, whose id `pool.sh release` kills.
tok=$(date +%s%N)
[[ $tok =~ ^[0-9]+$ ]] || die "bad token '$tok'"
# A dropped connection can lose the launch command (2026-11-22, t3: "Connection was lost"), so the
# launch is confirmed by a separate call that sees the pgid file, and retried up to 3 times.
for try in 1 2 3; do
  printf '%%%%bash\n[ -s /content/job/pgid ] || { setsid nohup bash /content/job/run.sh %s %s > /content/job/run.log 2>&1 & echo $! > /content/job/pgid; }\n' "$tok" "$prebuild" | c > /dev/null
  printf '%%%%bash\n[ -s /content/job/pgid ] && echo "LAUNCH""ED"\n' | c 60 | command grep -a -q LAUNCHED && break
  [ $try = 3 ] && { echo "launch failed on $vm"; exit 4; }
  sleep 10
done
# Wait for DONE. A failed poll (connection lost) is retried; a job process that is gone with no
# DONE file ends the wait with exit 5, so a dead job never makes this loop run forever. An empty
# or non-numeric pgid counts as dead: `kill -0` with no argument would only print an error.
while :; do
  r=$(printf '%%%%bash\np=$(cat /content/job/pgid 2>/dev/null)\nif [ -e /content/job/DONE-%s ]; then echo "JOB""DONE"; elif [[ $p =~ ^[0-9]+$ ]] && kill -0 "$p" 2>/dev/null; then echo "RUN""NING"; elif [ -e /content/job/DONE-%s ]; then echo "JOB""DONE"; else echo "JOB""DEAD"; fi\n' "$tok" "$tok" | c 60)
  case $r in *JOBDONE*) break;; *JOBDEAD*) echo "job died on $vm"; printf '%%%%bash\ntail -20 /content/job/run.log\n' | c; exit 5;; esac
  sleep 20
done
timeout 600 colab --auth=adc download -s "$vm" /content/job/out.tgz "$out/out.tgz" 2>&1 | scrub | tail -1
tar xzf "$out/out.tgz" -C "$out" && cp "$out/out/summary.tsv" "$out/summary.tsv" || die "no results in $out/out.tgz"
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
vm=$(./pool.sh acquire hm me) || vm=$(./pool.sh acquire l4 me) || { echo "no VM free; try again in 2 min"; exit 1; }
[ -n "$vm" ] || { echo "pool.sh printed no VM name"; exit 1; }
./job.sh "$vm" "$PWD" "$O" "$O.tests" > "$O.log" 2>&1; echo "job rc=$?"
./pool.sh release "$vm"          # always, also after a failure
cat "$O/summary.tsv"
```

### Instructions to give AI agents that use the pool

When several AI agents share the pool, each one needs these rules in its instructions (they
are what was given to the agents here):

- Run every build, test, clippy run and benchmark on a VM. Only editing, formatting, git and
  the push scripts run on the local machine. A job already running locally may finish there.
- If no VM can be had (every shape exits 3), wait 15 seconds and try again (the reaper stops a
  VM idle for 60 s, so a longer wait lets a freed VM be stopped instead of reused); do not fall back
  to running the job locally, or the local machine is overloaded again.
- **Benchmarks: run a whole scaling series in one job on one VM.** A Colab VM shares its host
  with other users, so a ratio measured across two VMs, or against a local number, means
  nothing. A ratio that fails is run once more on the same VM before it is analyzed; report
  both runs.
- Get VMs only through `pool.sh acquire`, never `colab new`; return them only through
  `pool.sh release`, never `colab stop`. Release also after a failure. Hold one VM at a time.
- Run `job.sh` in the background and keep working; do not poll it.
- Read `summary.tsv`; analyze a failure on the VM like a local one, including failures that
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
| 8 test threads on a 50 GB VM | Out-of-memory | Thread back-off; the cause of the peak is open (section 4) |
| Editing `job.sh` in place while jobs ran it | bash reads a script as it runs, so a running job can read the new text at its old offset and run half a command | Write `job.sh.new`, test it, then `mv` it over the old one: a running job keeps the old file (`/proc/<pid>/fd/255` shows it as deleted) |
| `pgrep -f '<pattern>'` to find a job | Matched the shell running the `pgrep` command itself, whose command line contains the pattern | Record the job's pid when it starts, or look a pid up with `ps -o pid,args -p <pid>` |
| An empty variable in `sed -i "/^$n /d"` | Would delete every line of the pool file | Check every value against a pattern first; `set -u` (section 6) |
| `kill -- -$(cat pgid)` | A pgid file holding 1 would kill every process on the VM | Accept only a number greater than 1 |
| `--tpu v5e1` | Not in `colab new --help`, which lists `V5E1` and `V6E1` | Use the spelling the help lists |
