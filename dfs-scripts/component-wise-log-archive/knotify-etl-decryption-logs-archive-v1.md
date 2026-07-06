## Knotify ETL Decryption Log Archive Srcipts

```bash
#!/bin/bash
set -euo pipefail

SRC_DIR="/home/knotify/etlfordecrypt/log/archive"
DEST_DIR="/LOGS/app4/knotify/etlfordecrypt"
COMPONENT="decryption-etl"
LOCK_FILE="/tmp/knotify_etl_decryption_archive.lock"

exec 200>"$LOCK_FILE"
flock -n 200 || { echo "Another instance is running."; exit 1; }

# Validate source directory
if [ ! -d "$SRC_DIR" ]; then
    echo "ERROR: Source directory not found: $SRC_DIR"
    exit 1
fi

cd "$SRC_DIR"

YESTERDAY=$(date -d "yesterday" +%Y-%m-%d)
#YESTERDAY="2026-01-02"
ARCHIVE_NAME="${COMPONENT}-${YESTERDAY}.tar.gz"

echo "Processing date: $YESTERDAY"

# 1️.If archive already exists in destination → exit safely
if [ -f "$DEST_DIR/$ARCHIVE_NAME" ]; then
    echo "Archive already exists in destination. Nothing to do."
    exit 0
fi

# 2️.Recovery case: archive exists in source (previous crash)
if [ -f "$ARCHIVE_NAME" ]; then
    echo "Archive exists in source. Recovering by moving it to destination..."
    mkdir -p "$DEST_DIR"
    mv -f "$ARCHIVE_NAME" "$DEST_DIR/"

    if [ -f "$DEST_DIR/$ARCHIVE_NAME" ]; then
        echo "Recovery move successful."
    else
        echo "ERROR: Recovery move failed!"
        exit 1
    fi
fi

# Enable safe glob handling
shopt -s nullglob
FILES=(${COMPONENT}-${YESTERDAY}-*.log)

if [ ${#FILES[@]} -eq 0 ]; then
    echo "No hourly log files found for $YESTERDAY"
    exit 0
fi

# 3️.Normal archive creation flow
echo "Creating archive: $ARCHIVE_NAME"

tar -czf "$ARCHIVE_NAME" "${FILES[@]}"
echo "Archive created successfully."

mkdir -p "$DEST_DIR"
mv -f "$ARCHIVE_NAME" "$DEST_DIR/"

if [ -f "$DEST_DIR/$ARCHIVE_NAME" ]; then
    echo "Archive moved successfully."
    echo "Deleting hourly log files..."
    rm -f "${FILES[@]}"
else
    echo "ERROR: Archive move failed!"
    exit 1
fi

echo "Done."
echo "All eligible logs processed successfully."
```

## Set up Cron Job

- Open cronjob by this command
	```bash
	crontab -e
	```

- Insert below entry
    ```bash
    00 22 * * * /home/scripts/knotify_etl_decryption_log_move.sh > /home/scripts/etl_decryption_log_move_output.log
    ```

