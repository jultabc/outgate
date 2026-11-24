# gitleaks-traces + trace-guard

A small rule pack and a POSIX `sh` pre-push bridge for [gitleaks](https://github.com/gitleaks/gitleaks).
It looks for personal traces before a push leaves your machine: personal e-mail
addresses, home directory paths, Korean resident-number and phone patterns, and
words you add yourself (names, internal repository names, host names).
It also runs gitleaks' own default secret rules.

Tested with gitleaks **8.30.1**. Other major versions are refused.

## What is checked

For the commits a push would add (range = pushed tip, excluding every tip that
already exists on the remote and locally):

- file contents, including lines that only appear in merge-conflict resolutions
- file and folder names: every path in the range (adds, modifies, deletes, both names of a rename; empty files and pure renames included) is written as a `TRACEGUARD-PATH:<path>` line (raw names via `git log -z`; control characters inside a name, including newline and tab, become `?`; backslashes and quotes stay) and scanned with `gitleaks stdin`; the `-name` rules of the pack only match those lines
- commit messages, author and committer name and e-mail
- annotated tag objects (tagger, message), including tag-of-tag chains
- the names of the pushed branches and tags

Everything runs twice for files: once with `gitleaks-traces.toml` (plus your
words file), once with gitleaks' default rules.

## What is NOT handled

- Binary files (gitleaks skips them).
- Submodules and Git LFS content.
- Issues, repository descriptions, release notes.
- Pull request titles, bodies and comments. Check them by hand:
  `printf '%s\n' "$TEXT" | gitleaks stdin -c gitleaks-traces.toml --redact`
- Packages and release attachments. Unpack and check:
  `gitleaks dir <unpacked folder> -c gitleaks-traces.toml --redact --ignore-gitleaks-allow`
  (add a second run without `-c` for the default secret rules; do not use `useDefault` inside the pack, see below).
- Refs other than `refs/heads/*` and `refs/tags/*` (for example `refs/notes/*`) are refused.
- Deleting a remote branch or tag is allowed without a scan (nothing is uploaded).

## Finding the file

Findings print a path id instead of the path. To see the names, run locally:

    git log -m --name-only --no-renames <short commit> -1

and hash a candidate: `printf '%s' '<path>' | shasum -a 256 | cut -c1-12`. The line number is the line in the file
(file findings) or in the path list (name findings, the list is the sorted unique names of the range).
The allow-file fingerprint format is `<full commit sha>:<path>:<rule id>:<line>` (one per line, `.gitleaksignore` format);
build it from the local output. Name findings and findings in commit, tag or ref text have no file in their
fingerprint and cannot be allowed through the ignore file.

Not scanned: commit header fields other than author, committer and message (for example `mergetag`).

Whole old history: run `gitleaks git <repo> -c gitleaks-traces.toml --log-opts="-m --all" --redact`
(without a range).

## Install

Requirements: `git`, `gitleaks` 8.30.1 or newer 8.x on `PATH`, `mktemp`. `perl`
is optional (it enforces the 60 second limit on `git ls-remote`); `python3` is
optional (it prints finding details; without it only counts are shown).

As a hook in one repository:

    # symlink, so the pack is found next to the real script:
    ln -s /absolute/path/to/pack/bin/trace-guard .git/hooks/pre-push

For all repositories: put a `pre-push` file that runs `exec /absolute/path/to/pack/bin/trace-guard "$@"` in
a folder and set `git config --global core.hooksPath <folder>`.

## Configuration (`git config`)

| Key | Meaning |
|---|---|
| `traceguard.pack` | Path to the rule pack. Default: `../gitleaks-traces.toml` next to the real path of `trace-guard`. |
| `traceguard.words` | Optional file with your own `[[rules]]` tables, see `words.example.toml`. Any other table (`[allowlist]`, `[extend]`, `[[allowlists]]` ...) makes the hook refuse to run. |
| `traceguard.ignorefile` | Optional allow list in `.gitleaksignore` format (one fingerprint per line, built as described under "Finding the file"). Must be an absolute path outside the repository and must not be a symbolic link (a symlinked directory that resolves into the repository is refused too). Its path is not printed (it is hidden unless `TRACEGUARD_SHOW=1`). |

Allow-style files inside the pushed repository (`.gitleaksignore`,
`.gitleaks.toml`, `.gitattributes`, `gitleaks:allow` comments) are ignored on purpose, so a commit cannot switch the
check off. `GITLEAKS_*` environment variables are cleared. The pack must not use `[extend] useDefault = true`:
gitleaks' default global allowlist (lock files, `node_modules`, ...) would then also silence these rules.

## Behaviour

- Exit 0 = nothing found. Any other exit blocks the push.
- Unknown is blocked: gitleaks missing, version older than 8.30.1 or not major 8, unreadable or invalid
  config, `git ls-remote` failure or timeout, unexpected gitleaks exit status, unsupported ref.
- Output never contains a file path, ref name, push URL or fingerprint. A finding prints: report kind, rule id, short commit (8 hex), line, and a path id (first 12 hex of the SHA-256 of the path). Refs are shown as `ref#<position>`. `TRACEGUARD_SHOW=1` in the environment prints raw values (local terminal only).
- gitleaks can exit 0 although its own `git log` failed. The hook therefore first reads every object of the range itself (`git log -p -m` in the temporary clone) and treats any error line on gitleaks' stderr as unknown = blocked (partial clones with missing blobs are blocked).
- Refs are classified by the remote ref field of the pre-push input: `refs/heads/*` and `refs/tags/*` are scanned, a delete is skipped, anything else blocks. Each input line must have exactly 4 fields with 40 or 64 hex shas.
- The scan runs in a temporary shared clone without a work tree and is deleted afterwards.

### Race between `ls-remote` and push

The hook asks the remote for its branch and tag tips at hook time. If somebody pushes between that
question and your push, the range was computed from slightly older tips: the hook then scans
more than needed, never less. Tips that do not exist locally are ignored, which also scans more.
In a shallow repository everything reachable is scanned.

## Rules

`gitleaks-traces.toml`: `personal-email`, `home-folder`, `kr-rrn`, `kr-phone` and a `-name` twin of each (the `-name` rules only work on the marker lines the hook writes; do not use them with `gitleaks git` or `gitleaks dir`).
The e-mail domain list and the example/system account names that are never reported as a home folder are listed
in the comment block at the top of the file.

## Portability note

Name conversion uses `tr '\001-\037\177' '?'` and `tr '\000' '\n'` under `LC_ALL=C`. Octal escapes and ranges work the same in BSD tr (macOS, tested) and GNU tr.

An allow file that is a hard link to a file inside the repository cannot be detected by the hook.
