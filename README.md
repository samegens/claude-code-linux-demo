# Claude Code Linux Kernel Demo

A live demo where Claude Code makes a real change to the Linux kernel source
tree — adding a brand new syscall — builds it, boots it, and proves the
change actually works, all in a few minutes.

## Goal

Show that Claude Code can operate on one of the largest, most demanding
codebases in existence (the Linux kernel) and produce a working, booting
change, not just a diff that looks plausible.

The change: a new syscall, `claude_fib`, that computes the n-th Fibonacci
number in the kernel, computed efficiently (better than naive O(n)/O(2^n)),
guarding against 64-bit overflow instead of silently wrapping. The exact
argument signature (return value vs. output pointer) and overflow boundary
are left for Claude to design live — see step 3 below — rather than fixed
here, so the demo shows real design work, not a replay of a known answer.
We call it from a tiny C program, generated to match whatever signature
Claude designs, to prove the running kernel actually contains the new code
and computes correct results.

## Why this approach

- **QEMU via `virtme-ng` (`vng`), not real hardware.** Booting a
  freshly built kernel on the presenter's actual Framework 16 laptop would
  be fast to demo but risky (bad kernel = need recovery) and slow to iterate
  on (real reboot). `vng` boots the just-built kernel in QEMU using the
  host's live root filesystem, in seconds, with no initramfs to build and
  no reboot of the host required.
- **A new syscall, not a proc/sysfs file.** More clearly "real kernel
  internals" to a developer audience: touches the syscall table, the
  syscall entry declarations, and kernel/sys.c.
- **Minimal virtme config, not defconfig/allyesconfig.** `vng --kconfig`
  generates a trimmed config with just enough drivers for the VM (virtio
  console/net/block, 9p, etc.), which builds far faster than a full distro
  config while still being a legitimate kernel build.
- **Run `vng` from this repo, not from the kernel tree.** `vng` shares
  the whole host filesystem into the guest over virtiofs and starts the
  guest shell in whatever directory you invoked it from. A kernel build
  tree accumulates 100k+ object/build files, and an interactive shell's
  prompt (e.g. starship running `git status` on every prompt draw) stat's
  and opens a large fraction of whatever directory it's sitting in. Doing
  that against the kernel tree can exhaust the shared virtiofs file
  descriptor budget and produce guest-side `ENFILE` ("too many open files
  in system") errors within seconds of booting — even though the kernel
  itself is fine. `vng --run <linux repo>` builds/boots the kernel from
  that path while leaving the guest shell's cwd (and therefore the
  prompt's filesystem chatter) in this much smaller repo instead.

## Prerequisites

- Linux host with:
  - `qemu-system-x86_64` (already installed via `dnf` on this machine)
  - `gcc` and standard kernel build dependencies
  - Python 3 + `pip`
- `virtme-ng` installed for the current user (no root needed):

  ```sh
  python3 -m pip install --user --break-system-packages virtme-ng
  export PATH="$HOME/.local/bin:$PATH"   # add to shell rc if not already
  ```

- A clone of the Linux kernel source tree (this demo uses the master branch from
  `https://github.com/torvalds/linux`).
- `lld` (LLVM's linker) for a noticeably faster `vmlinux` link/kallsyms
  stage — kbuild only accepts GNU `ld` or `LLD`, so this is the one step
  in this whole demo that needs `sudo`:

  ```sh
  sudo dnf install -y lld
  ```

- This repo and the Linux repo are next to eachother, so the Linux can be
  accessed through `../linux`.

## Step-by-step

1. **Generate a minimal build config for the VM:**

   ```sh
   cd ../linux
   vng --kconfig
   ```

   This writes a `.config` trimmed for booting under `vng`/QEMU (much
   faster to build than defconfig/distro configs).

2. **Build a baseline kernel first**:

   ```sh
   cd ../linux
   make -j$(nproc) LD=ld.lld
   ```

3. **Give Claude Code this prompt live from the linux directory**:

   ```text
   We're adding a new syscall, claude_fib, to the Linux kernel tree as a live demo. Work through it in this order:

   1. Design the syscall: the argument signature (return value vs. output pointer, etc.), and how it computes the n-th Fibonacci number for an unsigned integer n. Compute it efficiently (better than naive O(n)/O(2^n)) and guard against 64-bit overflow instead of silently wrapping.

   2. Before touching the kernel, write a test program at ../claude-code-linux-demo/test/claude_fib_test.c that calls the new syscall using the exact calling convention from your design, and prints the result (or error) for a handful of values — including some you expect to succeed and some you expect to hit the overflow guard. Build it with gcc on the host.

   3. Stop there and tell me it's ready. I'll boot the baseline kernel with vng myself and run the test program live to show the syscall doesn't exist yet (ENOSYS).

   4. Once I confirm that, implement the syscall matching your design from step 1 exactly, rebuild the kernel with lld for faster linking, and self-test it yourself (boot it with vng and run the test program, or exercise the syscall directly) to confirm correctness and that the overflow guard triggers at the right boundary, before handing back to me.

   5. Once you've confirmed it works, tell me it's ready. I'll boot it interactively myself to show it working live.
   ```

   **At hand-off point 1 (step 3 in the prompt)**, confirm the new syscall
   doesn't exist yet against the baseline kernel. Two ways to show this
   live, from this repo:

   - **Scripted one-shot** (fastest):

     ```sh
     cd ../claude-code-linux-demo
     vng --run ../linux -- test/claude_fib_test
     ```

   - **Interactive shell**:

     ```sh
     cd ../claude-code-linux-demo
     vng --run ../linux
     ```

     then inside the VM shell:

     ```sh
     uname -r          # confirm this is the VM's kernel, not the host's
     test/claude_fib_test
     echo $?
     strace -e trace=syscall test/claude_fib_test 2>&1 | grep -A1 ENOSYS
     ```

     `strace` shows the raw `syscall(...)` returning `-1 ENOSYS` at
     the kernel boundary — a concrete visual for a developer audience.
     Exit the VM shell with `exit` or Ctrl-D when done.

   **Explaining the interactive shell to an audience:** the VM shell
   looks identical to your normal host shell — same files, same user,
   same prompt — because `vng` boots the VM using your host's *live root
   filesystem* (shared over virtiofs/9p), not a separate disk image.
   That's what makes iteration fast, but it means "I'm in a different
   kernel now" isn't visually obvious. Three commands make it concrete
   (numbers below are one real run on this machine):

   | Check | Host | Inside `vng --run <linux repo>` |
   | --- | --- | --- |
   | `uname -r` | `7.1.10-200.fc44.x86_64` | `7.3.0-rc1+` |
   | `hostname` | `fluitzwaan` | `virtme-ng` |
   | `uptime` | up 1 day, 8h39m | **up 0 min**, load 0.00 |
   | `cat /proc/version` | Fedora's official build | shows *you* just built it: username, exact compiler/linker versions, and a build timestamp from moments ago |

   Framing that tends to land: "same car interior (filesystem), different
   engine (kernel)." `uptime` showing 0 minutes is usually the detail
   that visibly clicks — proof this shell session only just came into
   existence.

   **At hand-off point 2 (step 5 in the prompt)**, once Claude confirms
   the rebuilt kernel self-tests correctly, re-run the same test binary
   (no rebuild needed) to show it live:

   ```sh
   vng --run ../linux -- test/claude_fib_test
   ```

   or repeat the interactive-shell walkthrough from hand-off point 1 —
   same commands, now against the modified kernel, showing real Fibonacci
   values instead of `ENOSYS`.
