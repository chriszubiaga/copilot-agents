---
name: bash
description: 'Deep knowledge for writing, reviewing, and debugging Bash scripts on Linux and macOS. Covers POSIX portability, GNU vs BSD utilities, error handling, shellcheck patterns, and common scripting idioms. Load when the user is working with .sh files, shell scripts, or shell-related automation targeting Linux or macOS.'
---

# Bash Scripting Skill

## Scope

This skill applies to Bash and POSIX-compatible shell scripts targeting **Linux** and/or **macOS**. It covers authoring, debugging, portability review, and safe execution patterns.

---

## Script Header Requirements

Every generated script must start with:

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

- `set -e` — exit immediately on any non-zero return code.
- `set -u` — treat unset variables as errors.
- `set -o pipefail` — propagate failures through pipelines (not just the last command).
- `IFS=$'\n\t'` — prevent word-splitting on spaces in filenames and variable values.

For POSIX-only scripts (no Bash features), use `#!/bin/sh` and drop Bash-specific flags.

---

## Error Handling Patterns

### Error trap with context

```bash
err() {
  echo "[ERROR] line ${1}: ${2}" >&2
  exit 1
}
trap 'err ${LINENO} "${BASH_COMMAND}"' ERR
```

### Cleanup on exit

```bash
TMPDIR_WORK=""
cleanup() {
  [[ -n "${TMPDIR_WORK}" ]] && rm -rf "${TMPDIR_WORK}"
}
trap cleanup EXIT
TMPDIR_WORK=$(mktemp -d)
```

### Retry logic

```bash
retry() {
  local attempts="${1}"; local delay="${2}"; shift 2
  local i=0
  until "$@"; do
    i=$(( i + 1 ))
    [[ "${i}" -ge "${attempts}" ]] && return 1
    echo "Retrying in ${delay}s (attempt ${i}/${attempts})..." >&2
    sleep "${delay}"
  done
}
# Usage: retry 3 5 curl -fsSL https://example.com
```

---

## Argument Parsing

### Simple positional args

```bash
usage() {
  cat <<EOF
Usage: $(basename "$0") [OPTIONS] <input_file>

Options:
  -o, --output <dir>   Output directory (default: ./out)
  -v, --verbose        Enable verbose output
  -h, --help           Show this help message
EOF
}

OUTPUT_DIR="./out"
VERBOSE=false

while [[ $# -gt 0 ]]; do
  case "$1" in
    -o|--output)  OUTPUT_DIR="$2"; shift 2 ;;
    -v|--verbose) VERBOSE=true; shift ;;
    -h|--help)    usage; exit 0 ;;
    --)           shift; break ;;
    -*)           echo "Unknown option: $1" >&2; usage; exit 1 ;;
    *)            INPUT_FILE="$1"; shift ;;
  esac
done

[[ -z "${INPUT_FILE:-}" ]] && { echo "Error: input_file required." >&2; usage; exit 1; }
```

---

## Linux vs macOS Portability

### Detection

```bash
OS="$(uname -s)"
case "${OS}" in
  Linux*)   PLATFORM=linux ;;
  Darwin*)  PLATFORM=macos ;;
  *)        echo "Unsupported OS: ${OS}" >&2; exit 1 ;;
esac
```

### Key GNU vs BSD differences

| Tool       | GNU (Linux)                        | BSD (macOS)                              | Portable alternative        |
| ---------- | ---------------------------------- | ---------------------------------------- | --------------------------- |
| `date`     | `date -d "2 days ago"`             | `date -v -2d`                            | `python3 -c "..."` or `gdate` |
| `sed`      | `sed -i 's/a/b/'`                  | `sed -i '' 's/a/b/'`                     | Use `perl -pi -e`           |
| `awk`      | mostly compatible                  | mostly compatible                        | —                           |
| `readlink` | `readlink -f`                      | not available natively                   | `realpath` or Python        |
| `mktemp`   | `mktemp -d /tmp/foo.XXXXXX`        | same syntax                              | —                           |
| `stat`     | `stat -c "%s" file`                | `stat -f "%z" file`                      | use `wc -c < file`          |
| `grep`     | `-P` for PCRE                      | no `-P` (use `grep -E` or `perl`)        | `grep -E`                   |
| `find`     | `find . -printf "%f\n"`            | no `-printf`                             | `basename` in a loop        |

