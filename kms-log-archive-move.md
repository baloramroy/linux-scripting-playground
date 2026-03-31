```bash
#!/bin/bash
set -euo pipefail

SRC_DIR="/home/kms/bin/log"
DEST_DIR="/LOGS/app6/kms/kmslog"
COMPONENT="kms"
DAYS=3
LOCK_FILE="/tmp/kms_archive.lock"

exec 200>"$LOCK_FILE"
flock -n 200 || { echo "Another instance is running."; exit 1; }

# Validate source directory
if [ ! -d "$SRC_DIR" ]; then
    echo "ERROR: Source directory not found: $SRC_DIR"
    exit 1
fi

cd "$SRC_DIR"

#Create destination if it is doesnot exist
mkdir -p "$DEST_DIR"

echo "Archiving logs older than $DAYS days..."

# Enable safe glob handling
shopt -s nullglob

# Collect unique dates older than N days
mapfile -t DATES < <(
    find . -maxdepth 1 -type f \
        -name "${COMPONENT}-*.log.gz" \
        -mtime +"$DAYS" \
    | awk -F'[-]' '{print $2"-"$3"-"$4}' \
    | sort -u
)

if [ ${#DATES[@]} -eq 0 ]; then
    echo "No logs older than $DAYS days found."
    exit 0
fi

# --------------------------------------------------
# Process each date
# --------------------------------------------------
for DATE in "${DATES[@]}"; do

    ARCHIVE_NAME="${COMPONENT}-${DATE}.tar.gz"

    echo ""
    echo "Processing date: $DATE"

    # 1️⃣ If archive already exists in destination → skip safely
    if [ -f "$DEST_DIR/$ARCHIVE_NAME" ]; then
        echo "Archive already exists in destination. Skipping."
        
        continue
    fi

    # 2️⃣ Recovery case: archive exists in source
    if [ -f "$ARCHIVE_NAME" ]; then
        echo "Archive exists in source. Recovering by moving it to destination..."
        mv -f "$ARCHIVE_NAME" "$DEST_DIR/"
        echo "Recovery move successful."
        rm -f ${COMPONENT}-${DATE}-*.log.gz
        echo "Source Log Remove Successful."
        continue
    fi

    FILES=(${COMPONENT}-${DATE}-*.log.gz)

    if [ ${#FILES[@]} -eq 0 ]; then
        echo "No hourly log files found for $DATE"
        continue
    fi

    # 3️⃣ Normal archive creation flow
    echo "Creating archive: $ARCHIVE_NAME"
    tar -czf "$ARCHIVE_NAME" "${FILES[@]}"
    echo "Archive created successfully."

    mv -f "$ARCHIVE_NAME" "$DEST_DIR/"
    if [ -f "$DEST_DIR/$ARCHIVE_NAME" ]; then
        echo "Archive moved successfully"
        echo "Deleting hourly log files.."
        rm -f "${FILES[@]}"
    else
        echo "ERROR: Archive move failed!" 
        exit 1
    fi
done

echo ""
echo "All eligible logs processed successfully."
```
