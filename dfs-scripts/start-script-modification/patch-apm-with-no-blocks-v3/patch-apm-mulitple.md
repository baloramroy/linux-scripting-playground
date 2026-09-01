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
import os
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


def find_commented_old_block(lines: list[str], patterns: list[str]):
    """
    Find a contiguous run of lines that are ALREADY commented out (start with
    '#') but whose un-commented text matches `patterns` in order. This
    detects a case where someone manually commented out the entire old APM
    block already. Used only as an insertion anchor (to keep the new block
    visually grouped with the old one) -- these lines are NOT touched/
    re-commented, since they're already inactive.

    Returns a list of matched line indices (contiguous), or None if not found.
    """
    n = len(patterns)
    num_lines = len(lines)

    for start in range(num_lines - n + 1):
        window = [lines[i].strip() for i in range(start, start + n)]
        ok = True
        for stripped, pattern in zip(window, patterns):
            if not stripped.startswith('#'):
                ok = False
                break
            uncommented = stripped.lstrip('#').strip()
            if not re.match(pattern, uncommented):
                ok = False
                break
        if ok:
            return list(range(start, start + n))

    return None


def find_last_java_opts_line(lines: list[str]):
    """
    Find the index of the LAST active (non-comment) line that is a
    JAVA_OPTS="$JAVA_OPTS ..." assignment. Used as the insertion anchor
    when no existing (old-style) APM block is found in the file at all.
    Returns the index, or None if no such line exists anywhere in the file.
    """
    pattern = r'^JAVA_OPTS="\$JAVA_OPTS\b.*"$'
    last_idx = None
    for i, line in enumerate(lines):
        stripped = line.strip()
        if stripped.startswith('#') or stripped == '':
            continue
        if re.match(pattern, stripped):
            last_idx = i
    return last_idx


def already_patched(lines: list[str]) -> bool:
    """
    Detects the new-block marker robustly, tolerating formatting variations
    like '#NEW APM', '# NEW APM', '#new apm', extra spaces, etc. -- so a
    hand-edited or differently-spaced marker still counts as already patched
    and the script never double-inserts the block.
    """
    marker_normalized = re.sub(r'\s+', '', NEW_BLOCK_MARKER).lower()
    for l in lines:
        stripped = l.strip()
        if stripped.startswith('#'):
            normalized = re.sub(r'\s+', '', stripped).lower()
            if normalized == marker_normalized:
                return True
    return False


def patch_lines(lines: list[str], service_name: str, node_name: str):
    """
    Returns (new_lines, changed, reason, matched_old_lines, new_block_lines)
    matched_old_lines / new_block_lines are plain strings (no trailing \n)
    for clean logging purposes. matched_old_lines is [] for a fresh insert
    (no old block existed).
    """
    if already_patched(lines):
        return lines, False, "already patched (marker found), skipping", [], []

    matched_indices = find_block_by_pattern(lines, OLD_BLOCK_PATTERNS)

    if matched_indices is not None:
        # Existing old block found -- comment it out, insert new block after it
        matched_set = set(matched_indices)
        last_idx = matched_indices[-1]
        matched_old_lines = [lines[i].rstrip('\n') for i in matched_indices]

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

        return new_lines, True, "patched", matched_old_lines, new_block

    # No ACTIVE old block found. Check if an old block exists but is already
    # fully commented out (e.g. someone manually disabled it before) --
    # if so, insert the new block right after it to keep them grouped,
    # rather than falling through to a possibly-distant JAVA_OPTS line.
    commented_old_indices = find_commented_old_block(lines, OLD_BLOCK_PATTERNS)
    if commented_old_indices is not None:
        anchor_idx = commented_old_indices[-1]
        anchor_line = lines[anchor_idx].rstrip('\n')
        new_block = build_new_block(service_name, node_name)
        new_block_lines = ['\n'] + [l + '\n' for l in new_block]

        new_lines = list(lines[:anchor_idx + 1]) + new_block_lines + list(lines[anchor_idx + 1:])

        return new_lines, True, "patched_after_commented_block", \
            [f"(old block was already commented out -- inserted after: {anchor_line})"], new_block

    # No old block at all (active or commented) -- fresh insert after the
    # last active JAVA_OPTS= line found anywhere in the file.
    anchor_idx = find_last_java_opts_line(lines)
    if anchor_idx is None:
        return lines, False, "NO_ANCHOR_FOUND - no JAVA_OPTS= line in file, manual review needed", [], []

    anchor_line = lines[anchor_idx].rstrip('\n')
    new_block = build_new_block(service_name, node_name)
    new_block_lines = ['\n'] + [l + '\n' for l in new_block]

    new_lines = list(lines[:anchor_idx + 1]) + new_block_lines + list(lines[anchor_idx + 1:])

    return new_lines, True, "patched_fresh", [f"(no old block -- inserted after: {anchor_line})"], new_block


