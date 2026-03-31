
# Generate Fake Log with Actual File Size
## Script: generate_fake_logs.sh

```
#!/bin/bash
set -euo pipefail

#########################################
# CONFIGURABLE VARIABLES
#########################################

COMPONENT="${1:-extch}"
HOSTNAME_VAR="${2:-nagad-app6}"
INSTANCE="${3:-INST_1}"
LOG_DATE="${4:-2026-03-30}"      # Format: YYYY-MM-DD
OUTPUT_DIR="${5:-./fake_logs}"   # Default output directory
MIN_SIZE_KB="${6:-100}"          # Minimum log size in KB
MAX_SIZE_KB="${7:-2500}"         # Maximum log size in KB

#########################################

mkdir -p "$OUTPUT_DIR"

echo "Generating fake logs..."
echo "Component : $COMPONENT"
echo "Hostname  : $HOSTNAME_VAR"
echo "Instance  : $INSTANCE"
echo "Date      : $LOG_DATE"
echo "Output Dir: $OUTPUT_DIR"
echo "-------------------------------------"

for HOUR in $(seq -w 0 23)
do
    FILE_NAME="${COMPONENT}-${HOSTNAME_VAR}-${INSTANCE}-${LOG_DATE}-${HOUR}-0.log"
    FILE_PATH="${OUTPUT_DIR}/${FILE_NAME}"

    # Generate random size between MIN_SIZE_KB and MAX_SIZE_KB
    SIZE_KB=$(( RANDOM % (MAX_SIZE_KB - MIN_SIZE_KB + 1) + MIN_SIZE_KB ))

    # Create fake content
    head -c "${SIZE_KB}K" /dev/urandom | base64 | head -c "${SIZE_KB}K" > "$FILE_PATH"

    # Compress
    gzip -f "$FILE_PATH"

    echo "Created: ${FILE_NAME}.gz (${SIZE_KB} KB)"
done

echo "-------------------------------------"
echo "Fake log generation completed."
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
./generate_fake_logs.sh extch nagad-app6 INST_2 2026-04-01 /tmp/logs 200 3000
```

**Arguments order:**

```
$1 → component
$2 → hostname
$3 → instance
$4 → date (YYYY-MM-DD)
$5 → output directory
$6 → min size KB
$7 → max size KB
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

