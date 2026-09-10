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

ARCHIVE_FAILED=0
TOTAL_COMPONENTS=0
SUCCESSFUL_COMPONENTS=0
FAILED_COMPONENTS=0
FAILED_COMPONENTS_LIST=""

ARCHIVES_CREATED=0
ARCHIVES_RECOVERED=0
ARCHIVES_EXISTING=0

START_TIME=$(date '+%Y-%m-%d %H:%M:%S')

LOCK_FILE="/tmp/log_archive.lock"
LOG_DIR="/home/scripts/logs/log_archive"
LOG_FILE="$LOG_DIR/log_archive_$(date +%Y-%m-%d_%H%M%S).log"
ARCHIVE_RESULT_FILE="/tmp/log_archive_result_$$"

#-----------------------------------------------------------
# Declaring associative arrays
#-----------------------------------------------------------

declare -A FILE_GROUPS=()
# Declaring outside the function so that both functions can access it.

declare -A COMPONENT_CREATED=()
declare -A COMPONENT_RECOVERED=()
declare -A COMPONENT_EXISTING=()
declare -A COMPONENT_STATUS=()

#-----------------------------------------------------------
# Source Directories
#-----------------------------------------------------------

############################################################
# Component Configuration
############################################################

# Format:
#   "COMPONENT:SRC_DIR:DEST_DIR"

COMPONENTS=(
    "apigw-nagad-app7:/home/apigw/log/archive:/LOGS/app7/apigw"
    "dmscore-nagad-app7:/home/dmscore/log/archive:/LOGS/app7/dmscore"
)


############################################################
# Configuration Validation
############################################################

