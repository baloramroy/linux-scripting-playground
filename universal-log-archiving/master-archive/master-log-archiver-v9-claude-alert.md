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
TEAMS_NOTIFICATION_FAILED=0

START_TIME=$(date '+%Y-%m-%d %H:%M:%S')

LOCK_FILE="/tmp/master_archiver.lock"
LOG_DIR="/home/scripts/master_archiver/logs"
LOG_FILE="$LOG_DIR/log_archive_$(date +%Y-%m-%d_%H%M%S).log"

#-----------------------------------------------------------
# Teams Alert Configuration
#-----------------------------------------------------------

WEBHOOK_URL="https://default1-hVb4xMbs"

USE_PROXY="NO"
HTTPS_PROXY="http://10.210.10.173:4899"

#-----------------------------------------------------------
# Declaring associative arrays
#-----------------------------------------------------------

declare -A COMPONENT_CREATED=()
declare -A COMPONENT_RECOVERED=()
declare -A COMPONENT_EXISTING=()
declare -A COMPONENT_STATUS=()
declare -A COMPONENT_RESULT=()

#-----------------------------------------------------------
# Source Directories
#-----------------------------------------------------------

############################################################
# Component Configuration
############################################################

# Format:
#   "COMPONENT|SRC_DIR|DEST_DIR"