def syntax_ok(path: Path):
    result = subprocess.run(["bash", "-n", str(path)], capture_output=True, text=True)
    return result.returncode == 0, result.stderr


def verify_patch(path: Path):
    """
    Re-read the file from disk after writing and confirm:
      1. NEW_BLOCK_MARKER is present.
      2. The old block pattern is NOT found as an active (uncommented) block
         anymore (it should now be commented out).
    Returns (ok: bool, detail: str)
    """
    lines = path.read_text().splitlines(keepends=True)

    has_marker = already_patched(lines)
    old_still_active = find_block_by_pattern(lines, OLD_BLOCK_PATTERNS) is not None

    if has_marker and not old_still_active:
        return True, "verified: new block present, old block commented"
    if not has_marker:
        return False, "verification FAILED: new block marker not found after write"
    if old_still_active:
        return False, "verification FAILED: old block still active (uncommented) after write"
    return False, "verification FAILED: unknown state"


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
        return "FILE_NOT_FOUND"

    service_name = f"{service_prefix}{instance_for_naming}"
    node_name = f"{hostname_short}-{service_name}"

    original_lines = start_file.read_text().splitlines(keepends=True)
    new_lines, changed, reason, matched_old_lines, new_block = patch_lines(
        original_lines, service_name, node_name
    )

    print(f"Status: {reason}")
    print(f"  service_name      = {service_name}")
    print(f"  service_node_name = {node_name}")

    if not changed:
        if "already patched" in reason:
            return "ALREADY_PATCHED"
        if "NO_ANCHOR_FOUND" in reason:
            return "NO_ANCHOR_FOUND"
        return "SKIPPED_NOT_FOUND"

    print()
    if reason == "patched_fresh":
        print("  No existing APM block found in this file.")
        print(f"  {matched_old_lines[0]}")
    elif reason == "patched_after_commented_block":
        print("  Old APM block was already commented out (left as-is).")
        print(f"  {matched_old_lines[0]}")
    else:
        print("  Lines being COMMENTED OUT (old block):")
        for line in matched_old_lines:
            print(f"    - {line}")
    print()
    print("  Lines being INSERTED (new block):")
    for line in new_block:
        print(f"    + {line}")
    print()

    if not apply:
        print("[DRY RUN] No files written. (Not verified -- re-run with --apply to verify.)")
        if reason == "patched_fresh":
            return "WOULD_CHANGE_FRESH_DRY_RUN"
        elif reason == "patched_after_commented_block":
            return "WOULD_CHANGE_AFTER_COMMENTED_DRY_RUN"
        else:
            return "WOULD_CHANGE_DRY_RUN"

    # 1. Capture original ownership (script may run as root, which would
    #    otherwise silently reset owner/group to root:root on any file it
    #    creates -- shutil.copystat does NOT preserve ownership, only mode
    #    and timestamps).
    orig_stat = start_file.stat()
    orig_uid, orig_gid = orig_stat.st_uid, orig_stat.st_gid

    # 2. Backup
    today = datetime.now().strftime("%d%m%Y")
    backup_path = start_file.with_name(f"{start_file.name}.bkp.{today}")
    if backup_path.exists():
        print(f"[WARN] Backup already exists for today: {backup_path} (not overwritten)")
    else:
        shutil.copy2(start_file, backup_path)
        os.chown(backup_path, orig_uid, orig_gid)
        print(f"Backup created: {backup_path} (owner preserved: uid={orig_uid}, gid={orig_gid})")

    # 3. Write to temp file, validate, then replace
    tmp_path = start_file.with_suffix(".tmp_patch")
    tmp_path.write_text("".join(new_lines))

    ok, err = syntax_ok(tmp_path)
    if not ok:
        print(f"[ERROR] Syntax check failed, aborting for {label}:\n{err}")
        tmp_path.unlink(missing_ok=True)
        return "SYNTAX_CHECK_FAILED"

    shutil.copystat(start_file, tmp_path)
    os.chown(tmp_path, orig_uid, orig_gid)
    tmp_path.replace(start_file)

    # 3. Verify on-disk result
    verified, detail = verify_patch(start_file)
    is_fresh = reason in ("patched_fresh", "patched_after_commented_block")
    if verified:
        print(f"[OK] {label} patched successfully. {detail}")
        return "PATCHED_FRESH_VERIFIED" if is_fresh else "PATCHED_VERIFIED"
    else:
        print(f"[ERROR] {label} written but {detail}")
        return "PATCHED_VERIFY_FAILED"


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

    STATUS_LABELS = {
        "PATCHED_VERIFIED": "CHANGED - VERIFIED OK",
        "PATCHED_FRESH_VERIFIED": "CHANGED - NEW APM BLOCK ADDED - VERIFIED OK",
        "PATCHED_VERIFY_FAILED": "CHANGED - VERIFICATION FAILED (check manually!)",
        "SYNTAX_CHECK_FAILED": "NOT CHANGED - syntax check failed, aborted",
        "WOULD_CHANGE_DRY_RUN": "WOULD CHANGE (dry run - not verified yet)",
        "WOULD_CHANGE_FRESH_DRY_RUN": "WOULD ADD NEW APM BLOCK - no old block existed at all (dry run - not verified yet)",
        "WOULD_CHANGE_AFTER_COMMENTED_DRY_RUN": "WOULD ADD NEW APM BLOCK - after already-commented old block (dry run - not verified yet)",
        "ALREADY_PATCHED": "NO CHANGE - already patched",
        "SKIPPED_NOT_FOUND": "NO CHANGE - old block not found (manual review needed)",
        "NO_ANCHOR_FOUND": "NO CHANGE - no JAVA_OPTS line found at all (manual review needed)",
        "FILE_NOT_FOUND": "NOT CHANGED - start file not found",
    }

    VERIFIED_STATUSES = ("PATCHED_VERIFIED", "PATCHED_FRESH_VERIFIED")
    CHANGED_STATUSES = VERIFIED_STATUSES + ("PATCHED_VERIFY_FAILED",)
    FAILED_STATUSES = ("PATCHED_VERIFY_FAILED", "SYNTAX_CHECK_FAILED", "FILE_NOT_FOUND")

    print("\n" + "=" * 70)
    print("SUMMARY")
    changed_count = 0
    verified_count = 0
    failed_count = 0
    for key, status in results.items():
        label = STATUS_LABELS.get(status, status)
        marker = "✔" if status in VERIFIED_STATUSES else (
            "✖" if status in FAILED_STATUSES else "-"
        )
        print(f"  {marker} {key}: {label}")
        if status in CHANGED_STATUSES:
            changed_count += 1
        if status in VERIFIED_STATUSES:
            verified_count += 1
        if status in FAILED_STATUSES:
            failed_count += 1

    print("-" * 70)
    print(f"Total processed : {len(results)}")
    print(f"Changed         : {changed_count}")
    print(f"Verified OK     : {verified_count}")
    print(f"Failed/Attention: {failed_count}")


if __name__ == "__main__":
    main()
```