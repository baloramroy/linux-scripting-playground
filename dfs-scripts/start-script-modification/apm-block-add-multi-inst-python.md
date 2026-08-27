# Start Script Modification

## APM Block Insertion Script using Python for Multi-Instances

```python
#!/usr/bin/env python3
"""
patch_apm.py
------------
Backs up and patches start scripts for one or more components with new APM
JAVA_OPTS.

- Comments out the OLD active APM block (if found, uncommented).
- Inserts the NEW APM block right after it.
- Skips (idempotent) if already patched.
- Dry-run by default; use --apply to actually write changes.

Add/remove components by editing the COMPONENTS list below -- nothing else
needs to change.

Usage:
    python3 patch_apm.py                      # dry run, all components
    python3 patch_apm.py --apply              # apply, all components
    python3 patch_apm.py --only davs,dfs      # limit to specific components
    python3 patch_apm.py --apply --only dfs
"""

import argparse
import difflib
import re
import shutil
import socket
import subprocess
from datetime import datetime
from pathlib import Path

# -----------------------------------------------------------------------
# Configuration -- ADD / REMOVE COMPONENTS HERE
# -----------------------------------------------------------------------
# Each entry:
#   name           -> label used in logs / --only filter
#   base_dir       -> home/bin dir containing the instance subfolders
#   instances      -> list of instance subfolder names (each has a "start" file)
#   service_prefix -> prefix added to instance name for -Delastic.apm.service_name
#                      (use "" for no prefix)
#
# To add a new component, just append a dict here.
# To remove one, delete/comment its entry.

COMPONENTS = [
    {
        "name": "davs",
        "base_dir": Path("/home/davs/bin"),
        "instances": ["davs1", "davs2", "davs3"],
        "service_prefix": "",
    },
    {
        "name": "dfs",
        "base_dir": Path("/home/dfs/bin"),
        "instances": ["dfs1", "dfs2", "dfs3"],
        "service_prefix": "",
    },
    {
        "name": "portal_davs",
        "base_dir": Path("/home/portal_davs/bin"),
        "instances": ["davs1", "davs2", "davs3"],
        "service_prefix": "portal-",
    },
    {
        "name": "portal_dfs",
        "base_dir": Path("/home/portal_dfs/bin"),
        "instances": ["dfs1", "dfs2", "dfs3"],
        "service_prefix": "portal-",
    },
    {
        # No instance subfolder -- start file sits directly in bin/
        "name": "extch",
        "base_dir": Path("/home/extch/bin"),
        "instances": [],
        "service_prefix": "",
    },
    {
        "name": "bkofc-summary",
        "base_dir": Path("/home/bkofc-summary/bin"),
        "instances": ["bkofc-summary1", "bkofc-summary2"],
        "service_prefix": "",
    },
]

# -----------------------------------------------------------------------
# OLD block: matched by PATTERN, not exact value, because the actual
# jar version / service_name / etc. differ per component. Each entry is a
# regex that must match a line (after stripping trailing whitespace/newline)
# for it to count as part of the old, active APM block. Order matters and
# the 4 lines must be CONTIGUOUS and UNCOMMENTED (not starting with '#').
# -----------------------------------------------------------------------
OLD_BLOCK_PATTERNS = [
    r'^JAVA_OPTS="\$JAVA_OPTS -javaagent:.*"$',
    r'^JAVA_OPTS="\$JAVA_OPTS -Delastic\.apm\.service_name=.*"$',
    r'^JAVA_OPTS="\$JAVA_OPTS -Delastic\.apm\.server_url=.*"$',
    r'^JAVA_OPTS="\$JAVA_OPTS -Delastic\.apm\.application_packages=.*"$',
]

NEW_BLOCK_MARKER = "#NEW APM"
SECRET_TOKEN = "a728140d801e585402ca98b4949eb75af69d1757abad0c6425f3619cde85a68e"
APM_SERVER_URL = "https://apm.mynagad.com:8200"
APM_AGENT_JAR = "/opt/apm/bin/elastic-apm-agent.jar"


def get_short_hostname() -> str:
    """dc3-dfs-prod-core-txnhist01 -> txnhist01"""
    full = socket.gethostname()
    return full.split('-')[-1]


def build_new_block(service_name: str, node_name: str) -> list[str]:
    return [
        NEW_BLOCK_MARKER,
        f'JAVA_OPTS="$JAVA_OPTS -javaagent:{APM_AGENT_JAR}"',
        f'JAVA_OPTS="$JAVA_OPTS -Delastic.apm.service_name={service_name}"',
        f'JAVA_OPTS="$JAVA_OPTS -Delastic.apm.service_node_name={node_name}"',
        f'JAVA_OPTS="$JAVA_OPTS -Delastic.apm.secret_token={SECRET_TOKEN}"',
        f'JAVA_OPTS="$JAVA_OPTS -Delastic.apm.server_url={APM_SERVER_URL}"',
        f'JAVA_OPTS="$JAVA_OPTS -Delastic.apm.environment=Production"',
        f'JAVA_OPTS="$JAVA_OPTS -Delastic.apm.application_packages=$APPLICATION_PACKAGE"',
    ]


def find_block_by_pattern(lines: list[str], patterns: list[str]):
    """
    Find `len(patterns)` ACTIVE (non-comment) lines that match `patterns` in
    order. Comment lines may be interspersed between them (e.g. legacy
    commented-out config sitting between active lines) -- those are skipped
    and left untouched. The match fails if any OTHER active line (one that
    doesn't match the next expected pattern) appears in between -- i.e. the
    matched lines must be contiguous among active lines, not among all lines.

    Returns a sorted list of original line indices that matched, or None if
    the full pattern sequence wasn't found starting from the first active
    match.
    """
    n = len(patterns)
    num_lines = len(lines)

    for start in range(num_lines):
        stripped_start = lines[start].strip()
        if stripped_start.startswith('#') or stripped_start == '':
            continue
        if not re.match(patterns[0], stripped_start):
            continue

        # Try to match the rest of the sequence starting here, skipping
        # over comment/blank lines, but bailing out if we hit an active
        # line that doesn't match the next expected pattern.
        matched_indices = [start]
        pattern_idx = 1
        j = start + 1
        while j < num_lines and pattern_idx < n:
            stripped = lines[j].strip()
            if stripped.startswith('#') or stripped == '':
                j += 1
                continue
            if re.match(patterns[pattern_idx], stripped):
                matched_indices.append(j)
                pattern_idx += 1
                j += 1
            else:
                # An unrelated active line breaks the sequence
                break

        if pattern_idx == n:
            return matched_indices

    return None


def already_patched(lines: list[str]) -> bool:
    return any(NEW_BLOCK_MARKER in l for l in lines)


def patch_lines(lines: list[str], service_name: str, node_name: str):
    """Returns (new_lines, changed: bool, reason: str)"""
    if already_patched(lines):
        return lines, False, "already patched (marker found), skipping"

    matched_indices = find_block_by_pattern(lines, OLD_BLOCK_PATTERNS)
    if matched_indices is None:
        return lines, False, "OLD BLOCK NOT FOUND (uncommented, matching pattern) - manual review needed"

    matched_set = set(matched_indices)
    last_idx = matched_indices[-1]
    new_block = build_new_block(service_name, node_name)
    new_block_lines = ['\n'] + [l + '\n' for l in new_block]

    new_lines = []
    for i, line in enumerate(lines):
        if i in matched_set:
            new_lines.append('#' + line.rstrip('\n') + '\n')
        else:
            new_lines.append(line)
        if i == last_idx:
            new_lines.extend(new_block_lines)

    return new_lines, True, "patched"


def syntax_ok(path: Path):
    result = subprocess.run(["bash", "-n", str(path)], capture_output=True, text=True)
    return result.returncode == 0, result.stderr


def process_instance(component_name: str, base_dir: Path, instance,
                      service_prefix: str, apply: bool, hostname_short: str):
    if instance is None:
        start_file = base_dir / "start"
        label = component_name
        instance_for_naming = component_name
    else:
        start_file = base_dir / instance / "start"
        label = f"{component_name}/{instance}"
        instance_for_naming = instance

    print("=" * 70)
    print(f"Component/Instance: {label}  ->  {start_file}")

    if not start_file.exists():
        print(f"[ERROR] File not found: {start_file}")
        return False

    service_name = f"{service_prefix}{instance_for_naming}"
    node_name = f"{hostname_short}-{service_name}"

    original_lines = start_file.read_text().splitlines(keepends=True)
    new_lines, changed, reason = patch_lines(original_lines, service_name, node_name)

    print(f"Status: {reason}")
    print(f"  service_name      = {service_name}")
    print(f"  service_node_name = {node_name}")

    if not changed:
        return False

    diff = difflib.unified_diff(
        original_lines, new_lines,
        fromfile=str(start_file), tofile=str(start_file) + " (patched)"
    )
    print("".join(diff))

    if not apply:
        print("[DRY RUN] No files written.")
        return True

    # 1. Backup
    today = datetime.now().strftime("%d%m%Y")
    backup_path = start_file.with_name(f"{start_file.name}.bkp.{today}")
    if backup_path.exists():
        print(f"[WARN] Backup already exists for today: {backup_path} (not overwritten)")
    else:
        shutil.copy2(start_file, backup_path)
        print(f"Backup created: {backup_path}")

    # 2. Write to temp file, validate, then replace
    tmp_path = start_file.with_suffix(".tmp_patch")
    tmp_path.write_text("".join(new_lines))

    ok, err = syntax_ok(tmp_path)
    if not ok:
        print(f"[ERROR] Syntax check failed, aborting for {label}:\n{err}")
        tmp_path.unlink(missing_ok=True)
        return False

    shutil.copystat(start_file, tmp_path)
    tmp_path.replace(start_file)
    print(f"[OK] {label} patched successfully.")
    return True


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--apply", action="store_true",
                         help="Actually apply changes (default: dry run)")
    parser.add_argument("--only", type=str, default=None,
                         help="Comma-separated component names to limit to "
                              "(e.g. --only davs,dfs). Default: all configured components.")
    args = parser.parse_args()

    selected = None
    if args.only:
        selected = {n.strip() for n in args.only.split(",") if n.strip()}

    components = [c for c in COMPONENTS if selected is None or c["name"] in selected]

    if not components:
        print("[ERROR] No matching components found. Check --only value against COMPONENTS names:")
        print("  " + ", ".join(c["name"] for c in COMPONENTS))
        return

    hostname_short = get_short_hostname()
    print(f"Detected hostname: {socket.gethostname()}  -> short: {hostname_short}")
    print(f"Components to process: {', '.join(c['name'] for c in components)}")
    if not args.apply:
        print("Running in DRY RUN mode. Use --apply to write changes.\n")

    results = {}
    for comp in components:
        instance_list = comp["instances"] if comp["instances"] else [None]
        for instance in instance_list:
            key = f"{comp['name']}/{instance}" if instance else comp["name"]
            results[key] = process_instance(
                comp["name"], comp["base_dir"], instance,
                comp["service_prefix"], args.apply, hostname_short
            )

    print("\n" + "=" * 70)
    print("SUMMARY")
    for key, ok in results.items():
        print(f"  {key}: {'CHANGED' if ok else 'NO CHANGE / SKIPPED'}")


if __name__ == "__main__":
    main()

```

