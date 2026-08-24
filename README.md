# chezmoi-apply-xattr

A POSIX `sh` utility that tags files under a [chezmoi](https://www.chezmoi.io/)
target path with a `user.chezmoi` extended attribute (xattr), denoting whether
each file is `managed`, `unmanaged`, or `ignored`.

## Why

chezmoi's own state (source directory, `chezmoi.yaml`, boltdb) doesn't surface
at the filesystem layer, so tools that stat or list files in `$HOME` have no
cheap way to know whether a given path is chezmoi-managed without shelling out
to `chezmoi managed` on every lookup. This script runs once (e.g. after
`chezmoi apply`) and stamps an xattr directly onto each file, so downstream
consumers (eg. a file manager, a shell prompt segment, a Neovim `BufReadPre`
autocmd) can check with a single `getfattr`/`xattr` call instead of invoking
chezmoi.

This is also why the script is a standalone utility rather than a chezmoi
`run_` script: Invoking `chezmoi` from within a script that chezmoi itself is
running trips chezmoi's boltdb write-lock, since the parent `chezmoi apply`
process still holds it. Running this afterward, from a separate invocation,
avoids the deadlock.

## What it does

1. Confirms `chezmoi` is on `$PATH` and resolves `chezmoi target-path`.
2. Confirms an xattr tool is available (`setfattr` or `xattr`) and that the
   destination filesystem actually supports xattrs, by writing and removing a
   probe file.
3. For each of `chezmoi managed`, `chezmoi unmanaged`, and `chezmoi ignored`,
   resolves the matching paths and writes
   `user.chezmoi=<managed|unmanaged|ignored>` to each regular file or directory
   respectively (symlinks skipped).
4. Logs every skip, failure, and summary count to a per-run log file.

## Requirements

- POSIX-compliant `sh`
- `chezmoi`
- One of:
  - `setfattr` (Linux, typically part of `attr`/`acl` packages)
  - `xattr` (macOS, or the Python `xattr` package elsewhere)
- A destination filesystem that supports extended attributes (most Linux
  filesystems do; some network or overlay filesystems do not. The script detects
  this and fails closed)

## Installation

Place the script anywhere on `$PATH`, e.g. via chezmoi itself:

```jsonc
// ~/.local/share/chezmoi/.chezmoiexternals/chezmoi-apply-xattr.jsonc
// Assumes '~/.local/bin' is a PATH directory

{
    ".local/bin/chezmoi-apply-xattr": {
        "type": "file",
        "executable": true,
        "url": "https://github.com/chewygumxx/chezmoi-apply-xattr/releases/download/v1.0.0/chezmoi-apply-xattr"
    }
}
```

Since invoking this via chezmoi `run_` script deadlocks (see above), include it
within a wrapper around the `chezmoi` command instead, such that it executes
post-lock.

```sh
chezmoi() {
    command chezmoi "$@" || return
    [ "$1" -eq "apply" || "$1" -eq "init" ] && chezmoi-apply-xattr
}
```

## Usage

```sh
chezmoi-apply-xattr [-v|--verbose] [--]
```

| Flag              | Effect                                      |
|-------------------|---------------------------------------------|
| `-v`, `--verbose` | Also print `INFO`-level log lines to stderr |
| `--`              | End option parsing                          |

Reading the tag back:

```sh
# Linux
getfattr -n user.chezmoi --only-values -- "$file"

# macOS
xattr -p user.chezmoi "$file"
```

## Logging

Logs are written to
`${XDG_STATE_HOME:-$HOME/.local/state}/chezmoi-apply-xattr.log`, one line per
event, timestamped in UTC (`YYYY-MM-DDTHH:MM:SSZ`). `NOTICE`, `WARN`, `ERROR`,
and `FATAL` levels are always echoed to stderr as well; `INFO` is only echoed
with `-v`.

## Exit codes

| Code  | Meaning                                                                 |
|-------|-------------------------------------------------------------------------|
| `0`   | Completed (individual per-file tag failures are logged, not fatal)      |
| `1`   | Environment or resolution failure (target-path, xattr probe, path list) |
| `2`   | Unknown option                                                          |
| `127` | Required command not found (`chezmoi`, `setfattr`/`xattr`)              |

## Known limitations

- Paths containing newline characters are not supported (the script reads
  `chezmoi managed`/`unmanaged`/`ignored` output line-by-line).
- Symlinks are intentionally skipped rather than tagged or dereferenced.
