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
ARCHIVE_FAILED=0

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
# Configuration Validation
############################################################

validate_configuration() {

    local component
    local src_dir
    local dest_dir

    log_info "Validating component configuration..."

    #-------------------------------------------------------
    # Validate DAYS Variable
    #-------------------------------------------------------

    if ! [[ "$DAYS" =~ ^[1-9][0-9]*$ ]]; then
        log_error "Invalid DAYS value: '$DAYS'. DAYS must be a positive integer."
        return 1
    fi

    #-------------------------------------------------------
    # Validate every source component
    #-------------------------------------------------------

    for component in "${!SRC_DIRS[@]}"
    do

        src_dir="${SRC_DIRS[$component]}"

        if [[ -z "$src_dir" ]]; then

            log_error "Source directory is empty for component: $component"

            return 1

        fi

        if [[ -z "${DEST_DIRS[$component]+x}" ]]; then

            log_error "Destination directory is not configured for component: $component"

            return 1

        fi

        dest_dir="${DEST_DIRS[$component]}"

        if [[ -z "$dest_dir" ]]; then

            log_error "Destination directory is empty for component: $component"

            return 1

        fi

    done

    #-------------------------------------------------------
    # Validate every destination component
    #-------------------------------------------------------

    for component in "${!DEST_DIRS[@]}"
    do

        if [[ -z "${SRC_DIRS[$component]+x}" ]]; then

            log_error "Source directory is not configured for component: $component"

            return 1

        fi

    done

    log_info "Component configuration validation successful."

    return 0
}


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


log_info() {

    echo "[INFO] $*"

}


log_warning() {

    echo "[WARNING] $*"

}


log_error() {

    echo "[ERROR] $*"

}


############################################################
# Cutoff Date
############################################################

CUTOFF_DATE=$(date -d "$DAYS days ago" +%F)


############################################################
# Find and Group Logs by Date Wise
############################################################

find_and_group_logs() {

    local COMPONENT="$1"
    local CUTOFF_DATE="$2"

    local FILE_DATE

    FILE_GROUPS=()
    # FILE_GROUPS=() - Declaring inside find_and_group_logs() funtion, so that it clears the previous component's groups before processing the next component.

    shopt -s nullglob

    #-------------------------------------------------------
    # Group logs by filename date
    #-------------------------------------------------------

    for file in "${COMPONENT}"-INST_*-*.log.gz
    do
        [[ -f "$file" ]] || continue

        if [[ $file =~ ([0-9]{4}-[0-9]{2}-[0-9]{2}) ]]; then

            FILE_DATE="${BASH_REMATCH[1]}"

            if [[ "$FILE_DATE" < "$CUTOFF_DATE" || "$FILE_DATE" == "$CUTOFF_DATE" ]]; then

                FILE_GROUPS["$FILE_DATE"]+="$file"$'\n'

            fi
        fi
    done
}


############################################################
# Verify Archive
############################################################

verify_archive() {

    local ARCHIVE_NAME="$1"
    local FILES_ARRAY="$2"

    # local -n —> Creates a reference to an EXISTING array
    # References the array PASSED BY NAME
    local -n FILES_REF="$FILES_ARRAY"

    local FILE

    log_info "Verifying archive contents..."
    echo

    #-------------------------------------------------------
    # Verify archive integrity
    #-------------------------------------------------------

    #if ! tar -tzf "$ARCHIVE_NAME" >/dev/null 2>&1; then

    #    log_error "Archive is corrupted or cannot be read."

    #    return 1

    #fi

    #-------------------------------------------------------
    # Read archive contents once
    #-------------------------------------------------------

    #mapfile -t ARCHIVE_FILES < <(tar -tzf "$ARCHIVE_NAME")

    mapfile -t ARCHIVE_FILES < <(tar -tzf "$ARCHIVE_NAME") || {
        log_error "Archive is corrupted or cannot be read."
        return 1
    }

    #-------------------------------------------------------
    # Verify every source file exists in archive
    #-------------------------------------------------------

    for FILE in "${FILES_REF[@]}"
    do

        if printf '%s\n' "${ARCHIVE_FILES[@]}" | grep -Fx "$FILE" >/dev/null 2>&1; then

            log_info "Verified in archive : $FILE"

        else

            log_error "File NOT found in archive : $FILE"

            return 1

        fi

    done

    echo
    log_info "All source files verified in archive."

    return 0
}


############################################################
# Handle Existing Destination Archive and remove source
############################################################

