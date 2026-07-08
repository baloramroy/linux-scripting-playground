```bash
#!/bin/bash
set -euo pipefail

#--------------------------------------------------
# Configuration
#--------------------------------------------------

#SRC_DIR="/home/knotifydmzpush/log/archive"
SRC_DIR="/home/apigw/log/archive"
#DEST_DIR="/LOGS/knotifypush-dmz/knotifypush-dmz1"
DEST_DIR="/LOGS/app7/apigw"
COMPONENT="apigw-nagad-app7"
DAYS=3
LOCK_FILE="/tmp/${COMPONENT}_archive.lock"

#--------------------------------------------------

exec 200>"$LOCK_FILE"
flock -n 200 || {
    echo "Another instance is running."
    exit 1
}

#--------------------------------------------------
# Validate source directory
#--------------------------------------------------
if [[ ! -d "$SRC_DIR" ]]; then
    echo "ERROR: Source directory not found: $SRC_DIR"
    exit 1
fi

echo "Source directory found."
cd "$SRC_DIR"

#--------------------------------------------------
# Ensure destination exists
#--------------------------------------------------
if [[ ! -d "$DEST_DIR" ]]; then
    echo "Destination directory does not exist. Creating..."
    mkdir -p "$DEST_DIR"
else
    echo "Destination directory exists."
fi

echo
echo "Searching logs using filename date (older than or equal to $DAYS days)..."

#--------------------------------------------------
# Calculate cutoff date
#--------------------------------------------------
CUTOFF_DATE=$(date -d "$DAYS days ago" +%F)

#--------------------------------------------------
# Group files by date
#--------------------------------------------------
declare -A FILE_GROUPS

shopt -s nullglob

for file in "${COMPONENT}"-INST_*-*.log.gz; do

    # Extract YYYY-MM-DD from filename
    if [[ $file =~ ([0-9]{4}-[0-9]{2}-[0-9]{2}) ]]; then
        FILE_DATE="${BASH_REMATCH[1]}"

        # YYYY-MM-DD allows lexicographical comparison
        if [[ "$FILE_DATE" < "$CUTOFF_DATE" || "$FILE_DATE" == "$CUTOFF_DATE" ]]; then
            
            # Store filenames separated by newline
            FILE_GROUPS["$FILE_DATE"]+="$file"$'\n'

        fi
    fi
done

#--------------------------------------------------

if [[ ${#FILE_GROUPS[@]} -eq 0 ]]; then
    echo "No logs found older than or equal to $DAYS days."
    exit 0
fi

#--------------------------------------------------
# Process each date
#--------------------------------------------------
for DATE in $(printf '%s\n' "${!FILE_GROUPS[@]}" | sort); do

    ARCHIVE_NAME="${COMPONENT}-${DATE}.tar.gz"

    echo
    echo "=================================================="
    echo "Processing Date : $DATE"
    echo "Archive         : $ARCHIVE_NAME"
    echo "=================================================="

    #----------------------------------------------
    # Skip if archive already exists in destination
    #----------------------------------------------
    if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then
        echo "Archive already exists in destination."
        echo "Skipping..."
        continue
    fi

    #----------------------------------------------
    # Read files for this date
    #----------------------------------------------

    mapfile -t FILES <<< "${FILE_GROUPS[$DATE]}"

    #----------------------------------------------
    # Recovery case
    #----------------------------------------------
    if [[ -f "$ARCHIVE_NAME" ]]; then
        echo "Archive already exists in source."
        echo "Moving archive to destination..."

        mv -f "$ARCHIVE_NAME" "$DEST_DIR/"
        chmod 777 "$DEST_DIR/$ARCHIVE_NAME"

        echo "Removing source log files..."
        rm -f "${FILES[@]}"

        echo "Recovery completed."
        continue
    fi

    #----------------------------------------------
    # Files exist or not for this date
    #----------------------------------------------

    if [[ ${#FILES[@]} -eq 0 ]]; then
        echo "No files found for $DATE"
        continue
    fi

    echo "Files to archive:"
    printf '  %s\n' "${FILES[@]}"

    #----------------------------------------------
    # Create archive
    #----------------------------------------------
    echo
    echo "Creating archive..."

    tar -czf "$ARCHIVE_NAME" "${FILES[@]}"

    echo "Archive created."

    #----------------------------------------------
    # Move archive
    #----------------------------------------------
    mv -f "$ARCHIVE_NAME" "$DEST_DIR/"

    if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then

        chmod 777 "$DEST_DIR/$ARCHIVE_NAME"

        echo "Archive moved successfully."

        echo "Deleting source log files..."
        rm -f "${FILES[@]}"

        echo "Completed for $DATE"

    else
        echo "ERROR: Failed to move archive to destination."
        exit 1
    fi

done

echo
echo "========================================"
echo "All eligible logs processed successfully."
echo "========================================"

```