### Portable `sed -i`

```bash
sed_inplace() {
  if sed --version 2>/dev/null | grep -q GNU; then
    sed -i "$@"
  else
    sed -i '' "$@"
  fi
}
```

### Portable `readlink -f` / realpath

```bash
realpath_compat() {
  if command -v realpath &>/dev/null; then
    realpath "$1"
  elif command -v python3 &>/dev/null; then
    python3 -c "import os,sys; print(os.path.realpath(sys.argv[1]))" "$1"
  else
    # POSIX fallback
    ( cd "$(dirname "$1")" && echo "$PWD/$(basename "$1")" )
  fi
}
```

---

## Dependency Checks

Always check for required tools at the start of the script:

```bash
require_cmd() {
  for cmd in "$@"; do
    command -v "${cmd}" &>/dev/null || {
      echo "[ERROR] Required command not found: ${cmd}" >&2
      exit 1
    }
  done
}
require_cmd curl jq rsync
```

---

## Logging and Output

> The format, levels, and color rules below implement the workspace-wide standard. See `.github/skills/cli-output/SKILL.md` for the canonical spec.

```bash
# Colors (only when stdout is a terminal)
if [[ -t 1 ]]; then
  RED='\033[0;31m'; YELLOW='\033[1;33m'; GREEN='\033[0;32m'; RESET='\033[0m'
else
  RED=''; YELLOW=''; GREEN=''; RESET=''
fi

log_info()  { echo -e "${GREEN}[INFO]${RESET}  $*"; }
log_warn()  { echo -e "${YELLOW}[WARN]${RESET}  $*" >&2; }
log_error() { echo -e "${RED}[ERROR]${RESET} $*" >&2; }
```

---

## Shellcheck Compliance

- Always quote variables: `"${VAR}"` not `$VAR`.
- Use `[[ ]]` for conditionals, not `[ ]`, in Bash.
- Avoid `ls` output parsing — use `glob` or `find` + `while read`.
- Avoid `cat file | cmd`; prefer `cmd < file` or process substitution.
- Word-split safe loops:
  ```bash
  while IFS= read -r line; do
    echo "${line}"
  done < file.txt
  ```
- Use `$(...)` not backticks for command substitution.
- Declare local variables in functions: `local var="value"`.

---

## Temp Files and Safe Cleanup

```bash
WORK_DIR=$(mktemp -d)
trap 'rm -rf "${WORK_DIR}"' EXIT

WORK_FILE=$(mktemp "${WORK_DIR}/output.XXXXXX")
```

Never write to `/tmp/fixed-name` — always use `mktemp` to avoid race conditions and symlink attacks.

---

## Common Patterns

### Confirm before destructive action

```bash
confirm() {
  read -r -p "${1} [y/N] " response
  [[ "${response}" =~ ^[Yy]$ ]]
}
confirm "Delete all files in ${TARGET_DIR}?" || exit 0
```

### Dry-run flag

```bash
DRY_RUN=false
run() {
  if "${DRY_RUN}"; then
    echo "[DRY-RUN] $*"
  else
    "$@"
  fi
}
# Usage: run rm -rf "${old_dir}"
```

### Parallel execution with job control

```bash
MAX_JOBS=4
for item in "${items[@]}"; do
  process_item "${item}" &
  while (( $(jobs -r | wc -l) >= MAX_JOBS )); do sleep 0.1; done
done
wait
```

---

## Security Checklist

- [ ] No hardcoded credentials, tokens, or secrets.
- [ ] All user-supplied input is validated and quoted before use in commands.
- [ ] No `eval` with external/user input.
- [ ] Temp files created with `mktemp`; cleaned up on EXIT trap.
- [ ] No `curl | bash` patterns without hash verification.
- [ ] `sudo`/privilege escalation only when explicitly required and documented.
- [ ] No unscoped glob patterns that could match outside the project directory.

---

## Behavioral Conventions

- Mark destructive operations with a `# DESTRUCTIVE` comment and a brief rationale.
- Use OS detection (`uname -s`) and feature detection (`command -v`) — never assume a package is installed.
- Prefer POSIX utilities for portability; document GNU extensions when used.
- Always test the script with `bash -n script.sh` (syntax check) before proposing execution.
- Recommend `shellcheck` for additional static analysis.