---

## How to Run the Script

### 1. Save the script

For example:

```bash
vi patch_apm.py
```

Paste the final script and save it.

Verify:

```bash
ls -l patch_apm.py
```

---

### 2. Check Python

```bash
python3 --version
```

The script uses only Python standard-library modules, so **no `pip install` is required**.

---


## Useful Execution Examples

| Purpose             | Command                                        |
| ------------------- | ---------------------------------------------- |
| Dry run — all       | `python3 patch_apm.py`                         |
| Dry run — davs only | `python3 patch_apm.py --only davs`             |
| Dry run — multiple  | `python3 patch_apm.py --only davs,dfs`         |
| Apply — all         | `python3 patch_apm.py --apply`                 |
| Apply — davs only   | `python3 patch_apm.py --apply --only davs`     |
| Apply — multiple    | `python3 patch_apm.py --apply --only davs,dfs` |


---


## APM Startup Script Patcher — Architecture


```text
                              ┌──────────────────────────┐
                              │       patch_apm.py       │
                              │                          │
                              │  Dry-run by default     │
                              │  --apply → write changes│
                              └────────────┬─────────────┘
                                           │
                                           ▼
                              ┌──────────────────────────┐
                              │     Detect Hostname      │
                              │                          │
                              │ socket.gethostname()     │
                              │          ↓               │
                              │ extract short hostname   │
                              └────────────┬─────────────┘
                                           │
                                           ▼
                              ┌──────────────────────────┐
                              │    Read COMPONENTS       │
                              │                          │
                              │ davs                      │
                              │ dfs                       │
                              │ portal_davs               │
                              │ portal_dfs                │
                              │ extch                     │
                              │ bkofc-summary             │
                              └────────────┬─────────────┘
                                           │
                                           ▼
                              ┌──────────────────────────┐
                              │     --only specified?    │
                              └────────────┬─────────────┘
                                    YES    │    NO
                                     │     │
                                     ▼     ▼
                              Filter selected   Process all
                               components        components
                                     │             │
                                     └──────┬──────┘
                                            │
                                            ▼
                              ┌──────────────────────────┐
                              │  Process Each Component  │
                              │       / Instance         │
                              └────────────┬─────────────┘
                                           │
                                           ▼
                              ┌──────────────────────────┐
                              │ Determine start file     │
                              │                          │
                              │ Normal component:        │
                              │ /home/.../bin/INST/start │
                              │                          │
                              │ No-instance component:   │
                              │ /home/.../bin/start      │
                              └────────────┬─────────────┘
                                           │
                                           ▼
                              ┌──────────────────────────┐
                              │     File exists?         │
                              └────────────┬─────────────┘
                                    YES    │    NO
                                     │     │
                                     │     └──────────────► ERROR
                                     │                         │
                                     │                         ▼
                                     │                      SUMMARY
                                     │
                                     ▼
                              ┌──────────────────────────┐
                              │   Read start file        │
                              │   into lines[]           │
                              └────────────┬─────────────┘
                                           │
                                           ▼
                              ┌──────────────────────────┐
                              │     Already patched?     │
                              │                          │
                              │ Search for "#NEW APM"    │
                              └────────────┬─────────────┘
                                    YES    │    NO
                                     │     │
                                     ▼     ▼
                                   SKIP   Find OLD APM block
                                     │     using regex patterns
                                     │          │
                                     │          ▼
                                     │   ┌──────────────────────┐
                                     │   │ OLD BLOCK FOUND?     │
                                     │   └──────────┬───────────┘
                                     │        YES   │   NO
                                     │         │    │
                                     │         │    └──────────────►
                                     │         │                     MANUAL
                                     │         │                     REVIEW
                                     │         │                       │
                                     │         │                       ▼
                                     │         │                    SUMMARY
                                     │         │
                                     │         ▼
                                     │   Comment old APM
                                     │          +
                                     │   Build new APM block
                                     │          +
                                     │   Insert new block
                                     │          │
                                     │          ▼
                                     │   ┌─────────────────────┐
                                     │   │     Generate Diff   │
                                     │   │                     │
                                     │   │ Old APM → commented │
                                     │   │ New APM → inserted  │
                                     │   └──────────┬──────────┘
                                     │              │
                                     │              ▼
                                     │     ┌─────────────────┐
                                     │     │    --apply ?    │
                                     │     └────────┬────────┘
                                     │         NO   │   YES
                                     │          │   │
                                     │          │   ▼
                                     │          │  Create backup
                                     │          │       │
                                     │          │       ▼
                                     │          │  Write temporary
                                     │          │  .tmp_patch file
                                     │          │       │
                                     │          │       ▼
                                     │          │  Bash syntax check
                                     │          │      bash -n
                                     │          │       │
                                     │          │  ┌────┴────┐
                                     │          │ FAIL     PASS
                                     │          │  │         │
                                     │          │  ▼         ▼
                                     │          │ ABORT    Replace
                                     │          │ original  start
                                     │          │ file
                                     │          │
                                     │          ▼
                                     │       DRY RUN
                                     │       No file changed
                                     │
                                     └──────────────┐
                                                    │
                                                    ▼
                                         ┌────────────────────────┐
                                         │        SUMMARY         │
                                         │                        │
                                         │ davs/davs1: CHANGED    │
                                         │ davs/davs2: SKIPPED    │
                                         │ dfs/dfs1: CHANGED      │
                                         │ ...                    │
                                         └────────────────────────┘
```


