## DRS log archive and move to destination

```bash

#!/bin/bash
set -euo pipefail

SRC_DIR="/home/drs/log/archive"
DEST_DIR="/LOGS/app5/drs"
COMPONENT="drs"
HOSTNAME="nagad-app5"
INSTANCE="INST_1"
DAYS=3
LOCK_FILE="/tmp/drs_archive.lock"

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
        -name "${COMPONENT}-*.log" \
        -mtime +"$DAYS" \
    | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}' \
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

    ARCHIVE_NAME="${COMPONENT}-${HOSTNAME}-${DATE}.tar.gz"

    echo ""
    echo "Processing date: $DATE"

    # 1️ If archive already exists in destination → skip safely
    if [ -f "$DEST_DIR/$ARCHIVE_NAME" ]; then
        echo "Archive already exists in destination. Skipping."
        continue
    fi

    # 2️ Recovery case: archive exists in source
    if [ -f "$ARCHIVE_NAME" ]; then
        echo "Archive exists in source. Recovering by moving it to destination..."
        mv -f "$ARCHIVE_NAME" "$DEST_DIR/"
        echo "Archive in the source, successfully move to the destination."
	
		# Change permissions to 777 on the moved archive
		chmod 777 "$DEST_DIR/$ARCHIVE_NAME"
		echo "Permission changed to 777 on $DEST_DIR/$ARCHIVE_NAME"
        
		rm -f "${COMPONENT}-${HOSTNAME}-${INSTANCE}-${DATE}"-*.log
        echo "Log Remove Successfuly from the Source directory."
        
	continue
    fi

    FILES=(${COMPONENT}-${HOSTNAME}-${INSTANCE}-${DATE}-*.log)

    if [ ${#FILES[@]} -eq 0 ]; then
        echo "No hourly log files found for $DATE"
        continue
    else
        echo "Hourly logs found. Proceeding to create archive."
    fi

    # 3️ Normal archive creation flow
    echo "Creating archive: $ARCHIVE_NAME"
    tar -czf "$ARCHIVE_NAME" "${FILES[@]}"
    echo "Archive created successfully."

    mv -f "$ARCHIVE_NAME" "$DEST_DIR/"
    if [ -f "$DEST_DIR/$ARCHIVE_NAME" ]; then
        echo "Archive moved successfully"
	
		# Change permissions to 777 on the moved archive
		chmod 777 "$DEST_DIR/$ARCHIVE_NAME"
		echo "Permission changed to 777 on $DEST_DIR/$ARCHIVE_NAME"

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

## Set up Cron Job

- Open cronjob by this command
	```bash
	crontab -e
	```

- Insert below entry
  ```bash
  30 02 * * * /home/scripts/drs_log_move.sh > /home/scripts/log/drslog/drs-archive.log 2>&1
  ```