handle_existing_destination_archive() {

    local ARCHIVE_NAME="$1"
    local DEST_DIR="$2"
    local DATE="$3"
    local FILES_ARRAY="$4"

    # local -n —> Creates a reference to an EXISTING array
    # References the array PASSED BY NAME
    local -n FILES_REF="$FILES_ARRAY"


    echo
    log_info "Archive already exists in the destination on this ${DATE}."

    #-------------------------------------------------------
    # Verify existing destination archive
    #-------------------------------------------------------

    if ! verify_archive "$DEST_DIR/$ARCHIVE_NAME" "$FILES_ARRAY"; then

        log_error "Existing archive does not match source files."
        log_error "Source files will NOT be deleted."

        return 1

    fi

    #-------------------------------------------------------
    # Existing archive is valid
    #-------------------------------------------------------

    log_info "Existing archive matches all source files."

    #-------------------------------------------------------
    # Delete source files
    #-------------------------------------------------------

    echo
    log_warning "Deleting source files..."

    rm -f "${FILES_REF[@]}"

    log_info "Source files deleted successfully."
    log_info "Archive processing completed."

    return 0
}


############################################################
# Recover Existing Archive and Move to Destination
############################################################

recover_archive() {

    local ARCHIVE_NAME="$1"
    local DEST_DIR="$2"
    local FILES_ARRAY="$3"

    # local -n —> Creates a reference to an EXISTING array
    # References the array PASSED BY NAME
    local -n FILES_REF="$FILES_ARRAY"

    echo
    log_info "Existing Archive found in the source directiory on this ${DATE} date...."
    log_info "Recovery mode detected."

    #-------------------------------------------------------
    # Verify Archive
    #-------------------------------------------------------

    if ! verify_archive "$ARCHIVE_NAME" "$FILES_ARRAY"; then

        log_error "Recovery verification failed."
        log_error "Source files will NOT be deleted."

        return 1

    fi

    #-------------------------------------------------------
    # Move verified archive
    #-------------------------------------------------------

    log_info "Moving existing archive..."

    mv -f "$ARCHIVE_NAME" "$DEST_DIR/"

    chmod 777 "$DEST_DIR/$ARCHIVE_NAME"

    #-------------------------------------------------------
    # Delete source files
    #-------------------------------------------------------
    
    log_warning "Deleting source logs..."

    rm -f "${FILES_REF[@]}"

    log_info "Recovery completed."

    return 0
}


############################################################
# Create and Store Archive for Each Date
############################################################

create_archive() {

    local ARCHIVE_NAME="$1"
    local DEST_DIR="$2"
    local FILES_ARRAY="$3"
    
    # local -n —> Creates a reference to an EXISTING array
    # References the array PASSED BY NAME
    local -n FILES_REF="$FILES_ARRAY"

    #-------------------------------------------------------
    # Create Archive
    #-------------------------------------------------------

    echo
    log_info "Creating archive..."

    tar -czf "$ARCHIVE_NAME" "${FILES_REF[@]}"

    log_info "Archive created."

    #-------------------------------------------------------
    # Verify Archive
    #-------------------------------------------------------

    if ! verify_archive "$ARCHIVE_NAME" "$FILES_ARRAY"; then

        log_error "Archive verification failed."
        log_info "Source files will NOT be deleted."

        rm -f "$ARCHIVE_NAME"

        return 1

    fi

    #-------------------------------------------------------
    # Move Archive
    #-------------------------------------------------------

    log_info "Moving archive..."

    mv -f "$ARCHIVE_NAME" "$DEST_DIR/"

    #-------------------------------------------------------
    # Verify Destination
    #-------------------------------------------------------

    if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then

        chmod 777 "$DEST_DIR/$ARCHIVE_NAME"

        log_info "Archive moved successfully."

        #---------------------------------------------------
        # Delete Source Files
        #---------------------------------------------------

        log_warning "Deleting source log files..."

        rm -f "${FILES_REF[@]}"

        log_info "Completed."

    else

        log_error "Failed to move archive."
        log_info "Source files will NOT be deleted."

        return 1

    fi

    return 0
}


############################################################
# Process Each Date and Generate Archive for Each Date
############################################################

