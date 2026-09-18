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
# Microsoft Teams Notification
#--------------------------------------------------

WEBHOOK_URL="https://default1-hVb4xMbs"

USE_PROXY="NO"
HTTPS_PROXY="http://10.210.10.173:4899"


#--------------------------------------------------
# Execution Tracking
#--------------------------------------------------

START_TIME=$(date '+%F %T')

LOG_FILES_PROCESSED=0
ARCHIVES_CREATED=0
ARCHIVES_RECOVERED=0
ARCHIVES_EXISTING=0

COMPONENT_STATUS="SUCCESS"
COMPONENT_RESULT="Execution completed successfully."


#--------------------------------------------------
# Calculate Cutoff Date
#--------------------------------------------------

CUTOFF_DATE=$(date -d "$DAYS days ago" +%F)


#--------------------------------------------------
# Logging Setup
#--------------------------------------------------

mkdir -p "$LOG_DIR"


#--------------------------------------------------
# Logging
#--------------------------------------------------

log()
{
    printf '%s %s\n' "$(date '+%F %T')" "$*" | tee -a "$LOG_FILE"
}


#==================================================
# MAIN ARCHIVING LOGIC
#==================================================

main()
{

    #--------------------------------------------------
    # Acquire Lock
    #--------------------------------------------------

    exec 200>"$LOCK_FILE"

    flock -n 200 || {
        log "Another instance is running."
        exit 1
    }


    #--------------------------------------------------
    # Validate Source Directory
    #--------------------------------------------------
    echo
    log "INFO: Checking Source Directory......"

    if [[ ! -d "$SRC_DIR" ]]; then

        log "ERROR: Source directory not found: $SRC_DIR"

        exit 1

    fi

    log "Source directory found."

    cd "$SRC_DIR"


    #--------------------------------------------------
    # Ensure Destination Directory Exists
    #--------------------------------------------------
    echo
    log "INFO: Checking Destination Directory......"

    if [[ ! -d "$DEST_DIR" ]]; then

        log "Destination directory does not exist. Creating..."

        mkdir -p "$DEST_DIR"

    else
        log "Destination directory exists."
    fi


#    if [[ ! -d "$DEST_DIR" ]]; then
#
#        COMPONENT_RESULT="Destination directory does not exist. Create first...."
#
#        log "Destination directory does not exist. Creating..."
#
#        #mkdir -p "$DEST_DIR"
#        exit 1
#
#    else
#        log "Destination directory exists."
#    fi
    


    #--------------------------------------------------
    # Search Logs
    #--------------------------------------------------
    
    echo
    log "Searching logs using filename date (older than or equal to $DAYS days)..."


    #--------------------------------------------------
    # Group Files by Date
    #--------------------------------------------------

    declare -A FILE_GROUPS=()

    shopt -s nullglob


    for file in "${COMPONENT}"-INST_*-*.log.gz
    do

        [[ -f "$file" ]] || continue


        #----------------------------------------------
        # Extract YYYY-MM-DD from Filename
        #----------------------------------------------

        if [[ "$file" =~ ([0-9]{4}-[0-9]{2}-[0-9]{2}) ]]; then

            FILE_DATE="${BASH_REMATCH[1]}"


            #------------------------------------------
            # YYYY-MM-DD Allows Lexicographical Compare
            #------------------------------------------

            if [[ "$FILE_DATE" < "$CUTOFF_DATE" || "$FILE_DATE" == "$CUTOFF_DATE" ]]; then

                # Store each filename on a separate line
                FILE_GROUPS["$FILE_DATE"]+="$file"$'\n'

            fi

        else

            log "WARNING: Filename doesn't match expected date pattern, skipping: $file"

        fi

    done


    #--------------------------------------------------
    # Check for Eligible Files
    #--------------------------------------------------

    if [[ ${#FILE_GROUPS[@]} -eq 0 ]]; then

        COMPONENT_RESULT="No eligible logs found older than or equal to $DAYS days."

        log "No logs found older than or equal to $DAYS days."

        exit 0

    fi


    #--------------------------------------------------
    # Process Each Date
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
        # Skip if Archive Already Exists in Destination
        #----------------------------------------------

        if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then

            ARCHIVES_EXISTING=$((ARCHIVES_EXISTING + 1))

            log "Archive already exists in destination."
            log "Skipping..."

            continue

        fi


        #----------------------------------------------
        # Read Files for This Date
        #----------------------------------------------

        mapfile -t FILES < <(
            printf '%s' "${FILE_GROUPS[$DATE]}" | sort -V
        )


        echo

        log "Files being archived (${#FILES[@]}):"

        echo


        #------------------------------------------------
        # File Listing Intentionally Has NO Timestamp
        #------------------------------------------------

        printf '  %s\n' "${FILES[@]}" | tee -a "$LOG_FILE"


        #----------------------------------------------
        # Recovery Case
        #----------------------------------------------

        if [[ -f "$ARCHIVE_NAME" ]]; then

            echo

            log "Archive already exists in source."


            #------------------------------------------
            # Verify Existing Archive Before Trusting It
            #------------------------------------------

            ARCHIVE_COUNT=$(tar -tzf "$ARCHIVE_NAME" | wc -l)

            SRC_COUNT=${#FILES[@]}


            if [[ "$ARCHIVE_COUNT" -ne "$SRC_COUNT" ]]; then

                log "ERROR: Verification failed for existing archive $ARCHIVE_NAME"
                log "       Archive has $ARCHIVE_COUNT files, expected $SRC_COUNT."

                COMPONENT_RESULT="Verification failed for existing archive $ARCHIVE_NAME. Source logs were NOT deleted."

                exit 1

            fi

            log "Verification OK ($ARCHIVE_COUNT files). Moving archive to destination..."

            mv -f "$ARCHIVE_NAME" "$DEST_DIR/"

            #------------------------------------------
            # Verify Archive Move to Destination
            #------------------------------------------

            if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then

                log "Archive move to destination successfully."
                log "Archive found: $DEST_DIR/$ARCHIVE_NAME"

                chmod 777 "$DEST_DIR/$ARCHIVE_NAME"

                log "Removing source log files..."

                rm -f "${FILES[@]}"

                LOG_FILES_PROCESSED=$((LOG_FILES_PROCESSED + ${#FILES[@]}))
                ARCHIVES_RECOVERED=$((ARCHIVES_RECOVERED + 1))

                log "Recovery completed for $DATE"

            else

                log "ERROR: Destination verification failed."
                log "       Archive not found: $DEST_DIR/$ARCHIVE_NAME"
                log "       Source log files will NOT be deleted."

                COMPONENT_RESULT="Destination archive verification failed. Source logs were NOT deleted."

                exit 1

            fi

            continue

        fi


        #----------------------------------------------
        # Create Archive
        #----------------------------------------------

        echo

        log "Creating archive..."

        if tar -czf "$ARCHIVE_NAME" "${FILES[@]}"; then

            log "Archive created."

        else

            log "ERROR: Failed to create archive: $ARCHIVE_NAME"

            COMPONENT_RESULT="Failed to create archive $ARCHIVE_NAME. Source log files were NOT deleted."

            rm -f "$ARCHIVE_NAME"

            exit 1

        fi


        #----------------------------------------------
        # Verify Archive
        #----------------------------------------------

        log "Verifying archive contents..."

        ARCHIVE_COUNT=$(tar -tzf "$ARCHIVE_NAME" | wc -l)

        SRC_COUNT=${#FILES[@]}

        if [[ "$ARCHIVE_COUNT" -ne "$SRC_COUNT" ]]; then

            log "ERROR: Verification failed for $ARCHIVE_NAME"
            log "       Archive has $ARCHIVE_COUNT files, expected $SRC_COUNT."

            rm -f "$ARCHIVE_NAME"

            COMPONENT_RESULT="Archive verification failed for $ARCHIVE_NAME. Source logs were NOT deleted."

            exit 1

        fi


        log "Verification OK ($ARCHIVE_COUNT files matched)."


        #----------------------------------------------
        # Move Archive
        #----------------------------------------------

        mv -f "$ARCHIVE_NAME" "$DEST_DIR/"

        if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then

            log "Archive move to destination successfully."
            log "Archive found: $DEST_DIR/$ARCHIVE_NAME"

            chmod 777 "$DEST_DIR/$ARCHIVE_NAME"

            log "Deleting source log files..."

            rm -f "${FILES[@]}"

            LOG_FILES_PROCESSED=$((LOG_FILES_PROCESSED + ${#FILES[@]}))
            ARCHIVES_CREATED=$((ARCHIVES_CREATED + 1))

            log "Processing Completed for $DATE"

        else
            log "ERROR: Destination verification failed."
            log "       Archive not found: $DEST_DIR/$ARCHIVE_NAME"
            log "       Source log files will NOT be deleted."

            COMPONENT_RESULT="Archive couldn't found in the Destination. Source logs were NOT deleted."

            exit 1

        fi


    done < <(
        printf '%s\n' "${!FILE_GROUPS[@]}" | sort
    )


    #--------------------------------------------------
    # Archive Processing Completed
    #--------------------------------------------------

    echo

    log "========================================"
    log "All eligible logs processed successfully."
    log "========================================"

}


#==================================================
# TEAMS NOTIFICATION
#==================================================


#--------------------------------------------------
# Send Teams Notification
#--------------------------------------------------

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


#--------------------------------------------------
# Build Microsoft Teams Adaptive Card
#--------------------------------------------------

build_teams_card()
{
    cat <<EOF
{
    "type": "message",
    "attachments": [
        {
            "contentType": "application/vnd.microsoft.card.adaptive",
            "contentUrl": null,
            "content": {
                "\$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
                "type": "AdaptiveCard",
                "version": "1.5",
                "body": [
                    {
                        "type": "Container",
                        "style": "emphasis",
                        "bleed": true,
                        "items": [
                            {
                                "type": "ColumnSet",
                                "columns": [
                                    {
                                        "type": "Column",
                                        "width": "auto",
                                        "items": [
                                            {
                                                "type": "TextBlock",
                                                "text": "$STATUS_ICON",
                                                "size": "ExtraLarge",
                                                "weight": "Bolder"
                                            }
                                        ]
                                    },
                                    {
                                        "type": "Column",
                                        "width": "stretch",
                                        "items": [
                                            {
                                                "type": "TextBlock",
                                                "text": "Log Archive Execution",
                                                "size": "Large",
                                                "weight": "Bolder",
                                                "wrap": true
                                            },
                                            {
                                                "type": "TextBlock",
                                                "text": "$START_TIME → $END_TIME",
                                                "size": "Small",
                                                "isSubtle": true,
                                                "wrap": true,
                                                "spacing": "None"
                                            }
                                        ]
                                    }
                                ]
                            }
                        ]
                    },

                    {
                        "type": "Container",
                        "spacing": "Large",
                        "items": [
                            {
                                "type": "TextBlock",
                                "text": "Component",
                                "weight": "Bolder",
                                "size": "Medium"
                            },
                            {
                                "type": "TextBlock",
                                "text": "$COMPONENT",
                                "wrap": true,
                                "spacing": "Small"
                            }
                        ]
                    },

                    {
                        "type": "Container",
                        "spacing": "Large",
                        "items": [
                            {
                                "type": "TextBlock",
                                "text": "Execution Summary",
                                "weight": "Bolder",
                                "size": "Medium"
                            },

                            {
                                "type": "ColumnSet",
                                "spacing": "Medium",
                                "columns": [
                                    {
                                        "type": "Column",
                                        "width": "stretch",
                                        "items": [
                                            {
                                                "type": "Container",
                                                "style": "emphasis",
                                                "spacing": "None",
                                                "items": [
                                                    {
                                                        "type": "TextBlock",
                                                        "text": "$LOG_FILES_PROCESSED",
                                                        "size": "ExtraLarge",
                                                        "weight": "Bolder",
                                                        "horizontalAlignment": "Center",
                                                        "spacing": "None"
                                                    },
                                                    {
                                                        "type": "TextBlock",
                                                        "text": "Log Files Processed",
                                                        "size": "Small",
                                                        "wrap": true,
                                                        "horizontalAlignment": "Center",
                                                        "spacing": "Small"
                                                    }
                                                ]
                                            }
                                        ]
                                    },

                                    {
                                        "type": "Column",
                                        "width": "stretch",
                                        "items": [
                                            {
                                                "type": "Container",
                                                "style": "emphasis",
                                                "spacing": "None",
                                                "items": [
                                                    {
                                                        "type": "TextBlock",
                                                        "text": "$ARCHIVES_CREATED",
                                                        "size": "ExtraLarge",
                                                        "weight": "Bolder",
                                                        "horizontalAlignment": "Center",
                                                        "spacing": "None"
                                                    },
                                                    {
                                                        "type": "TextBlock",
                                                        "text": "Archives Created",
                                                        "size": "Small",
                                                        "wrap": true,
                                                        "horizontalAlignment": "Center",
                                                        "spacing": "Small"
                                                    }
                                                ]
                                            }
                                        ]
                                    },

                                    {
                                        "type": "Column",
                                        "width": "stretch",
                                        "items": [
                                            {
                                                "type": "Container",
                                                "style": "emphasis",
                                                "spacing": "None",
                                                "items": [
                                                    {
                                                        "type": "TextBlock",
                                                        "text": "$ARCHIVES_RECOVERED",
                                                        "size": "ExtraLarge",
                                                        "weight": "Bolder",
                                                        "horizontalAlignment": "Center",
                                                        "spacing": "None"
                                                    },
                                                    {
                                                        "type": "TextBlock",
                                                        "text": "Archives Recovered",
                                                        "size": "Small",
                                                        "wrap": true,
                                                        "horizontalAlignment": "Center",
                                                        "spacing": "Small"
                                                    }
                                                ]
                                            }
                                        ]
                                    },

                                    {
                                        "type": "Column",
                                        "width": "stretch",
                                        "items": [
                                            {
                                                "type": "Container",
                                                "style": "emphasis",
                                                "spacing": "None",
                                                "items": [
                                                    {
                                                        "type": "TextBlock",
                                                        "text": "$ARCHIVES_EXISTING",
                                                        "size": "ExtraLarge",
                                                        "weight": "Bolder",
                                                        "horizontalAlignment": "Center",
                                                        "spacing": "None"
                                                    },
                                                    {
                                                        "type": "TextBlock",
                                                        "text": "Already Existing",
                                                        "size": "Small",
                                                        "wrap": true,
                                                        "horizontalAlignment": "Center",
                                                        "spacing": "Small"
                                                    }
                                                ]
                                            }
                                        ]
                                    }
                                ]
                            },

                            {
                                "type": "FactSet",
                                "spacing": "Large",
                                "facts": [
                                    {
                                        "title": "Total Archives Handled",
                                        "value": "$TOTAL_ARCHIVES"
                                    },
                                    {
                                        "title": "Cutoff Date",
                                        "value": "$CUTOFF_DATE"
                                    }
                                ]
                            }
                        ]
                    },

                    {
                        "type": "Container",
                        "spacing": "Large",
                        "items": [
                            {
                                "type": "TextBlock",
                                "text": "Remarks",
                                "weight": "Bolder",
                                "size": "Medium"
                            },
                            {
                                "type": "TextBlock",
                                "text": "$COMPONENT_RESULT",
                                "wrap": true,
                                "spacing": "Small"
                            }
                        ]
                    },

                    {
                        "type": "Container",
                        "style": "$STATUS_COLOR",
                        "bleed": true,
                        "spacing": "Large",
                        "items": [
                            {
                                "type": "TextBlock",
                                "text": "Overall Status: $STATUS_TEXT",
                                "weight": "Bolder",
                                "size": "Medium",
                                "horizontalAlignment": "Center",
                                "wrap": true
                            }
                        ]
                    }
                ]
            }
        }
    ]
}
EOF
}


#--------------------------------------------------
# Build and Send Teams Adaptive Card
#--------------------------------------------------

send_teams_alert()
{
    local END_TIME
    local TOTAL_ARCHIVES
    local STATUS_ICON
    local STATUS_COLOR
    local STATUS_TEXT
    local JSON
    local RESULT_PARTS=()


    END_TIME=$(date '+%F %T')


    #--------------------------------------------------
    # Calculate Total Archives Handled
    #--------------------------------------------------

    TOTAL_ARCHIVES=$((ARCHIVES_CREATED + ARCHIVES_RECOVERED))


    #--------------------------------------------------
    # Build Execution Result
    #--------------------------------------------------

    if [[ "$COMPONENT_STATUS" == "SUCCESS" ]]; then

        if [[ "$ARCHIVES_CREATED" -gt 0 ]]; then
            RESULT_PARTS+=("$ARCHIVES_CREATED archive(s) created")
        fi

        if [[ "$ARCHIVES_RECOVERED" -gt 0 ]]; then
            RESULT_PARTS+=("$ARCHIVES_RECOVERED archive(s) recovered")
        fi

        if [[ "$ARCHIVES_EXISTING" -gt 0 ]]; then
            RESULT_PARTS+=("$ARCHIVES_EXISTING already existed and were skipped")
        fi

        if [[ "${#RESULT_PARTS[@]}" -gt 0 ]]; then

            COMPONENT_RESULT="$(IFS=', '; echo "${RESULT_PARTS[*]}")."

        else

            COMPONENT_RESULT="No eligible logs found older than or equal to $DAYS days."

        fi

    fi


    #--------------------------------------------------
    # Determine Status Display
    #--------------------------------------------------

    if [[ "$COMPONENT_STATUS" == "SUCCESS" ]]; then

        STATUS_ICON="✓"
        STATUS_COLOR="good"
        STATUS_TEXT="SUCCESS"

    else

        STATUS_ICON="✕"
        STATUS_COLOR="attention"
        STATUS_TEXT="FAILED"

    fi


    #--------------------------------------------------
    # Build Adaptive Card
    #--------------------------------------------------

    JSON=$(build_teams_card)


    #--------------------------------------------------
    # Send Notification
    #--------------------------------------------------

    if ! send_notification "$JSON"; then

        log "WARNING: Teams notification failed. Archive execution status is unchanged."

    else

        log "Teams notification sent successfully."

    fi
}


#==================================================
# EXIT HANDLER
#==================================================


#--------------------------------------------------
# Send Teams Alert When Script Exits
#--------------------------------------------------

on_exit()
{
    local EXIT_CODE=$?


    if [[ "$EXIT_CODE" -eq 0 ]]; then

        COMPONENT_STATUS="SUCCESS"

    else

        COMPONENT_STATUS="FAILED"


        if [[ -z "$COMPONENT_RESULT" || "$COMPONENT_RESULT" == "Execution completed successfully." ]]; then

            COMPONENT_RESULT="Execution failed. Check the archive log for details."

        fi

    fi

    send_teams_alert || true
    return "$EXIT_CODE"
}


#==================================================
# ENTRY POINT
#==================================================

trap on_exit EXIT

main "$@"
```