# chezmoi-apply-xattr

A POSIX `sh` utility that tags files under a [chezmoi](https://www.chezmoi.io/) target path with a `user.chezmoi` extended attribute (xattr), recording whether each file is `managed`, `unmanaged`, or `ignored` from chezmoi's perspective.

#### Why

chezmoi's own state (source directory, `chezmoi.yaml`, boltdb) doesn't surface at the filesystem layer, so tools that stat or list files in `$HOME` have no cheap way to know whether a given path is chezmoi-managed without shelling out to `chezmoi managed` on every lookup. This script runs once (e.g. after `chezmoi apply`) and stamps an xattr directly onto each file, so downstream consumers — a file manager, a shell prompt segment, a Neovim `BufReadPre` autocmd — can check with a single `getfattr`/`xattr` call instead of invoking chezmoi.

This is also why the script is a standalone utility rather than a chezmoi `run_` script: invoking `chezmoi` from within a script that chezmoi itself is running trips chezmoi's boltdb write-lock, since the parent `chezmoi apply` process still holds it. Running this afterward, from a separate invocation, avoids the deadlock.

#### What it does

1. Confirms `chezmoi` is on `$PATH` and resolves `chezmoi target-path`.
2. Confirms an xattr tool is available (`setfattr` or `xattr`) and that the destination filesystem actually supports xattrs, by writing and removing a probe file.
3. For each of `chezmoi managed`, `chezmoi unmanaged`, and `chezmoi ignored`, resolves the matching paths and writes `user.chezmoi=<managed|unmanaged|ignored>` to each regular file or directory (symlinks are skipped).
4. Logs every skip, failure, and summary count to a per-run log file.

#### Requirements

- POSIX-compliant `sh`
- `chezmoi`
- One of:
  - `setfattr` (Linux, part of `attr`/`acl` packages — on Arch: `pacman -S acl`)
  - `xattr` (macOS, or the Python `xattr` package elsewhere)
- A destination filesystem that supports extended attributes (most Linux filesystems do; some network or overlay filesystems do not — the script detects this and fails closed)

#### Installation

Place the script anywhere on `$PATH`, e.g. via chezmoi itself:

```sh
chezmoi add --create ~/.local/bin/chezmoi-apply-xattr
chmod +x ~/.local/bin/chezmoi-apply-xattr
```

Since invoking this from inside a chezmoi `run_` script deadlocks (see above), wire it up as a wrapper around the `chezmoi` command instead, so it runs *after* chezmoi has released its lock:

```zsh
chezmoi() {
    command chezmoi "$@"
    local exit_status=$?
    case "$1" in
        apply|init) chezmoi-apply-xattr ;;
    esac
    return $exit_status
}
```

#### Usage

```sh
chezmoi-apply-xattr [-v|--verbose] [--]
```

| Flag | Effect |
|---|---|
| `-v`, `--verbose` | Also print `INFO`-level log lines to stderr (these are always written to the log file regardless) |
| `--` | End option parsing |

Reading the tag back:

```sh
# Linux
getfattr -n user.chezmoi --only-values -- "$file"

# macOS
xattr -p user.chezmoi "$file"
```

#### Logging

Logs are written to `${XDG_STATE_HOME:-$HOME/.local/state}/chezmoi-apply-xattr.log`, one line per event, timestamped in UTC (`YYYY-MM-DDTHH:MM:SSZ`). `NOTICE`, `WARN`, `ERROR`, and `FATAL` levels are always echoed to stderr as well; `INFO` is only echoed with `-v`.

#### Exit codes

| Code | Meaning |
|---|---|
| `0` | Completed (individual per-file tag failures are logged, not fatal) |
| `1` | Environment or resolution failure (target-path, xattr probe, path list) |
| `2` | Unknown option |
| `127` | Required command not found (`chezmoi`, `setfattr`/`xattr`) |

#### Known limitations

- Paths containing newline characters are not supported (the script reads `chezmoi managed`/`unmanaged`/`ignored` output line-by-line).
- Symlinks are intentionally skipped rather than tagged or dereferenced.

#### Related conventions

- The `user.` xattr namespace prefix follows the Linux extended attribute convention described in `attr(5)`: unprivileged processes may only set attributes in this namespace on files they own.
- Log placement follows the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)'s `$XDG_STATE_HOME`.

#### License

MIT, © 2026 chewygumxx. See license header in the script itself.
