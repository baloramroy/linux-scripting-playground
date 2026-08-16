```bash
#!/bin/bash
#
# Multi Component Log Archive Script
#
# Features:
#   - Archives logs based on filename date (NOT mtime)
#   - Supports multiple components
#   - Supports multiple instances (INST_1, INST_2, ...)
#   - Creates one archive per day per component
#   - Recovery handling
#   - Single lock file
#

set -euo pipefail

############################################################
# Configuration
############################################################

DAYS=3
LOCK_FILE="/tmp/log_archive.lock"

LOG_DIR="/home/scripts/logs/log_archive"
LOG_FILE="$LOG_DIR/log_archive_$(date +%Y-%m-%d_%H%M%S).log"

declare -A FILE_GROUPS=()
# Declaring outside the function so that both functions can access it.


#-----------------------------------------------------------
# Source Directories
#-----------------------------------------------------------

declare -A SRC_DIRS=(
    ["apigw-nagad-app7"]="/home/apigw/log/archive"
    ["dmscore-nagad-app7"]="/home/dmscore/log/archive"
)

#-----------------------------------------------------------
# Destination Directories
#-----------------------------------------------------------

declare -A DEST_DIRS=(
    ["apigw-nagad-app7"]="/LOGS/app7/apigw"
    ["dmscore-nagad-app7"]="/LOGS/app7/dmscore"
)

############################################################
# Lock
############################################################

exec 200>"$LOCK_FILE"

flock -n 200 || {
    echo "Another archive process is already running."
    exit 1
}


############################################################
# Logging
############################################################

mkdir -p "$LOG_DIR"

exec >> "$LOG_FILE" 2>&1

############################################################
# Cutoff Date
############################################################

CUTOFF_DATE=$(date -d "$DAYS days ago" +%F)


############################################################
# Find and Group Logs
############################################################

find_and_group_logs() {

    local COMPONENT="$1"
    local CUTOFF_DATE="$2"

    FILE_GROUPS=()
    # FILE_GROUPS=() - Declaring inside find_and_group_logs() funtion, so that it clears the previous component's groups before processing the next component.

    shopt -s nullglob

    ########################################################
    # Group logs by filename date
    ########################################################

    for file in "${COMPONENT}"-INST_*-*.log.gz
    do
        [[ -f "$file" ]] || continue

        if [[ $file =~ ([0-9]{4}-[0-9]{2}-[0-9]{2}) ]]; then

            FILE_DATE="${BASH_REMATCH[1]}"

            if [[ "$FILE_DATE" < "$CUTOFF_DATE" || "$FILE_DATE" == "$CUTOFF_DATE" ]]; then

                FILE_GROUPS["$FILE_DATE"]+="$file "

            fi
        fi
    done
}


############################################################
# Archive Function
############################################################

archive_component() {

    local COMPONENT="$1"
    local SRC_DIR="$2"
    local DEST_DIR="$3"

    echo
    echo "############################################################"
    echo "Component      : $COMPONENT"
    echo "Source         : $SRC_DIR"
    echo "Destination    : $DEST_DIR"
    echo "############################################################"

    ########################################################
    # Validate Source
    ########################################################

    if [[ ! -d "$SRC_DIR" ]]; then
        echo "ERROR: Source directory not found: $SRC_DIR"
        echo "Skipping component."
        return
    fi

    echo "Source directory found."
    cd "$SRC_DIR"

    ########################################################
    # Ensure destination exists
    ########################################################

    if [[ ! -d "$DEST_DIR" ]]; then
        echo "Destination directory does not exist. Creating..."
        mkdir -p "$DEST_DIR"
    else
        echo "Destination directory exists."
    fi

    ########################################################
    # Find and Group Logs
    ########################################################

    echo
    echo "Searching logs using filename date (older than or equal to $DAYS days)..."

    find_and_group_logs "$COMPONENT" "$CUTOFF_DATE"

    if [[ ${#FILE_GROUPS[@]} -eq 0 ]]; then
        echo "No eligible logs found."
        return
    fi

    ########################################################
    # Process each date
    ########################################################

    while IFS= read -r DATE
    do

        ARCHIVE_NAME="${COMPONENT}-${DATE}.tar.gz"

        echo
        echo "============================================================"
        echo "Processing Date : $DATE"
        echo "Archive         : $ARCHIVE_NAME"
        echo "============================================================"

        ####################################################
        # Archive already exists in destination
        ####################################################

        if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then
            echo "Archive already exists."
            echo "Skipping..."
            continue
        fi

        read -ra FILES <<< "${FILE_GROUPS[$DATE]}"

        if [[ ${#FILES[@]} -eq 0 ]]; then
            echo "No files found."
            continue
        fi

        ####################################################
        # Show files
        ####################################################

        echo
        echo "Files being archived (${#FILES[@]}):"
        echo

        printf '    %s\n' "${FILES[@]}"

        ####################################################
        # Recovery
        ####################################################

        if [[ -f "$ARCHIVE_NAME" ]]; then

            echo
            echo "Recovery mode detected."

            echo "Verifying archive contents..."

            ################################################
            # Verify archive integrity
            ################################################

            if ! tar -tzf "$ARCHIVE_NAME" >/dev/null 2>&1; then
                echo "ERROR: Archive is corrupted or cannot be read."
                echo "Source files will NOT be deleted."
                continue
            fi

            ################################################
            # Read archive contents once
            ################################################

            mapfile -t ARCHIVE_FILES < <(tar -tzf "$ARCHIVE_NAME")

            ################################################
            # Verify every source file exists in archive
            ################################################

            RECOVERY_OK=true

            for FILE in "${FILES[@]}"
            do

                if printf '%s\n' "${ARCHIVE_FILES[@]}" | grep -Fxq "$FILE"; then

                    echo "Verified in archive : $FILE"

                else

                    echo "ERROR: File NOT found in archive : $FILE"
                    RECOVERY_OK=false

                fi

            done

            ################################################
            # Delete only if ALL files are verified
            ################################################

            if [[ "$RECOVERY_OK" != true ]]; then

                echo
                echo "ERROR: Recovery verification failed."
                echo "Source files will NOT be deleted."
                continue

            fi

            ################################################
            # Move verified archive
            ################################################

            echo
            echo "All source files verified in archive."

            echo "Moving existing archive..."

            mv -f "$ARCHIVE_NAME" "$DEST_DIR/"

            chmod 777 "$DEST_DIR/$ARCHIVE_NAME"

            ################################################
            # Delete source files
            ################################################

            echo "Deleting source logs..."

            rm -f "${FILES[@]}"

            echo "Recovery completed."

            continue

        fi


        ####################################################
        # Create Archive
        ####################################################

        echo
        echo "Creating archive..."

        tar -czf "$ARCHIVE_NAME" "${FILES[@]}"

        echo "Archive created."

        ####################################################
        # Verify Archive
        ####################################################

        echo
        echo "Verifying archive..."

        if ! tar -tzf "$ARCHIVE_NAME" >/dev/null 2>&1; then

            echo "ERROR : Archive verification failed."
            echo "Source files will NOT be deleted."

            rm -f "$ARCHIVE_NAME"

            continue

        fi

        ####################################################
        # Read archive contents once
        ####################################################

        mapfile -t ARCHIVE_FILES < <(tar -tzf "$ARCHIVE_NAME")

        ####################################################
        # Verify source files are inside archive
        ####################################################

        ARCHIVE_OK=true

        for FILE in "${FILES[@]}"
        do

            if printf '%s\n' "${ARCHIVE_FILES[@]}" | grep -Fxq "$FILE"; then

                echo "Verified in archive : $FILE"

            else

                echo "ERROR : File NOT found in archive : $FILE"
                ARCHIVE_OK=false

            fi

        done

        ####################################################
        # Stop if any file is missing
        ####################################################

        if [[ "$ARCHIVE_OK" != true ]]; then

            echo
            echo "ERROR : Archive verification failed."
            echo "Source files will NOT be deleted."

            rm -f "$ARCHIVE_NAME"

            continue

        fi

        echo
        echo "All source files verified in archive."

        ####################################################
        # Move Archive
        ####################################################

        echo "Moving archive..."

        mv -f "$ARCHIVE_NAME" "$DEST_DIR/"

        ####################################################
        # Verify Destination
        ####################################################

        if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then

            chmod 777 "$DEST_DIR/$ARCHIVE_NAME"

            echo "Archive moved successfully."

            ################################################
            # Delete Source Files
            ################################################

            echo "Deleting source log files..."

            rm -f "${FILES[@]}"

            echo "Completed."

        else

            echo "ERROR : Failed to move archive."
            echo "Source files will NOT be deleted."

            exit 1

        fi

    done < <(printf '%s\n' "${!FILE_GROUPS[@]}" | sort)

    echo
    echo "Finished component : $COMPONENT"
}

############################################################
# Main
############################################################

echo
echo "============================================================"
echo "Log Archive Started"
echo "Retention (Filename Date): $DAYS days"
echo "Cutoff Date              : $CUTOFF_DATE"
echo "============================================================"

while IFS= read -r COMPONENT
do

    archive_component \
        "$COMPONENT" \
        "${SRC_DIRS[$COMPONENT]}" \
        "${DEST_DIRS[$COMPONENT]}"

done < <(printf '%s\n' "${!SRC_DIRS[@]}" | sort)

echo
echo "============================================================"
echo "All components processed successfully."
echo "============================================================"
```
