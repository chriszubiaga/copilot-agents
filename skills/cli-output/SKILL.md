---
name: cli-output
description: 'Consistent CLI output and logging conventions across all generated scripts. Defines log levels, label format, timestamps, and terminal colors for Python, Bash, and PowerShell. Load alongside any language skill when generating scripts or CLI tools.'
---

# CLI Output & Logging Conventions

## Purpose

All generated scripts — regardless of language — must produce output that looks and behaves the same way. This skill defines the **single source of truth** for log format, levels, colors, and messaging style.

---

## Standard Log Format

```
[YYYY-MM-DDTHH:MM:SS] [LEVEL]  message
```

- **Timestamp** — ISO 8601, local time, seconds precision.
- **Level label** — fixed-width 5 chars, uppercase: `DEBUG`, `INFO `, `WARN `, `ERROR`.
- **Separator** — two spaces after the closing bracket for readability.
- **Message** — free text; no trailing punctuation required.

### Examples

```
[2026-03-01T14:22:05] [INFO ] Processing 42 files in /tmp/work
[2026-03-01T14:22:06] [WARN ] Skipping unreadable file: data.csv
[2026-03-01T14:22:07] [ERROR] Connection refused on port 5432
[2026-03-01T14:22:07] [DEBUG] Retry attempt 2/3, delay=5s
```

---

## Log Levels

| Level   | When to use                                              | Stream   |
| ------- | -------------------------------------------------------- | -------- |
| `DEBUG` | Internal state, retry details, variable dumps            | stderr   |
| `INFO ` | Normal progress milestones, counts, paths                | stderr   |
| `WARN ` | Non-fatal anomalies; execution continues                 | stderr   |
| `ERROR` | Failures that stop or skip a unit of work                | stderr   |

All diagnostic output (every level) goes to **stderr**. Only actual data/results go to **stdout** — this keeps output pipeable.

---

## Terminal Colors

Apply colors **only when stderr is a TTY** (interactive terminal). Never colorize when redirected to a file or pipe.

| Level   | Color          | ANSI code   |
| ------- | -------------- | ----------- |
| `DEBUG` | Dim/gray       | `\033[2m`   |
| `INFO ` | Green          | `\033[32m`  |
| `WARN ` | Yellow         | `\033[33m`  |
| `ERROR` | Red (bold)     | `\033[1;31m`|
| Reset   | —              | `\033[0m`   |

---

## Python Implementation

```python
import logging
import sys

_COLORS = {
    "DEBUG":    "\033[2m",
    "INFO":     "\033[32m",
    "WARNING":  "\033[33m",
    "ERROR":    "\033[1;31m",
    "CRITICAL": "\033[1;31m",
}
_RESET = "\033[0m"
_USE_COLOR = sys.stderr.isatty()

_LEVEL_LABEL = {
    "DEBUG":    "DEBUG",
    "INFO":     "INFO ",
    "WARNING":  "WARN ",
    "ERROR":    "ERROR",
    "CRITICAL": "ERROR",
}

class _CliFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        label = _LEVEL_LABEL.get(record.levelname, record.levelname[:5].upper())
        ts = self.formatTime(record, "%Y-%m-%dT%H:%M:%S")
        msg = record.getMessage()
        line = f"[{ts}] [{label}]  {msg}"
        if _USE_COLOR:
            color = _COLORS.get(record.levelname, "")
            line = f"{color}{line}{_RESET}"
        return line

def configure_logging(verbose: bool = False) -> None:
    level = logging.DEBUG if verbose else logging.INFO
    handler = logging.StreamHandler(sys.stderr)
    handler.setFormatter(_CliFormatter())
    logging.basicConfig(level=level, handlers=[handler])

log = logging.getLogger(__name__)
```

Usage:
```python
configure_logging(verbose=args.verbose)
log.info("Processing %d files", count)
log.warning("Skipping: %s", path)
log.error("Failed: %s", exc)
log.debug("State: %r", obj)
```

---

## Bash Implementation

```bash
# ── CLI output helpers ──────────────────────────────────────────────────────
_TS()    { date '+%Y-%m-%dT%H:%M:%S'; }
_LOG()   {
  local level="$1"; shift
  local msg="$*"
  local line="[$(_TS)] [${level}]  ${msg}"
  if [[ -t 2 ]]; then
    local c; local r='\033[0m'
    case "${level}" in
      DEBUG) c='\033[2m'   ;;
      "INFO ") c='\033[32m'  ;;
      "WARN ") c='\033[33m'  ;;
      ERROR) c='\033[1;31m' ;;
    esac
    echo -e "${c}${line}${r}" >&2
  else
    echo "${line}" >&2
  fi
}

log_debug() { _LOG "DEBUG" "$*"; }
log_info()  { _LOG "INFO " "$*"; }
log_warn()  { _LOG "WARN " "$*"; }
log_error() { _LOG "ERROR" "$*"; }
```

Usage:
```bash
log_info  "Processing ${count} files in ${dir}"
log_warn  "Skipping unreadable file: ${f}"
log_error "Connection refused on port ${port}"
log_debug "Retry attempt ${i}/${attempts}, delay=${delay}s"
```

---

## PowerShell Implementation

```powershell
# ── CLI output helpers ──────────────────────────────────────────────────────
$Script:UseColor = [bool]([System.Console]::IsErrorRedirected -eq $false)

function _CliLog {
    param(
        [ValidateSet('DEBUG','INFO ','WARN ','ERROR')]
        [string]$Level,
        [string]$Message
    )
    $ts   = Get-Date -Format 'yyyy-MM-ddTHH:mm:ss'
    $line = "[$ts] [$Level]  $Message"
    if ($Script:UseColor) {
        $color = switch ($Level.Trim()) {
            'DEBUG' { 'DarkGray' }
            'INFO'  { 'Green'    }
            'WARN'  { 'Yellow'   }
            'ERROR' { 'Red'      }
        }
        [Console]::Error.WriteLine("")   # flush
        Write-Host $line -ForegroundColor $color -NoNewline
        [Console]::Error.WriteLine("")
    } else {
        [Console]::Error.WriteLine($line)
    }
}

function Log-Debug { param([string]$m) _CliLog -Level 'DEBUG' -Message $m }
function Log-Info  { param([string]$m) _CliLog -Level 'INFO ' -Message $m }
function Log-Warn  { param([string]$m) _CliLog -Level 'WARN ' -Message $m }
function Log-Error { param([string]$m) _CliLog -Level 'ERROR' -Message $m }
```

> **Note:** PowerShell's built-in `Write-Warning` / `Write-Error` streams are preferred for pipeline composability. Use `Log-*` helpers only in standalone scripts where unified formatting matters more than stream semantics.

Usage:
```powershell
Log-Info  "Processing $count files in $dir"
Log-Warn  "Skipping unreadable file: $f"
Log-Error "Connection refused on port $port"
Log-Debug "Retry attempt $i/$attempts, delay=${delay}s"
```

---

## Conventions

- **No `print()` / `Write-Host` / `echo` for diagnostics** — always use the log helpers above.
- **Actual output (data, results)** goes to stdout via `print()` / `Write-Output` / `echo`.
- **Progress indicators** (spinners, progress bars) are acceptable on stderr when stderr is a TTY and the tool is long-running.
- **Dry-run prefix** — prepend `[DRY-RUN]` to any log line that describes a skipped action:
  ```
  [2026-03-01T14:22:05] [INFO ]  [DRY-RUN] Would delete 14 files in /tmp/old
  ```
- **Error exit messages** — always log at `ERROR` level before exiting non-zero; include the exit code in the message when useful.