COMPONENTS=(
    "apigw-nagad-app7|/home/apigw/log/archive|/LOGS/app7/apigw"
    "dmscore-nagad-app7|/home/dmscore/log/archive|/LOGS/app7/dmscore"
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

        IFS='|' read -r name src dest <<< "$entry"

        #-------------------------------------------------------
        # Validate component entry format
        #-------------------------------------------------------

        # Guard against unexpected delimiter in the entry.
        # Reconstruct the entry from parsed fields and compare it
        # with the original entry to detect extra fields.

        if [[ "$entry" != "$name|$src|$dest" ]]; then
            log_error "Component entry has unexpected field count (check for extra delimiter): '$entry'"
            return 1
        fi

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
    local GROUPS_ARRAY="$2"
    local CUTOFF_DATE="$3"

    local -n GROUPS_REF="$GROUPS_ARRAY"

    local FILE_DATE

    GROUPS_REF=()

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

                GROUPS_REF["$FILE_DATE"]+="$file"$'\n'

            fi
        
        else

            log_warning "Filename doesn't match expected date pattern, skipping: $file"
        
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
        
        log_error "File(s) NOT found in archive (${#MISSING_FILES[@]} missing):"
        printf '    [MISSING] %s\n' "${MISSING_FILES[@]}"
        echo

        return 1
    fi

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

    #-------------------------------------------------------
    # Verify Existing Archive
    #-------------------------------------------------------

    echo
    log_info "Archive already exists in the destination on this ${DATE}."

    if ! verify_archive "$DEST_DIR/$ARCHIVE_NAME" "$FILES_ARRAY"; then

        log_error "Existing archive does not match source files."
        log_error "Source files will NOT be deleted."

        return 1

    fi

    log_info "Existing archive matches all source files."

    COMP_ARCHIVES_EXISTING=$((COMP_ARCHIVES_EXISTING + 1))

    #-------------------------------------------------------
    # Delete Source Files
    #-------------------------------------------------------

    echo
    log_warning "Deleting source log files..."

    if ! rm -f "${FILES_REF[@]}"; then

        log_error "Failed to delete one or more source log files."
        log_error "Existing archive is verified, but source files were NOT completely deleted."

        return 1

    fi

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
    local DATE="$3"
    local FILES_ARRAY="$4"

    # local -n —> Creates a reference to an EXISTING array
    # References the array PASSED BY NAME
    local -n FILES_REF="$FILES_ARRAY"

    #-------------------------------------------------------
    # Recovery Verification
    #-------------------------------------------------------

    echo
    log_info "Existing archive found in the source directory on this ${DATE} date."
    log_info "Recovery mode detected."

    if ! verify_archive "$ARCHIVE_NAME" "$FILES_ARRAY"; then

        log_error "Recovery verification failed."
        log_error "Source files will NOT be deleted."

        local CORRUPT_ARCHIVE
        CORRUPT_ARCHIVE="${ARCHIVE_NAME}.corrupt.$(date '+%Y%m%d_%H%M%S')"

        if ! mv -f "$ARCHIVE_NAME" "$CORRUPT_ARCHIVE"; then
            log_error "Failed to quarantine invalid recovery archive: $ARCHIVE_NAME"
        else
            log_warning "Invalid recovery archive quarantined as: $CORRUPT_ARCHIVE"
        fi

        return 1

    fi

    #-------------------------------------------------------
    # Move Archive
    #-------------------------------------------------------

    echo
    log_info "Moving recovered archive..."

    if ! mv -f "$ARCHIVE_NAME" "$DEST_DIR/"; then

        log_error "Failed to move recovered archive: $ARCHIVE_NAME"
        log_error "Source files will NOT be deleted."

        return 1

    fi

    if [[ ! -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then

        log_error "Recovered archive is not present in destination after move: $DEST_DIR/$ARCHIVE_NAME"
        log_error "Source files will NOT be deleted."

        return 1

    fi

    log_info "Recovered archive moved successfully."

    COMP_ARCHIVES_RECOVERED=$((COMP_ARCHIVES_RECOVERED + 1))

    #-------------------------------------------------------
    # Set Archive Permissions
    #-------------------------------------------------------

    if ! chmod 777 "$DEST_DIR/$ARCHIVE_NAME"; then
        log_warning "Failed to set permissions on archive: $DEST_DIR/$ARCHIVE_NAME"
        log_warning "Continuing anyway — archive is present and verified."
    fi

    #-------------------------------------------------------
    # Delete Source Files
    #-------------------------------------------------------

    echo
    log_warning "Deleting source log files..."

    if ! rm -f "${FILES_REF[@]}"; then

        log_error "Failed to delete one or more source log files."
        log_error "Recovered archive is stored successfully, but source files were NOT completely deleted."

        return 1

    fi

    log_info "Source files deleted successfully."
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

        if ! rm -f "$ARCHIVE_NAME"; then
            log_error "Failed to remove invalid archive: $ARCHIVE_NAME"
        fi

        return 1

    fi

    #-------------------------------------------------------
    # Move Archive
    #-------------------------------------------------------

    echo
    log_info "Moving archive..."

    if ! mv -f "$ARCHIVE_NAME" "$DEST_DIR/"; then
        log_error "Failed to move archive: $ARCHIVE_NAME"
        log_info "Source files will NOT be deleted."
        return 1
    fi

    if [[ ! -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then
        log_error "Archive is not present in destination after move: $DEST_DIR/$ARCHIVE_NAME"
        log_info "Source files will NOT be deleted."
        return 1
    fi

    log_info "Archive moved successfully."

    COMP_ARCHIVES_CREATED=$((COMP_ARCHIVES_CREATED + 1))

    #-------------------------------------------------------
    # Set Archive Permissions
    #-------------------------------------------------------

    if ! chmod 777 "$DEST_DIR/$ARCHIVE_NAME"; then
        log_warning "Failed to set permissions on archive: $DEST_DIR/$ARCHIVE_NAME"
        log_warning "Continuing anyway — archive is present and verified."
    fi

    #-------------------------------------------------------
    # Delete Source Files
    #-------------------------------------------------------

    echo
    log_warning "Deleting source log files..."

    if ! rm -f "${FILES_REF[@]}"; then
        log_error "Failed to delete one or more source log files."
        log_error "Archive was created successfully, but source files were NOT completely deleted."
        return 1
    fi

    log_info "Source files deleted successfully."
    log_info "Archive processing completed."

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

        handle_existing_destination_archive "$ARCHIVE_NAME" "$DEST_DIR" "$DATE" FILES

        return $?

    fi


    #-------------------------------------------------------
    # Recovery if Archive exist in the source
    #-------------------------------------------------------

    if [[ -f "$ARCHIVE_NAME" ]]; then

        recover_archive "$ARCHIVE_NAME" "$DEST_DIR" "$DATE" FILES

        return $?

    fi

    #-------------------------------------------------------
    # Create Archive if not in source or destination
    #-------------------------------------------------------

    if ! create_archive "$ARCHIVE_NAME" "$DEST_DIR" FILES; then
        return 1
    fi

}


#########################################################################
# Archive Function -> find_and_group_logs () and process_archive_by_date ()
#########################################################################

archive_component() {

    local COMPONENT="$1"
    local SRC_DIR="$2"
    local DEST_DIR="$3"

    local COMP_ARCHIVES_CREATED=0
    local COMP_ARCHIVES_RECOVERED=0
    local COMP_ARCHIVES_EXISTING=0

    local COMPONENT_PROCESS_FAILED=0
    local COMPONENT_PROCESS_STATUS=""
    local COMPONENT_PROCESS_RESULT=""

    local -A FILE_GROUPS=()

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

        COMPONENT_STATUS["$COMPONENT"]="FAILED"

        return 1
    fi

    log_info "Source directory found."

    #-------------------------------------------------------
    # Enter Source Directory
    #-------------------------------------------------------

    if ! pushd "$SRC_DIR" > /dev/null; then
        log_error "Failed to enter source directory: $SRC_DIR"

        COMPONENT_STATUS["$COMPONENT"]="FAILED"

        return 1
    fi

    # IMPORTANT:
    # No direct return is allowed from this point
    # until popd is executed.

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

    find_and_group_logs "$COMPONENT" FILE_GROUPS "$CUTOFF_DATE"

    #-------------------------------------------------------
    # Process eligible logs
    #-------------------------------------------------------

    if [[ ${#FILE_GROUPS[@]} -eq 0 ]]; then

        log_info "No eligible logs found."

        # Valid NO-OP condition.
        COMPONENT_PROCESS_STATUS=""
        COMPONENT_PROCESS_RESULT="No eligible logs"

    else

        # Eligible logs found.
        COMPONENT_PROCESS_STATUS="SUCCESS"
        COMPONENT_PROCESS_RESULT="Archive Processed"

        #---------------------------------------------------
        # Process each date
        #---------------------------------------------------

        while IFS= read -r DATE
        do

            if ! process_archive_by_date "$COMPONENT" "$DATE" "$DEST_DIR"; then

                echo
                log_error "Archive processing failed for date: $DATE"

                COMPONENT_PROCESS_FAILED=1
                COMPONENT_PROCESS_STATUS="FAILED"
                COMPONENT_PROCESS_RESULT="Archive failed"
            fi

        done < <(printf '%s\n' "${!FILE_GROUPS[@]}" | sort)

    fi

    echo
    log_info "Finished component : $COMPONENT"

    #-------------------------------------------------------
    # Restore previous working directory
    #-------------------------------------------------------

    if ! popd > /dev/null; then
        log_error "Failed to restore previous working directory."

        COMPONENT_PROCESS_FAILED=1
        COMPONENT_PROCESS_STATUS="FAILED"
    fi

    #-------------------------------------------------------
    # Store Component Summary
    #-------------------------------------------------------

    COMPONENT_CREATED["$COMPONENT"]="$COMP_ARCHIVES_CREATED"
    COMPONENT_RECOVERED["$COMPONENT"]="$COMP_ARCHIVES_RECOVERED"
    COMPONENT_EXISTING["$COMPONENT"]="$COMP_ARCHIVES_EXISTING"
    COMPONENT_STATUS["$COMPONENT"]="$COMPONENT_PROCESS_STATUS"
    COMPONENT_RESULT["$COMPONENT"]="$COMPONENT_PROCESS_RESULT"

    #-------------------------------------------------------
    # Update Global Archive Counters
    #-------------------------------------------------------

    ARCHIVES_CREATED=$((ARCHIVES_CREATED + COMP_ARCHIVES_CREATED))
    ARCHIVES_RECOVERED=$((ARCHIVES_RECOVERED + COMP_ARCHIVES_RECOVERED))
    ARCHIVES_EXISTING=$((ARCHIVES_EXISTING + COMP_ARCHIVES_EXISTING))

    #-------------------------------------------------------
    # Return Component Status
    #-------------------------------------------------------

    if [[ "$COMPONENT_PROCESS_FAILED" -ne 0 ]]; then
        log_error "Component processing failed : $COMPONENT"
        return 1
    fi

    # No eligible logs = valid NO-OP.
    if [[ -z "$COMPONENT_PROCESS_STATUS" ]]; then
        return 2
    fi

    return 0
}


############################################################
# Print Execution Summary
############################################################
print_execution_summary()
{
    echo
    echo "====================================================================================="
    echo "Master Log Archive Execution Summary"
    echo "====================================================================================="

    printf "%-30s %10s\n" "Metric" "Count"
    echo "-------------------------------------------------------------------------------------"

    printf "%-30s %10s\n" "Total Components"      "$TOTAL_COMPONENTS"
    printf "%-30s %10s\n" "Successful Components" "$SUCCESSFUL_COMPONENTS"
    printf "%-30s %10s\n" "Failed Components"     "$FAILED_COMPONENTS"
    printf "%-30s %10s\n" "Archives Created"      "$ARCHIVES_CREATED"
    printf "%-30s %10s\n" "Archives Recovered"    "$ARCHIVES_RECOVERED"
    printf "%-30s %10s\n" "Archives Existing"     "$ARCHIVES_EXISTING"

    echo "-------------------------------------------------------------------------------------"
    echo

    echo "Each Component Summary"
    echo "-------------------------------------------------------------------------------------"

    printf "%-30s %8s %11s %10s   %-17s %10s\n" \
        "Component" \
        "Created" \
        "Recovered" \
        "Existing" \
        "Result" \
        "Status"

    echo "-------------------------------------------------------------------------------------"

    for COMPONENT_ENTRY in "${COMPONENTS[@]}"
    do
        IFS='|' read -r COMPONENT SRC_DIR DEST_DIR <<< "$COMPONENT_ENTRY"

        printf "%-30s %8s %11s %10s   %-17s %10s\n" \
            "$COMPONENT" \
            "${COMPONENT_CREATED[$COMPONENT]:-0}" \
            "${COMPONENT_RECOVERED[$COMPONENT]:-0}" \
            "${COMPONENT_EXISTING[$COMPONENT]:-0}" \
            "${COMPONENT_RESULT[$COMPONENT]:-N/A}" \
            "${COMPONENT_STATUS[$COMPONENT]:-N/A}"
    done



    echo "-------------------------------------------------------------------------------------"
    echo
    printf "%-30s %10s\n" "Overall Status" "$OVERALL_STATUS"
}


############################################################
# Main Logic Start From Here -> archive_component() Funtion
############################################################

#-------------------------------------------------------
# 
#-------------------------------------------------------

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
    IFS='|' read -r COMPONENT SRC_DIR DEST_DIR <<< "$COMPONENT_ENTRY"

    TOTAL_COMPONENTS=$((TOTAL_COMPONENTS + 1))

    if archive_component "$COMPONENT" "$SRC_DIR" "$DEST_DIR"; then
        RC=0
    else
        RC=$?
    fi

    case "$RC" in

        0)
            SUCCESSFUL_COMPONENTS=$((SUCCESSFUL_COMPONENTS + 1))
            ;;

        2)
            # No eligible logs.
            # Do not count as success or failure.
            ;;

        *)
            FAILED_COMPONENTS=$((FAILED_COMPONENTS + 1))
            FAILED_COMPONENTS_LIST+="$COMPONENT, "
            ARCHIVE_FAILED=1
            ;;

    esac

done


END_TIME=$(date '+%Y-%m-%d %H:%M:%S')


############################################################
# Terminal Execution Summary (Redesigned)
############################################################

if [[ "$ARCHIVE_FAILED" -ne 0 ]]; then
    OVERALL_STATUS="FAILED"
else
    OVERALL_STATUS="SUCCESS"
fi

print_execution_summary


############################################################
# Prepare Teams Execution Summary
############################################################

prepare_teams_summary()
{

    #-------------------------------------------------------
    # Overall Status
    #-------------------------------------------------------

    if [[ "$OVERALL_STATUS" == "FAILED" ]]; then
        OVERALL_COLOR="attention"
        OVERALL_ICON="❌"
        OVERALL_TEXT="FAILED"
    else
        OVERALL_COLOR="good"
        OVERALL_ICON="✅"
        OVERALL_TEXT="SUCCESS"
    fi

    #-------------------------------------------------------
    # Failed Tile Color and Style
    #-------------------------------------------------------

    if [[ "$FAILED_COMPONENTS" -gt 0 ]]; then
        FAILED_COLOR="attention"
        FAILED_STYLE="attention"
    else
        FAILED_COLOR="default"
        FAILED_STYLE="emphasis"
    fi

    #-------------------------------------------------------
    # Build Component Rows
    #-------------------------------------------------------

    COMPONENT_ROWS=""

    for COMPONENT_ENTRY in "${COMPONENTS[@]}"
    do
        IFS='|' read -r COMPONENT SRC_DIR DEST_DIR <<< "$COMPONENT_ENTRY"

        STATUS="${COMPONENT_STATUS[$COMPONENT]:-}"

        case "$STATUS" in
            SUCCESS) STATUS_ICON="✅"; NAME_COLOR="default"   ;;
            FAILED)  STATUS_ICON="❌"; NAME_COLOR="attention" ;;
            *)       STATUS_ICON="➖"; NAME_COLOR="default"   ;;
        esac

        ROW=$(cat <<EOF
{
  "type": "Container",
  "separator": true,
  "spacing": "Large",
  "items": [
    {
      "type": "ColumnSet",
      "columns": [
        {
          "type": "Column",
          "width": "auto",
          "verticalContentAlignment": "Center",
          "items": [
            { "type": "TextBlock", "text": "${STATUS_ICON}", "size": "Large" }
          ]
        },
        {
          "type": "Column",
          "width": "stretch",
          "items": [
            { "type": "TextBlock", "text": "${COMPONENT}", "size": "Medium", "weight": "Bolder", "color": "${NAME_COLOR}", "wrap": true },
            { "type": "TextBlock", "text": "${COMPONENT_RESULT[$COMPONENT]:-No result}", "isSubtle": true, "spacing": "None", "wrap": true },
            { "type": "TextBlock", "text": "${COMPONENT_CREATED[$COMPONENT]:-0} created  ·  ${COMPONENT_RECOVERED[$COMPONENT]:-0} recovered  ·  ${COMPONENT_EXISTING[$COMPONENT]:-0} existing", "isSubtle": true, "spacing": "None", "wrap": true }
          ]
        }
      ]
    }
  ]
}
EOF
)

        if [[ -n "$COMPONENT_ROWS" ]]; then
            COMPONENT_ROWS+=","
        fi

        COMPONENT_ROWS+="$ROW"

    done

    #-------------------------------------------------------
    # Optional Failed Component Section (owns its leading comma)
    #-------------------------------------------------------

    FAILED_COMPONENT_SECTION=""

    if [[ "$FAILED_COMPONENTS" -gt 0 ]]; then

        FAILED_COMPONENT_SECTION=$(cat <<EOF
,
{
  "type": "TextBlock",
  "text": "**Failed Component List:** ${FAILED_COMPONENTS_LIST%, }",
  "wrap": true,
  "color": "attention",
  "spacing": "Medium"
}
EOF
)

    fi

}


############################################################
# Generate Teams Adaptive Card
############################################################

generate_teams_card()
{
    cat <<EOF
{
  "type": "message",
  "attachments": [
    {
      "contentType": "application/vnd.microsoft.card.adaptive",
      "content": {
        "\$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
        "type": "AdaptiveCard",
        "version": "1.5",
        "msteams": { "width": "Full" },
        "body": [

          {
            "type": "ColumnSet",
            "columns": [
              {
                "type": "Column",
                "width": "auto",
                "verticalContentAlignment": "Center",
                "items": [
                  { "type": "TextBlock", "text": "${OVERALL_ICON}", "size": "ExtraLarge" }
                ]
              },
              {
                "type": "Column",
                "width": "stretch",
                "items": [
                  { "type": "TextBlock", "text": "Archive Execution Summary", "size": "ExtraLarge", "weight": "Bolder", "wrap": true },
                  { "type": "TextBlock", "text": "Start at: ${START_TIME}  →  End at: ${END_TIME}", "isSubtle": true, "spacing": "None", "wrap": true }
                ]
              }
            ]
          },

          {
            "type": "ColumnSet",
            "spacing": "Large",
            "columns": [
              {
                "type": "Column",
                "width": "stretch",
                "style": "emphasis",
                "roundedCorners": true,
                "minHeight": "150px",
                "verticalContentAlignment": "Center",
                "items": [
                  { "type": "TextBlock", "text": "${ARCHIVES_CREATED}", "size": "ExtraLarge", "weight": "Bolder", "horizontalAlignment": "Center" },
                  { "type": "TextBlock", "text": "Total Archives Created", "isSubtle": true, "spacing": "Medium", "horizontalAlignment": "Center", "wrap": true }
                ]
              },
              {
                "type": "Column",
                "width": "stretch",
                "spacing": "Small",
                "style": "emphasis",
                "roundedCorners": true,
                "minHeight": "150px",
                "verticalContentAlignment": "Center",
                "items": [
                  { "type": "TextBlock", "text": "${ARCHIVES_RECOVERED}", "size": "ExtraLarge", "weight": "Bolder", "horizontalAlignment": "Center" },
                  { "type": "TextBlock", "text": "Total Archives Recovered", "isSubtle": true, "spacing": "Medium", "horizontalAlignment": "Center", "wrap": true }
                ]
              },
              {
                "type": "Column",
                "width": "stretch",
                "spacing": "Small",
                "style": "emphasis",
                "roundedCorners": true,
                "minHeight": "150px",
                "verticalContentAlignment": "Center",
                "items": [
                  { "type": "TextBlock", "text": "${ARCHIVES_EXISTING}", "size": "ExtraLarge", "weight": "Bolder", "horizontalAlignment": "Center" },
                  { "type": "TextBlock", "text": "Total Already Archived", "isSubtle": true, "spacing": "Medium", "horizontalAlignment": "Center", "wrap": true }
                ]
              },
              {
                "type": "Column",
                "width": "stretch",
                "spacing": "Small",
                "style": "${FAILED_STYLE}",
                "roundedCorners": true,
                "minHeight": "150px",
                "verticalContentAlignment": "Center",
                "items": [
                  { "type": "TextBlock", "text": "${FAILED_COMPONENTS}", "size": "ExtraLarge", "weight": "Bolder", "color": "${FAILED_COLOR}", "horizontalAlignment": "Center" },
                  { "type": "TextBlock", "text": "Total Failed Components", "isSubtle": true, "spacing": "Medium", "horizontalAlignment": "Center", "wrap": true }
                ]
              }
            ]
          },

          {
            "type": "TextBlock",
            "text": "Summary Per Components",
            "isSubtle": true,
            "weight": "Bolder",
            "spacing": "ExtraLarge"
          },

          {
            "type": "TextBlock",
            "text": "COMPONENTS  (${SUCCESSFUL_COMPONENTS}/${TOTAL_COMPONENTS} succeeded)",
            "isSubtle": true,
            "weight": "Bolder",
            "spacing": "Large"
          },

          ${COMPONENT_ROWS}
          ${FAILED_COMPONENT_SECTION}
          ,
          {
            "type": "TextBlock",
            "text": "Overall Status: ${OVERALL_TEXT}",
            "size": "Large",
            "weight": "Bolder",
            "color": "${OVERALL_COLOR}",
            "horizontalAlignment": "Center",
            "separator": true,
            "spacing": "Large",
            "wrap": true
          }

        ]
      }
    }
  ]
}
EOF
}


############################################################
# Send Summary to Teams
############################################################

send_notification()
{
    local JSON="$1"

    if [[ "$USE_PROXY" == "YES" ]]; then

        curl -sS -f \
            --connect-timeout 10 \
            --max-time 30 \
            --retry 2 \
            --retry-delay 2 \
            -x "$HTTPS_PROXY" \
            -H "Content-Type: application/json" \
            -d "$JSON" \
            "$WEBHOOK_URL" >/dev/null

    else

        curl -sS -f \
            --connect-timeout 10 \
            --max-time 30 \
            --retry 2 \
            --retry-delay 2 \
            -H "Content-Type: application/json" \
            -d "$JSON" \
            "$WEBHOOK_URL" >/dev/null

    fi
}


############################################################
# Build and Send Teams Summary
############################################################
prepare_teams_summary

ADAPTIVE_CARD=$(generate_teams_card)

if ! send_notification "$ADAPTIVE_CARD"; then
    TEAMS_NOTIFICATION_FAILED=1
    log_error "Teams notification failed, but archive execution result is preserved."
fi



#-------------------------------------------------------
# Show Exit Status
#-------------------------------------------------------

if [[ "$ARCHIVE_FAILED" -ne 0 ]]; then
    exit 1
fi

exit 0

```