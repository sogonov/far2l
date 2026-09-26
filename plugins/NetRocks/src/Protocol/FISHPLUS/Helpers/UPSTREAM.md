# Where helper.sh and helper.ps1 come from, and how to update them

`helper.ps1` in this directory is a **byte-for-byte copy** from f4. `helper.sh`
is a copy plus one mechanical delta, described under **Local delta** below and
re-applied by `sync-from-f4.sh` on every refresh:

| File | Upstream | Path | Commit | License |
|---|---|---|---|---|
| `helper.sh`  | https://github.com/unxed/f4 | `plugins/netfox/fishplus/helper.sh`  | `5fee8a6c` (missing `info` target reported as `no such file or directory`) | BSD-3-Clause (see the f4 repository) |
| `helper.ps1` | https://github.com/unxed/f4 | `plugins/netfox/fishplus/helper.ps1` | `a2b69c8d` (fix POSIX paths in pwsh background job runspaces on Unix) | BSD-3-Clause (see the f4 repository) |

Apart from that delta they are deliberately **not** edited here, not even to
remove the parts NetRocks does not call yet. f4 is where the protocol is developed, and every local edit
would have to be re-applied by hand on every refresh. Keeping the files
identical makes an update a copy plus a version bump.

`helper.sh` is uploaded to every peer whose login shell is POSIX; `helper.ps1`
is uploaded when the transport probes into a PowerShell host instead. Both
speak the same FISH+ wire; the only difference is how the bootstrap line
delivers them (see `../FishPlusScript.cpp`, `BootstrapLine()` vs
`BootstrapLinePwshB64()`).

The only thing done to either at runtime is what f4 does too: the literal
`__F4_TOKEN__` is replaced with the per-session token, and comments and blank
lines are stripped before upload (`FishPlusScript.cpp`, `Compact()`).

## Local delta

`helper.sh` prints its protocol lines with `echo "..."` in eleven places. A
POSIX `sh` builtin `echo` interprets backslash escapes with no opt-in - it is
`dash` on Debian and Ubuntu, so this is the common case - and a filename is
allowed to contain a backslash. On such a peer:

```sh
$ dash -c 'V="weird\name.txt"; echo "$V"'
weird
ame.txt
```

The worst of the eleven is `f4_end`, which prints the response terminator: it
is called as `f4_end err "$(f4_flat "$F4OUT")"`, and `f4_flat` folds real
`\n\r\t` to spaces but leaves a backslash alone. So a file named
`weird\name.txt` splits the terminator itself in two, and the client loses
stream sync rather than merely mis-parsing one reply. Reproduced against a
Linux peer: `enum` listed the name correctly while `info` on it answered
`stat failed (2)`.

The delta replaces every `echo "<text with a substitution>"` that writes to the
protocol stream with `printf '%s\n' "<same text>"`, which is what upstream
already does in ten other places in the same file - the eleven are an
oversight, not a style. One `echo` writing to stderr (`>&2`) is left alone, and
so are the ones printing a bare literal.

`sync-from-f4.sh` re-applies this after copying, so a refresh cannot silently
drop it, and reports how many sites it touched. **When that count comes out
zero, upstream has taken the fix**: drop the delta from the script, and this
section with it. Upstream tracking: <https://github.com/unxed/f4/issues>

## Updating

```sh
# from the far2l source root
NetRocks/src/Protocol/FISHPLUS/Helpers/sync-from-f4.sh /path/to/f4
```

The script copies both files, verifies the token placeholder is still present
and prints the two upstream protocol version numbers so the table above can be
updated.

## What has to be checked after an update

The helpers and the C++ client are two halves of one protocol. These are the
places that are coupled, in the order they would break:

1. **`F4PROTO` in helper.sh (`$F4PROTO` in helper.ps1) vs `PROTOCOL_VERSION`
   in `FishPlusScript.h`.** A bump means the wire format changed; read f4's
   `FISH+.md` changelog before doing anything else. The handshake refuses a
   mismatch rather than guessing. Both helpers must report the same number.
2. **The bootstraps.** `BootstrapLine()` in `FishPlusScript.cpp` is a port of
   `BootstrapLine()` in f4's `script.go`, and `BootstrapLinePwshB64()` is a
   port of `Base64BootstrapLinePwsh()`. Each must agree with its Go peer on
   the ready marker (`F4RDY<token>`) and, for the sh path, the end marker
   (`F4EOF`).  If f4 changes how a helper is fed in, this is what changes
   with it.
3. **`Compact()`.** Same file, port of the Go function of the same name. Both
   helpers are written so that they survive comment stripping — no
   here-documents, no multi-line literals, no PowerShell here-strings. If
   upstream ever adds one, `Compact()` must learn about it or the upload will
   break in a way that looks like a hung session.
4. **Listing formats.** `FishPlusListing.cpp` ports f4's `fs.go` and `ls.go`.
   The formats are pinned by `F4FMT_FIND`, `F4FMT_STAT`, `F4FMT_BSD` and
   `F4LSOPT` in the helper; a new column anywhere means a new field count here.
5. **The `flavor:` feature tag.** The pwsh helper announces `flavor:pwsh` in
   its banner so the client can key path translation (Cygwin-shape `/c/foo`)
   on it. If a new flavor ever appears, `Features::Flavor()` and the C++
   callers that consume it have to grow with it.
6. **New commands.** They cost nothing: an unknown command is simply never
   sent. Adding support for one is additive on this side.

## Running the protocol tests after an update

f4's own `*AgainstLocalShell` and `*AgainstLocalPwsh` tests are the reference.
On this side the equivalent is to point a FISH+ site at `localhost` and walk
a directory tree, but the cheaper check is f4's, because both clients speak
the same wire protocol to the same scripts:

```sh
cd /path/to/f4 && go test ./plugins/netfox/fishplus/
```

Run it at least once with `/bin/sh` pointing at `dash`, and preferably also at
`busybox sh`. `bash` is forgiving in ways the shells on real hosts are not.
For the PowerShell helper, the equivalent tests are gated on `pwsh` being on
PATH.
