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

   The `[ -s pgid ] ||` guard makes the launch safe to repeat.
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
   Killing the process group matters: a job still running when its VM is handed on would write
   into the next job's directory and mix its results into the new job's.

### Memory-heavy test runs: back off on out-of-memory

Run each test command with a thread count, and when it dies of out-of-memory, run it again
with half the threads, down to 1. Record the peak memory with `/usr/bin/time -v`:

```bash
t=8
while :; do
  /usr/bin/time -v -o $n.time timeout 3600 cargo test $line -- --test-threads=$t > $n.log 2>&1; rc=$?
  if [ $rc -ne 0 ] && [ $t -gt 1 ] && grep -q -E 'SIGKILL|signal: 9|memory allocation of .* failed' $n.log; then
    t=$((t / 2)); continue
  fi
  break
done
awk '/Maximum resident/ {print $6}' $n.time     # peak RSS in kB
```

This exposed a real problem worth knowing about. One Rust test binary peaked at 40 GB on a
single thread, and out-of-memory at 51.5 GB with 8 threads, although no single test in it
used more than 3.0 GB. The cause was glibc's allocator keeping each finished test's freed
memory in its arenas instead of returning it to the system. Calling `malloc_trim(0)` at the
end of each test thread (in the test harness only) cut a two-test process from 4.17 GB to
2.49 GB. If a test binary's peak is far above its largest test, look for this before buying
a bigger VM.

## 5. Shapes, cost and limits

Units per hour are what one VM of the shape costs, read from `colab usage` with only that VM
running. Cores and RAM are what the VM reports (`nproc`, `free -g`).

| Shape (`new` flags) | Cores | RAM | Units/hr | Units per core-hour | Notes |
|---|---|---|---|---|---|
| standard CPU (none) | 2 | 12 GB | 0.08 | 0.040 | Cheapest per core |
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

Creating a VM per job wastes the setup time, and leaving VMs idle wastes units. A small
shared pool solved both. The state is a text file, one line per VM:
`<name> <shape> <busy|idle> <owner> <epoch of last change>`, and every change takes an
`flock`, so two clients never get the same VM.

- **acquire `<shape>` `<owner>`**: under the lock, take an idle VM of the shape if there is
  one and mark it busy; otherwise `colab new -s <shape><k> <flags>` with the next free
  number. If `new` fails (limit reached, `Service Unavailable`), return an error so the
  caller can wait, try another shape, or run locally.
- **release `<name>`**: kill the job's process group and delete `/content/job` on the VM,
  then mark the VM idle with the time.
- **reap `<seconds>`**: stop every VM idle longer than that. Run it every 15 s from a
  background loop (`while :; do pool.sh reap 60; sleep 15; done`); a minute of idleness was
  the rule used here.
- **list**: print the file.

Because the job directory is deleted on release but the toolchain and build cache stay, a
VM handed from one job to the next keeps its warm build. Keep the pool busy: queue work so
that a released VM is acquired again within the reap interval.

To measure a shape's limit, acquire VMs of that shape until `new` fails, and read
`colab usage` with each running alone to get its rate. Stop all of them afterward.

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
| 8 test threads on a 50 GB VM | Out-of-memory | Thread back-off; and the allocator fix in section 4 |
