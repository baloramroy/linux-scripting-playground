```python
#!/usr/bin/env python3
"""
patch_apm_single.py
--------------------
Backs up and patches start scripts for SINGLE-INSTANCE components -- i.e.
components where the start script lives directly at:

    /home/<component>/bin/start

(no per-instance subfolder like davs1/davs2/davs3).

This is a separate, standalone script from patch_apm.py (multi-instance).
Keep them separate so multi-instance components are never touched by this
script and vice versa.

- Comments out the OLD active APM block (matched by pattern, not exact
  value, since jar version / service_name value differ per component).
- Inserts the NEW APM block right after it, with a blank line in between.
- Skips (idempotent) if already patched.
- Dry-run by default; use --apply to actually write changes.
- Backup format: start.bkp.DDMMYYYY

Add/remove components by editing the COMPONENTS list below.

Usage:
    python3 patch_apm_single.py                       # dry run, all components
    python3 patch_apm_single.py --apply               # apply, all components
    python3 patch_apm_single.py --only cms,cp          # limit to specific components
    python3 patch_apm_single.py --apply --only spg
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
# Configuration -- ADD / REMOVE SINGLE-INSTANCE COMPONENTS HERE
# -----------------------------------------------------------------------
# Each entry:
#   name           -> component name; also the folder under /home and the
#                      value used for -Delastic.apm.service_name
#   base_dir       -> home/bin dir containing the "start" file directly
#   service_prefix -> prefix added to component name for service_name
#                      (use "" for no prefix)
#
# To add a new single-instance component, just append a dict here.
# To remove one, delete/comment its entry.

COMPONENTS = [
    {"name": "cms", "base_dir": Path("/home/cms/bin"), "service_prefix": ""},
    {"name": "cp", "base_dir": Path("/home/cp/bin"), "service_prefix": ""},
    {"name": "cs", "base_dir": Path("/home/cs/bin"), "service_prefix": ""},
    {"name": "davs", "base_dir": Path("/home/davs/bin"), "service_prefix": ""},
    {"name": "dfs", "base_dir": Path("/home/dfs/bin"), "service_prefix": ""},
    {"name": "drs", "base_dir": Path("/home/drs/bin"), "service_prefix": ""},
    {"name": "dmscore", "base_dir": Path("/home/dmscore/bin"), "service_prefix": ""},
    {"name": "extch", "base_dir": Path("/home/extch/bin"), "service_prefix": ""},
    {"name": "extchSMS", "base_dir": Path("/home/extchSMS/bin"), "service_prefix": ""},
    {"name": "ias", "base_dir": Path("/home/ias/bin"), "service_prefix": ""},
    #{"name": "kms", "base_dir": Path("/home/kms/bin"), "service_prefix": ""},
    {"name": "knotify", "base_dir": Path("/home/knotify/bin"), "service_prefix": ""},
    {"name": "kod", "base_dir": Path("/home/kod/bin"), "service_prefix": ""},
    {"name": "map", "base_dir": Path("/home/map/bin"), "service_prefix": ""},
    {"name": "pcs", "base_dir": Path("/home/pcs/bin"), "service_prefix": ""},
    {"name": "spg", "base_dir": Path("/home/spg/bin"), "service_prefix": ""},
    {"name": "tms", "base_dir": Path("/home/tms/bin"), "service_prefix": ""},
    {"name": "tsp", "base_dir": Path("/home/tsp/bin"), "service_prefix": ""},
    {"name": "bds", "base_dir": Path("/home/bds/bin"), "service_prefix": ""},
    {"name": "ecs", "base_dir": Path("/home/ecs/bin"), "service_prefix": ""},
    {"name": "rms", "base_dir": Path("/home/rms/bin"), "service_prefix": ""},
    {"name": "rpg", "base_dir": Path("/home/rpg/bin"), "service_prefix": ""},
    {"name": "mps", "base_dir": Path("/home/mps/bin"), "service_prefix": ""},
    {"name": "bkofc", "base_dir": Path("/home/bkofc/bin"), "service_prefix": ""},
    {"name": "utilityservice", "base_dir": Path("/home/utilityservice/bin"), "service_prefix": ""},
    {"name": "npsb_recon", "base_dir": Path("/home/npsb_recon/bin"), "service_prefix": ""},
    # NOTE: "davs", "dfs", "extch", "bkofc-summary" mutli instance are NOT here --
    # davs/dfs/portal_davs/portal_dfs are multi-instance (use patch_apm.py),
    # extch and bkofc-summary were already added to patch_apm.py earlier.
    # Add any further single-instance components below.
]

# -----------------------------------------------------------------------
# OLD block: matched by PATTERN, not exact value (jar version, service_name
# value, etc. differ per component). Order matters; comment lines may be
# interspersed between the active lines (they are skipped, not touched).
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
    order. Comment/blank lines may be interspersed between them (skipped,
    untouched). Fails if an unrelated active line breaks the sequence.
    Returns a list of matched line indices, or None if not found.
    """
    n = len(patterns)
    num_lines = len(lines)

    for start in range(num_lines):
        stripped_start = lines[start].strip()
        if stripped_start.startswith('#') or stripped_start == '':
            continue
        if not re.match(patterns[0], stripped_start):
            continue

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


def process_component(component_name: str, base_dir: Path,
                       service_prefix: str, apply: bool, hostname_short: str):
    start_file = base_dir / "start"
    print("=" * 70)
    print(f"Component: {component_name}  ->  {start_file}")

    if not start_file.exists():
        print(f"[ERROR] File not found: {start_file}")
        return False

    service_name = f"{service_prefix}{component_name}"
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
        print(f"[ERROR] Syntax check failed, aborting for {component_name}:\n{err}")
        tmp_path.unlink(missing_ok=True)
        return False

    shutil.copystat(start_file, tmp_path)
    tmp_path.replace(start_file)
    print(f"[OK] {component_name} patched successfully.")
    return True


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--apply", action="store_true",
                         help="Actually apply changes (default: dry run)")
    parser.add_argument("--only", type=str, default=None,
                         help="Comma-separated component names to limit to "
                              "(e.g. --only cms,cp). Default: all configured components.")
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
        results[comp["name"]] = process_component(
            comp["name"], comp["base_dir"], comp["service_prefix"],
            args.apply, hostname_short
        )

    print("\n" + "=" * 70)
    print("SUMMARY")
    for name, ok in results.items():
        print(f"  {name}: {'CHANGED' if ok else 'NO CHANGE / SKIPPED'}")


if __name__ == "__main__":
    main()

```