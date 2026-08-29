# mac-tidy

Reclaim RAM on a macOS dev machine by stopping leaked background processes.

Dry run by default. It prints what it *would* stop, ranked by how much memory each
group actually holds, and changes nothing until you pass a flag.

```
$ mac-tidy

=== before ===
Load Avg: 421.83, 334.73, 159.39
PhysMem: 35G used (6986M wired, 12G compressor), 377M unused.
Swap: total = 11264.00M  used = 9988.44M  free = 1275.56M  (encrypted)
  swap is 88% full — this is what makes the load average spike; free RAM, not CPU

=== biggest wins, ranked ===
    5818 MB  Chrome (44 procs)                  --chrome  restart, session restored
    1472 MB  Figma                              --apps    graceful quit
    1469 MB  ChatGPT                            --apps    graceful quit
     540 MB  idle claude bg sessions            --claude  see caveat below
     133 MB  stale dev daemons (22 procs)       --go      always safe
```

## Why the load average lies

A load average of 400 on a 14-core laptop looks like a runaway process. It usually
isn't. If the CPU is mostly idle while the load number climbs, those are threads
blocked on page-ins: memory is full, the compressor is working, and swap is nearly
exhausted. Nothing you kill in Activity Monitor's CPU tab will help, because CPU was
never the constraint.

So mac-tidy ranks candidates by resident memory, not CPU, and tells you plainly when
swap pressure is what you are actually feeling.

## Install

```sh
curl -o ~/.local/bin/mac-tidy https://raw.githubusercontent.com/0-sv/mac-tidy/main/mac-tidy
chmod +x ~/.local/bin/mac-tidy
```

Or clone and symlink it somewhere on your `PATH`. No dependencies beyond what ships
with macOS, plus `adb` and `xcrun` if you want the Android/iOS checks to do anything.

## Usage

```
mac-tidy                report only, change nothing (default)
mac-tidy --go           stop tier-1 stale dev daemons (safe)
mac-tidy --apps         also gracefully quit idle RAM-hog GUI apps
mac-tidy --chrome       also restart Chrome (session restored, tabs unloaded)
mac-tidy --claude       also stop idle Claude Code background sessions
mac-tidy --all          everything above
mac-tidy --purge        run `sudo purge` at the end
mac-tidy -v             show every pid considered
```

### Tier 1 — `--go`, always safe

Things that leaked and either respawn on demand or cost nothing to restart:

- MCP servers orphaned by a dead parent session (reparented to launchd)
- AWS SSM port-forward tunnels older than 90 minutes **with no live connection**
- Dev servers and test runners (expo, metro, jest, vitest, vite, `tsc --watch`)
  that are orphaned or whose parent shell is gone
- Gradle / Kotlin compile daemons
- `adb` server when no device is attached
- `CoreSimulatorService` when no simulator is booted and it is over 2h old
- Detached shells with no children
- `watchman`, but only if it has ballooned past 800MB

### Tier 2 — `--apps`, `--chrome`

Graceful quits via AppleScript, so unsaved work still prompts you. Chrome is
quit and reopened: it restores the session with tabs unloaded, which is usually
the single largest win on a machine that has been up for days.

Override the app list with `MAC_TIDY_APPS="Figma Slack"`.

### Tier 3 — `--claude`

Stops idle Claude Code background sessions. Sessions actively running a job are
always skipped (they hold a `caffeinate` descendant). **Caveat:** a paused job you
meant to resume would be lost.

## Safety

- Dry run unless you pass a flag. Tiers past the first each need their own flag.
- Never touches a process outside your own uid.
- Protects its own process tree, so it cannot kill the shell or session running it.
- Sends `SIGTERM`, waits 3s, and only then `SIGKILL`.
- Age and size gates stop it fighting respawns — see below.

## Two things it deliberately will not kill

Both learned by killing them and watching what happened:

- **`watchman`** respawns immediately and grows to ~550MB re-crawling your project
  tree. Killing it costs more than it frees, so it is only targeted when genuinely
  bloated.
- **`biome lsp-proxy`** leaks one process per editor window and reload — 20 of them
  is normal after a few days. VS Code's extension host respawns all of them within
  seconds of a kill, so mac-tidy reports them and tells you to reload the window
  instead. This applies to most LSP proxies, not just Biome.

Kill-respawn loops are the main failure mode of scripts like this. Every group has an
age gate for that reason.

## License

MIT
