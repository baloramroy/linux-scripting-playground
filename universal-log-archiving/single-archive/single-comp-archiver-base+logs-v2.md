```bash
#!/bin/bash
set -euo pipefail

#--------------------------------------------------
# Configuration
#--------------------------------------------------

DAYS=3
COMPONENT="png-ngd-dc2-npsb-konaswitch01"

SRC_DIR="/root/single-archiver/png"
DEST_DIR="/root/single-archiver/LOGS/png"

LOCK_FILE="/tmp/${COMPONENT}_archive.lock"
LOG_DIR="/root/single-archiver/archive_script_logs"
LOG_FILE="${LOG_DIR}/${COMPONENT}-$(date +%F).log"

#--------------------------------------------------
# Logging setup
#--------------------------------------------------

mkdir -p "$LOG_DIR"

log()
{
    printf '%s %s\n' "$(date '+%F %T')" "$*" | tee -a "$LOG_FILE"
}

#--------------------------------------------------

exec 200>"$LOCK_FILE"

flock -n 200 || {
    log "Another instance is running."
    exit 1
}

#--------------------------------------------------
# Validate source directory
#--------------------------------------------------

if [[ ! -d "$SRC_DIR" ]]; then
    log "ERROR: Source directory not found: $SRC_DIR"
    exit 1
fi

log "Source directory found."

cd "$SRC_DIR"

#--------------------------------------------------
# Ensure destination exists
#--------------------------------------------------

if [[ ! -d "$DEST_DIR" ]]; then
    log "Destination directory does not exist. Creating..."
    mkdir -p "$DEST_DIR"
else
    log "Destination directory exists."
fi

echo

log "Searching logs using filename date (older than or equal to $DAYS days)..."

#--------------------------------------------------
# Calculate cutoff date
#--------------------------------------------------

CUTOFF_DATE=$(date -d "$DAYS days ago" +%F)

#--------------------------------------------------
# Group files by date
#--------------------------------------------------

declare -A FILE_GROUPS=()

shopt -s nullglob

for file in "${COMPONENT}"-INST_*-*.log.gz
do

    [[ -f "$file" ]] || continue

    # Extract YYYY-MM-DD from filename
    if [[ $file =~ ([0-9]{4}-[0-9]{2}-[0-9]{2}) ]]; then

        FILE_DATE="${BASH_REMATCH[1]}"

        # YYYY-MM-DD allows lexicographical comparison
        if [[ "$FILE_DATE" < "$CUTOFF_DATE" || "$FILE_DATE" == "$CUTOFF_DATE" ]]; then

            # Store each filename on a separate line
            FILE_GROUPS["$FILE_DATE"]+="$file"$'\n'

        fi

    else

        log "WARNING: Filename doesn't match expected date pattern, skipping: $file"

    fi
done

#--------------------------------------------------

if [[ ${#FILE_GROUPS[@]} -eq 0 ]]; then
    log "No logs found older than or equal to $DAYS days."
    exit 0
fi

#--------------------------------------------------
# Process each date
#--------------------------------------------------

while IFS= read -r DATE
do

    ARCHIVE_NAME="${COMPONENT}-${DATE}.tar.gz"

    echo
    log "=================================================="
    log "Processing Date : $DATE"
    log "Archive         : $ARCHIVE_NAME"
    log "=================================================="

    #----------------------------------------------
    # Skip if archive already exists in destination
    #----------------------------------------------

    if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then

        log "Archive already exists in destination."
        log "Skipping..."

        continue
    fi


    #----------------------------------------------
    # Read files for this date
    #----------------------------------------------

    #FILES=()

    mapfile -t FILES < <(
        printf '%s' "${FILE_GROUPS[$DATE]}" | sort -V
    )

    echo
    log "Files being archived (${#FILES[@]}):"
    echo


    #------------------------------------------------
    # File listing intentionally has NO timestamp
    #------------------------------------------------

    printf '  %s\n' "${FILES[@]}" | tee -a "$LOG_FILE"


    #----------------------------------------------
    # Recovery case
    #----------------------------------------------

    if [[ -f "$ARCHIVE_NAME" ]]; then

        log "Archive already exists in source."

        #------------------------------------------
        # Verify existing archive before trusting it
        #------------------------------------------

        ARCHIVE_COUNT=$(tar -tzf "$ARCHIVE_NAME" | wc -l)
        SRC_COUNT=${#FILES[@]}

        if [[ "$ARCHIVE_COUNT" -ne "$SRC_COUNT" ]]; then

            log "ERROR: Verification failed for existing archive $ARCHIVE_NAME"
            log "       Archive has $ARCHIVE_COUNT files, expected $SRC_COUNT."

            exit 1
        fi

        log "Verification OK ($ARCHIVE_COUNT files). Moving archive to destination..."

        mv -f "$ARCHIVE_NAME" "$DEST_DIR/"

        #------------------------------------------
        # Verify archive move to destination
        #------------------------------------------

        if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then

            log "Archive move to destination successfully."
            log "Archive found: $DEST_DIR/$ARCHIVE_NAME"

            chmod 777 "$DEST_DIR/$ARCHIVE_NAME"

            log "Removing source log files..."

            rm -f "${FILES[@]}"

            log "Recovery completed."

        else

            log "ERROR: Destination verification failed."
            log "       Archive not found: $DEST_DIR/$ARCHIVE_NAME"
            log "       Source log files will NOT be deleted."

            exit 1
        fi

        continue
    fi

    #----------------------------------------------
    # Create archive
    #----------------------------------------------

    echo

    log "Creating archive..."

    tar -czf "$ARCHIVE_NAME" "${FILES[@]}"

    log "Archive created."

    #----------------------------------------------
    # Verify archive
    #----------------------------------------------

    log "Verifying archive contents..."

    ARCHIVE_COUNT=$(tar -tzf "$ARCHIVE_NAME" | wc -l)
    SRC_COUNT=${#FILES[@]}

    if [[ "$ARCHIVE_COUNT" -ne "$SRC_COUNT" ]]; then

        log "ERROR: Verification failed for $ARCHIVE_NAME"
        log "       Archive has $ARCHIVE_COUNT files, expected $SRC_COUNT."

        rm -f "$ARCHIVE_NAME"

        exit 1
    fi

    log "Verification OK ($ARCHIVE_COUNT files matched)."

    #----------------------------------------------
    # Move archive
    #----------------------------------------------

    mv -f "$ARCHIVE_NAME" "$DEST_DIR/"

    if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then

        chmod 777 "$DEST_DIR/$ARCHIVE_NAME"

        log "Archive moved successfully."

        log "Deleting source log files..."

        rm -f "${FILES[@]}"

        log "Completed for $DATE"

    else

        log "ERROR: Failed to move archive to destination."

        exit 1
    fi

done < <(printf '%s\n' "${!FILE_GROUPS[@]}" | sort)

echo
log "========================================"
log "All eligible logs processed successfully."
log "========================================"

```