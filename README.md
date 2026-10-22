> **This project's home is now [github.com/globe-teck/agent-box-oom-guard](https://github.com/globe-teck/agent-box-oom-guard).** This copy stays up for existing links. Please open issues and pull requests there.

# agent-box-oom-guard

[![CI](https://github.com/leighsrodney-prog/agent-box-oom-guard/actions/workflows/ci.yml/badge.svg)](https://github.com/leighsrodney-prog/agent-box-oom-guard/actions/workflows/ci.yml)

A priority-aware out-of-memory guard for a shared Linux machine running several AI agents, each with its own browser.

## The problem

When several agents share one box, each with a Chrome or Playwright session and a Node or Python runtime, memory runs out quickly. With no swap, the kernel OOM killer picks whichever process has the highest `oom_score`. That can be the one browser session that must not die, such as a long-lived logged-in dashboard. Before the kill, the box can thrash for minutes, and every agent freezes. These boxes are often containers with no systemd and no cron, so nothing keeps a watchdog running.

## Design

| Piece | What it does |
|---|---|
| `bin/swapfile_setup.sh` | Creates and enables a swapfile and sets `vm.swappiness`. **Dry-run by default**; pass `--apply` to make changes. `--persist` also adds the swapfile to `/etc/fstab`. |
| earlyoom (external) | Kills a process early, before the kernel thrashes. The config notes are below. |
| `bin/mem_watch.sh` | Checks `MemAvailable` every 60 s. Under the threshold, it calls a notify hook and lists the top 5 processes by RSS. Repeat alerts are deduped for 30 minutes. It re-runs `chrome_oom_prio.sh` on every loop. |
| `bin/chrome_oom_prio.sh` | Finds each Chrome main process, reads its `--remote-debugging-port`, and fetches `/json/list`. That call is **read-only**: no tab is opened, closed or driven. If a tab URL matches a "critical" pattern, that browser is protected. |
| `bin/oom_guard_ensure.sh` | Idempotent and protected by a `flock` lock. It turns the swapfile back on, starts earlyoom and mem_watch if they are not running, and applies the Chrome ranks. Hook it into `.bashrc` on boxes with no init system. |

### Kill order (`oom_score_adj`)

| Process | Default adj | Env var |
|---|---|---|
| Renderers of ordinary browsers (tabs) | **+800** (killed first) | `OOMG_ADJ_RENDERER` |
| Everything else (agent runtimes, jobs) | unchanged (≈0) | |
| Main process of ordinary browsers | **-100** | `OOMG_ADJ_MAIN` |
| Critical browser: main process and **all** descendants | **-500** (killed last) | `OOMG_ADJ_CRITICAL` |

The resulting kill order is: ordinary tabs, then the largest remaining process, then the critical browser. Chrome resets renderer scores when tabs are created, so the script runs again on every mem_watch loop.

A browser counts as critical when either:
- a tab URL from `/json/list` matches `OOMG_CRITICAL_PATTERNS`, an extended regex that defaults to `example\.com/critical`. Example: `OOMG_CRITICAL_PATTERNS='app\.example\.com/dashboard|admin\.example\.org'`; or
- its `--user-data-dir` matches `OOMG_CRITICAL_PROFILE_RE`, which is off by default.

### earlyoom: two choices that matter

- **Don't add `node` to `--avoid`.** earlyoom matches the process name (`comm`). Playwright drivers, MCP servers, agent daemons and a runaway `node build.js` all show up as `node`. Avoiding `node` would protect the very job that is eating memory. Avoid only names that are truly unique, such as your X server and init.
- **Don't use `--prefer chrome`.** Every Chrome process, including the critical one, is named `chrome`, so `--prefer` can't tell them apart. Rank them with `oom_score_adj` instead. earlyoom honours it, because it picks by `oom_score`.

Default in the ensure script: `earlyoom -m 5 -s 10 -r 3600 -p --avoid '^(Xvfb|Xorg|sshd|init|tini|earlyoom|mem_watch.sh)$'`. Override it with `OOMG_EARLYOOM_ARGS`.

## Install

```bash
git clone https://github.com/leighsrodney-prog/agent-box-oom-guard.git ~/agent-box-oom-guard
~/agent-box-oom-guard/bin/swapfile_setup.sh --size 4G             # review the plan
~/agent-box-oom-guard/bin/swapfile_setup.sh --size 4G --apply
sudo apt-get install -y earlyoom                                  # optional but recommended
OOMG_CRITICAL_PATTERNS='app\.example\.com/critical' ~/agent-box-oom-guard/bin/chrome_oom_prio.sh  # check the output
```

Then add `examples/bashrc-snippet.sh` to `~/.bashrc` **above** the interactive guard (`case $- in *i*) ;; *) return;; esac`). That way non-interactive shells started by agents also keep the guard alive. Put your `OOMG_*` settings above that snippet.

**Notify hook:** set `OOMG_NOTIFY` to any command. It is called as `cmd "<title>" "<body>"`. A non-zero exit is logged as `notify-failed`. See `examples/notify-webhook.sh`. With no hook set, alerts go to `logger` and stderr.

State and logs go to `${XDG_STATE_HOME:-~/.local/state}/oom-guard/`.

Lowering `oom_score_adj` below its current value needs `CAP_SYS_RESOURCE`. The script tries a direct write first, then `sudo -n tee`, and it never prompts. Raising renderers to +800 works without privileges.

## Undo

```bash
# 1. remove the snippet from ~/.bashrc
pkill -f agent-box-oom-guard/bin/mem_watch.sh
sudo pkill -x earlyoom
sudo swapoff /swapfile && sudo rm /swapfile        # and remove its line from /etc/fstab if you used --persist
sudo sysctl -w vm.swappiness=60                     # or your previous value
rm -rf ~/.local/state/oom-guard
```
The `oom_score_adj` values are per process and go away when the processes exit. To reset them sooner, restart the browsers.

## What is and isn't tested

**Tested in CI** (pytest, 23 tests, no root): a fake `/proc` tree (`comm`, `cmdline`, `stat`, `status`, `oom_score_adj`, `meminfo`) and a fake CDP fetcher that serves JSON fixtures. The tests cover:
- critical, ordinary and renderer ranking
- descendants found through the ppid chain
- custom patterns, and matching on the URL only (not the title)
- missing or unreachable debug ports
- the profile regex
- idempotence and picking up new PIDs
- dry-run
- the only CDP endpoint requested is `/json/list`
- mem_watch threshold, dedupe, re-alert after the window, notify failure, logging and the per-loop prio call
- swapfile dry-run
- the ensure lock

shellcheck also runs on all scripts.

**Not tested in CI:** writing real `/proc/<pid>/oom_score_adj`, the `sudo -n` paths, real swapfile creation (`--apply`), actually starting earlyoom, the `.bashrc` hook in a real login, and real Chrome or CDP. The design comes from a setup that ran on a real multi-agent box. This repo is a clean rewrite and has not run on a live box as is. Linux only; it needs bash 4+, `flock`, `pgrep` and `curl`.

## Authorship

Built with AI coding assistance. I defined the requirements, ran the tests, and reviewed the changes.

## License

MIT. See [LICENSE](LICENSE).