---

## Component Structure

The script is designed around the `COMPONENTS` configuration.

```text
COMPONENTS
│
├── davs
│   ├── davs1
│   ├── davs2
│   └── davs3
│
├── dfs
│   ├── dfs1
│   ├── dfs2
│   └── dfs3
│
├── portal_davs
│   ├── davs1
│   ├── davs2
│   └── davs3
│
├── portal_dfs
│   ├── dfs1
│   ├── dfs2
│   └── dfs3
│
├── extch
│   └── start
│
└── bkofc-summary
    ├── bkofc-summary1
    └── bkofc-summary2
```

The important design is that **you only modify `COMPONENTS` when adding or removing applications**.

---

## APM Configuration Transformation

For every target `start` file, the script performs:

```text
OLD:

JAVA_OPTS="...old APM..."
JAVA_OPTS="...service_name..."
JAVA_OPTS="...server_url..."
JAVA_OPTS="...application_packages..."
```

becomes:

```text
#JAVA_OPTS="...old APM..."
#JAVA_OPTS="...service_name..."
#JAVA_OPTS="...server_url..."
#JAVA_OPTS="...application_packages..."

#NEW APM

JAVA_OPTS="...new APM..."
JAVA_OPTS="...service_name=davs1..."
JAVA_OPTS="...service_node_name=hostname-davs1..."
JAVA_OPTS="...secret_token..."
JAVA_OPTS="...server_url..."
JAVA_OPTS="...environment=Production..."
JAVA_OPTS="...application_packages..."
```


---

## Recommended Production Procedure

For production, I would follow this exact sequence:

```text
1. Copy patch_apm.py to server
          │
          ▼
2. Verify COMPONENTS configuration
          │
          ▼
3. Run dry run
   python3 patch_apm.py
          │
          ▼
4. Review ALL diffs
          │
          ▼
5. If necessary, test one component first
   python3 patch_apm.py --apply --only davs
          │
          ▼
6. Verify the patched start file
          │
          ▼
7. Apply remaining components
   python3 patch_apm.py --apply --only dfs,...
          │
          ▼
8. Check SUMMARY
          │
          ▼
9. Restart applications according to your normal
   application deployment/restart procedure
```

## Important distinction

The script **does not restart any application**.

Its responsibility ends at:

```text
START FILE PATCHED
       +
BACKUP CREATED
       +
SYNTAX VALIDATED
```

Application restart is a separate operational step.