process_archive_by_date() {

    local COMPONENT="$1"
    local DATE="$2"
    local DEST_DIR="$3"

    local ARCHIVE_NAME
    
    # local -a —> Declares a new local array
    # Creates a NEW, EMPTY local array
    local -a FILES=()

    ARCHIVE_NAME="${COMPONENT}-${DATE}.tar.gz"

    echo
    echo "============================================================"
    echo "Processing Date : $DATE"
    echo "Archive         : $ARCHIVE_NAME"
    echo "============================================================"


    #-------------------------------------------------------
    # Get files for this date
    #-------------------------------------------------------

    #read -ra FILES <<< "${FILE_GROUPS[$DATE]}"
    mapfile -t FILES < <(
        printf '%s' "${FILE_GROUPS[$DATE]}" | sort -V
    )

    if [[ ${#FILES[@]} -eq 0 ]]; then

        log_info "No files found on this ${DATE}."

        return

    fi

    #-------------------------------------------------------
    # Show files which going to archive
    #-------------------------------------------------------

    echo
    log_info "Files being archived (${#FILES[@]}):"
    echo

    printf '    %s\n' "${FILES[@]}"


    #-------------------------------------------------------
    # If archive already exists in destination
    #-------------------------------------------------------

    if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then

        handle_existing_destination_archive \
            "$ARCHIVE_NAME" \
            "$DEST_DIR" \
            "$DATE" \
            FILES

        return $?

    fi


    #-------------------------------------------------------
    # Recovery if Archive exist in the source
    #-------------------------------------------------------

    if [[ -f "$ARCHIVE_NAME" ]]; then

        recover_archive \
            "$ARCHIVE_NAME" \
            "$DEST_DIR" \
            FILES

        return

    fi

    #-------------------------------------------------------
    # Create Archive if not in source or destination
    #-------------------------------------------------------

    create_archive \
        "$ARCHIVE_NAME" \
        "$DEST_DIR" \
        FILES

}


#########################################################################
# Archive Function -> find_and_group_logs () and process_archive_by_date ()
#########################################################################

archive_component() {

    (
        local COMPONENT="$1"
        local SRC_DIR="$2"
        local DEST_DIR="$3"

        echo
        echo "############################################################"
        echo "Component      : $COMPONENT"
        echo "Source         : $SRC_DIR"
        echo "Destination    : $DEST_DIR"
        echo "############################################################"

        #-------------------------------------------------------
        # Validate Source
        #-------------------------------------------------------

        if [[ ! -d "$SRC_DIR" ]]; then
            log_error "Source directory not found: $SRC_DIR"
            log_info "Skipping component."
            return 1
        fi
        
        log_info "Source directory found."
        cd "$SRC_DIR"

        #-------------------------------------------------------
        # Ensure destination exists
        #-------------------------------------------------------

        if [[ ! -d "$DEST_DIR" ]]; then
            log_info "Destination directory does not exist. Creating..."
            mkdir -p "$DEST_DIR"
        else
            log_info "Destination directory exists."
        fi

        #-------------------------------------------------------
        # Find and Group Log Files
        #-------------------------------------------------------

        echo
        log_info "Searching logs using filename date (older than or equal to $DAYS days)..."

        find_and_group_logs "$COMPONENT" "$CUTOFF_DATE"

        if [[ ${#FILE_GROUPS[@]} -eq 0 ]]; then
            log_info "No eligible logs found."
            return
        fi

        #-------------------------------------------------------
        # Process each date
        #-------------------------------------------------------

        while IFS= read -r DATE
        do

            process_archive_by_date \
                "$COMPONENT" \
                "$DATE" \
                "$DEST_DIR"

        done < <(printf '%s\n' "${!FILE_GROUPS[@]}" | sort)

        echo
        log_info "Finished component : $COMPONENT"
    )


}

############################################################
# Main Funtion -> archive_component() Funtion
############################################################

echo
echo "============================================================"
echo "Log Archive Started"
echo "Retention (Filename Date): $DAYS days"
echo "Cutoff Date              : $CUTOFF_DATE"
echo "============================================================"

#-------------------------------------------------------
# Validate Configuration
#-------------------------------------------------------

if ! validate_configuration; then

    log_error "Configuration validation failed."
    log_error "Archive process aborted."

    exit 1

fi

while IFS= read -r COMPONENT
do
    if ! archive_component \
        "$COMPONENT" \
        "${SRC_DIRS[$COMPONENT]}" \
        "${DEST_DIRS[$COMPONENT]}"
    then
        ARCHIVE_FAILED=1
    fi
done < <(printf '%s\n' "${!SRC_DIRS[@]}" | sort)


if [[ "$ARCHIVE_FAILED" -ne 0 ]]; then
    exit 1
fi

exit 0

echo
echo "============================================================"
echo "All components processed successfully."
echo "============================================================"
```