validate_configuration() {

    local entry
    local name src dest

    log_info "Validating component configuration..."

    if [[ ${#COMPONENTS[@]} -eq 0 ]]; then
        log_error "No components configured."
        return 1
    fi

    for entry in "${COMPONENTS[@]}"; do

        IFS=':' read -r name src dest <<< "$entry"

        if [[ -z "$name" ]]; then
            log_error "Component entry has empty name: '$entry'"
            return 1
        fi

        if [[ -z "$src" ]]; then
            log_error "Source directory is empty for component: $name"
            return 1
        fi

        if [[ -z "$dest" ]]; then
            log_error "Destination directory is empty for component: $name"
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


log_timestamp() {

    date '+%Y-%m-%d %H:%M:%S'

}


log_info() {

    echo "[$(log_timestamp)] [INFO] $*"

}


log_warning() {

    echo "[$(log_timestamp)] [WARNING] $*"

}


log_error() {

    echo "[$(log_timestamp)] [ERROR] $*"

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
    local -n FILES_REF="$FILES_ARRAY"

    local FILE
    local ARCHIVE_FILE
    local ARCHIVE_OUTPUT
    local NORMALIZED_FILE

    local -a ARCHIVE_FILES=()
    local -a MISSING_FILES=()
    local -A ARCHIVE_SET=()

    echo
    log_info "Verifying archive contents..."

    # 1. Verify archive readability and capture file listing
    if ! ARCHIVE_OUTPUT=$(tar -tzf "$ARCHIVE_NAME" 2>&1); then
        log_error "Archive is corrupted or cannot be read: $ARCHIVE_OUTPUT"
        return 1
    fi

    # 2. Parse archive listing into an array
    mapfile -t ARCHIVE_FILES <<< "$ARCHIVE_OUTPUT"

    # 3. Build archive lookup set
    for ARCHIVE_FILE in "${ARCHIVE_FILES[@]}"
    do
        NORMALIZED_FILE="${ARCHIVE_FILE#./}"
        ARCHIVE_SET["$NORMALIZED_FILE"]=1
    done

    # 4. Check all source files using direct lookup
    for FILE in "${FILES_REF[@]}"
    do
        NORMALIZED_FILE="${FILE#./}"

        if [[ -z "${ARCHIVE_SET[$NORMALIZED_FILE]+x}" ]]; then
            MISSING_FILES+=("$FILE")
        fi
    done

    # 5. Report missing files
    if [[ ${#MISSING_FILES[@]} -gt 0 ]]; then
        echo
        log_error "File(s) NOT found in archive (${#MISSING_FILES[@]} missing):"
        printf '    [MISSING] %s\n' "${MISSING_FILES[@]}"
        return 1
    fi

    echo
    log_info "All ${#FILES_REF[@]} source files verified in archive."
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

        echo
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

    COMP_ARCHIVES_EXISTING=$((COMP_ARCHIVES_EXISTING + 1))

    return 0
}


############################################################
# Recover Existing Archive and Move to Destination
############################################################

recover_archive() {

    local ARCHIVE_NAME="$1"
    local DEST_DIR="$2"
    local DATE="$3"
    local FILES_ARRAY="$4"

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
        
        echo
        log_error "Recovery verification failed."
        log_error "Source files will NOT be deleted."

        local CORRUPT_ARCHIVE
        CORRUPT_ARCHIVE="${ARCHIVE_NAME}.corrupt.$(date '+%Y%m%d_%H%M%S')"

        if mv -f "$ARCHIVE_NAME" "$CORRUPT_ARCHIVE"; then
            log_warning "Invalid recovery archive quarantined as: $CORRUPT_ARCHIVE"
        else
            log_error "Failed to quarantine invalid recovery archive: $ARCHIVE_NAME"
        fi

        return 1
    fi

    #-------------------------------------------------------
    # Move verified archive
    #-------------------------------------------------------

    log_info "Moving existing archive..."

    if ! mv -f "$ARCHIVE_NAME" "$DEST_DIR/"; then
        log_error "Failed to move recovered archive to destination."
        log_error "Source files will NOT be deleted."
        return 1
    fi

    if [[ ! -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then
        log_error "Recovered archive is not present in destination after move."
        log_error "Source files will NOT be deleted."
        return 1
    fi

    chmod 777 "$DEST_DIR/$ARCHIVE_NAME"

    #-------------------------------------------------------
    # Delete source files
    #-------------------------------------------------------
    
    log_warning "Deleting source logs..."

    rm -f "${FILES_REF[@]}"

    log_info "Source files deleted successfully."
    log_info "Recovery completed."

    COMP_ARCHIVES_RECOVERED=$((COMP_ARCHIVES_RECOVERED + 1))

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

    if ! tar -czf "$ARCHIVE_NAME" "${FILES_REF[@]}"; then

        log_error "Failed to create archive: $ARCHIVE_NAME"
        log_info "Source files will NOT be deleted."

        return 1

    fi

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
    echo
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

        echo
        log_warning "Deleting source log files..."

        rm -f "${FILES_REF[@]}"

        log_info "Source files deleted successfully."
        log_info "Archive processing completed."
        
        COMP_ARCHIVES_CREATED=$((COMP_ARCHIVES_CREATED + 1))

    else
        echo
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
            "$DATE" \
            FILES

        return $?

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

        local COMP_ARCHIVES_CREATED=0
        local COMP_ARCHIVES_RECOVERED=0
        local COMP_ARCHIVES_EXISTING=0
        local COMPONENT_PROCESS_FAILED=0
        local COMPONENT_STATUS="SUCCESS"

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

            if ! process_archive_by_date \
                "$COMPONENT" \
                "$DATE" \
                "$DEST_DIR"
            then
                echo
                log_error "Archive processing failed for date: $DATE"
                COMPONENT_PROCESS_FAILED=1
                COMPONENT_STATUS="FAILED"

            fi

        done < <(printf '%s\n' "${!FILE_GROUPS[@]}" | sort)

        echo
        log_info "Finished component : $COMPONENT"

        printf '%s|%s|%s|%s|%s\n' \
            "$COMPONENT" \
            "$COMP_ARCHIVES_CREATED" \
            "$COMP_ARCHIVES_RECOVERED" \
            "$COMP_ARCHIVES_EXISTING" \
            "$COMPONENT_STATUS" \
            >> "$ARCHIVE_RESULT_FILE"


        if [[ "$COMPONENT_PROCESS_FAILED" -ne 0 ]]; then

            log_error "Component processing failed : $COMPONENT"
            return 1

        fi        

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


for COMPONENT_ENTRY in "${COMPONENTS[@]}"
do
    IFS=':' read -r COMPONENT SRC_DIR DEST_DIR <<< "$COMPONENT_ENTRY"

    TOTAL_COMPONENTS=$((TOTAL_COMPONENTS + 1))

    if archive_component \
        "$COMPONENT" \
        "$SRC_DIR" \
        "$DEST_DIR"
    then

        SUCCESSFUL_COMPONENTS=$((SUCCESSFUL_COMPONENTS + 1))

    else

        FAILED_COMPONENTS=$((FAILED_COMPONENTS + 1))
        FAILED_COMPONENTS_LIST+="$COMPONENT, "
        ARCHIVE_FAILED=1

    fi

    #-------------------------------------------------------------
    # Collect archive counters and status from component subshell
    #-------------------------------------------------------------

    if [[ -f "$ARCHIVE_RESULT_FILE" ]]; then

        while IFS='|' read -r RESULT_COMPONENT CREATED RECOVERED EXISTING STATUS
        do

            if [[ "$RESULT_COMPONENT" == "$COMPONENT" ]]; then

                ARCHIVES_CREATED=$((ARCHIVES_CREATED + CREATED))
                ARCHIVES_RECOVERED=$((ARCHIVES_RECOVERED + RECOVERED))
                ARCHIVES_EXISTING=$((ARCHIVES_EXISTING + EXISTING))

                COMPONENT_CREATED["$RESULT_COMPONENT"]="$CREATED"
                COMPONENT_RECOVERED["$RESULT_COMPONENT"]="$RECOVERED"
                COMPONENT_EXISTING["$RESULT_COMPONENT"]="$EXISTING"
                COMPONENT_STATUS["$RESULT_COMPONENT"]="$STATUS"

            fi

        done < "$ARCHIVE_RESULT_FILE"

        # Clear result after processing this component
        > "$ARCHIVE_RESULT_FILE"

    fi

done < <(printf '%s\n' "${!SRC_DIRS[@]}" | sort)


END_TIME=$(date '+%Y-%m-%d %H:%M:%S')

#-------------------------------------------------------
# Execution Summary Start from Here
#-------------------------------------------------------

echo
echo "======================================================================"
echo "Archive Execution Summary"
echo "======================================================================"
echo "Start Time               : $START_TIME"
echo "End Time                 : $END_TIME"
echo
echo "Total Components         : $TOTAL_COMPONENTS"
echo "Successful Components    : $SUCCESSFUL_COMPONENTS"
echo "Failed Components        : $FAILED_COMPONENTS"
echo
echo "Each Component Summary"
echo "----------------------------------------------------------------------"
printf "%-22s %8s %11s %10s %10s\n" \
    "Component" \
    "Created" \
    "Recovered" \
    "Existing" \
    "Status"
echo "----------------------------------------------------------------------"

for COMPONENT_ENTRY in "${COMPONENTS[@]}"
do

    IFS=':' read -r COMPONENT SRC_DIR DEST_DIR <<< "$COMPONENT_ENTRY"

    printf "%-22s %8s %11s %10s %10s\n" \
        "$COMPONENT" \
        "${COMPONENT_CREATED[$COMPONENT]:-0}" \
        "${COMPONENT_RECOVERED[$COMPONENT]:-0}" \
        "${COMPONENT_EXISTING[$COMPONENT]:-0}" \
        "${COMPONENT_STATUS[$COMPONENT]:-}"

done

echo "----------------------------------------------------------------------"
echo
echo "Total Archives Created   : $ARCHIVES_CREATED"
echo "Total Archives Recovered : $ARCHIVES_RECOVERED"
echo "Total Archives Exist     : $ARCHIVES_EXISTING"

if [[ "$FAILED_COMPONENTS" -gt 0 ]]; then
    echo "Failed Component List    : ${FAILED_COMPONENTS_LIST%, }"
fi

if [[ "$ARCHIVE_FAILED" -ne 0 ]]; then
    echo "Overall Status           : FAILED"
else
    echo "Overall Status           : SUCCESS"
fi

echo "======================================================================"

#-------------------------------------------------------
# Cleanup Result file and Show Exit Status
#-------------------------------------------------------

rm -f "$ARCHIVE_RESULT_FILE"

if [[ "$ARCHIVE_FAILED" -ne 0 ]]; then
    exit 1
fi

exit 0

```