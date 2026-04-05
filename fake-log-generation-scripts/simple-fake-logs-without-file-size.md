# Simple Script to Generate Fake Logs with Empty File Size.

## Script: `simple_fake_logs.sh`
```bash
#!/bin/bash
set -euo pipefail

COMPONENT="${1:-extch}"
HOSTNAME_VAR="${2:-nagad-app6}"
INSTANCE="${3:-INST_1}"
LOG_DATE="${4:-2026-03-30}"
OUTPUT_DIR="${5:-./fake_logs}"

mkdir -p "$OUTPUT_DIR"

for HOUR in {00..23}
do
    FILE_NAME="${COMPONENT}-${HOSTNAME_VAR}-${INSTANCE}-${LOG_DATE}-${HOUR}-0.log.gz"
    FILE_PATH="${OUTPUT_DIR}/${FILE_NAME}"
    touch "$FILE_PATH"

    # Set modification time based on LOG_DATE and HOUR
    touch -t "$(date -d "${LOG_DATE} ${HOUR}:00:00" +%Y%m%d%H%M.%S)" "$FILE_PATH"
done

echo "Fake log files created."
```

---

## How to Use

**1. Make executable**

```bash
chmod +x generate_fake_logs.sh
```

**2. Run with defaults**

```bash
./generate_fake_logs.sh
```

**3. Run with custom values**

```bash
./generate_fake_logs.sh extch nagad-app6 INST_2 2026-04-01 /tmp/logs
```

**Arguments order:**

```
$1 → component
$2 → hostname
$3 → instance
$4 → date (YYYY-MM-DD)
$5 → output directory
```

---

## Output Example

```
extch-nagad-app6-INST_1-2026-03-30-00-0.log.gz
extch-nagad-app6-INST_1-2026-03-30-01-0.log.gz
...
extch-nagad-app6-INST_1-2026-03-30-23-0.log.gz
```

---

## Important Warning
If later someone runs:
```bash
gunzip file.log.gz
```
> It will fail because it is not actually compressed file.

---
