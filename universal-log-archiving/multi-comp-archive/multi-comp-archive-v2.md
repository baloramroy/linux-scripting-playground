```bash
#!/bin/bash
set -uo pipefail
# NOTE: intentionally NOT using 'set -e' here.
# With multiple components we do not want one component's failure to kill
# the whole run — each component is processed inside process_component(),
# which traps its own errors and returns a status instead of exiting.

#==================================================
# Configuration
#==================================================

DAYS=3

# Add / remove components here.
# Format per entry:  "component_name|src_dir|dest_dir"
# Each component can have a completely independent source and
# destination directory — nothing is auto-derived from a base path.
COMPONENTS=(
    "png-ngd-dc2-npsb-konaswitch01|/root/single-archiver/png|/root/single-archiver/LOGS/png"
    "png-ngd-dc2-npsb-konaswitch02|/root/single-archiver/png2|/root/single-archiver/LOGS/png2"
)

RUN_ID="$(date +%F_%H%M%S)"
LOCK_FILE="/tmp/multi_component_archive.lock"
LOG_DIR="/root/single-archiver/archive_script_logs"
LOG_FILE="${LOG_DIR}/multi_archive-$(date +%F).log"

#--------------------------------------------------
# Microsoft Teams Notification
#--------------------------------------------------

WEBHOOK_URL="https://default1-hVb4xMbs"

USE_PROXY="NO"
HTTPS_PROXY="http://10.210.10.173:4899"

#==================================================
# Execution tracking (GLOBAL / run-level totals)
#==================================================

START_TIME=$(date '+%F %T')
CUTOFF_DATE=$(date -d "$DAYS days ago" +%F)

TOTAL_LOG_FILES_PROCESSED=0
TOTAL_ARCHIVES_CREATED=0
TOTAL_ARCHIVES_RECOVERED=0
TOTAL_ARCHIVES_EXISTING=0

RUN_STATUS="SUCCESS"          # flips to FAILED if ANY component fails
COMPONENTS_OK=0
COMPONENTS_FAILED=0

# Per-component results, keyed by component name.
declare -A COMP_STATUS         # SUCCESS / FAILED
declare -A COMP_SUMMARY        # one-line human summary
declare -A COMP_CREATED
declare -A COMP_RECOVERED
declare -A COMP_EXISTING
declare -A COMP_PROCESSED

#==================================================
# Logging
#==================================================

mkdir -p "$LOG_DIR"

log()
{
    printf '%s %s\n' "$(date '+%F %T')" "$*" | tee -a "$LOG_FILE"
}

#==================================================
# Teams notification helpers
#==================================================

send_notification()
{
    local JSON="$1"

    if [[ "$USE_PROXY" == "YES" ]]; then
        curl -sS -f \
            -x "$HTTPS_PROXY" \
            -H "Content-Type: application/json" \
            -d "$JSON" \
            "$WEBHOOK_URL" >/dev/null
    else
        curl -sS -f \
            -H "Content-Type: application/json" \
            -d "$JSON" \
            "$WEBHOOK_URL" >/dev/null
    fi
}

# Builds one FactSet block (as JSON) summarizing a single component.
build_component_fact_block()
{
    local comp="$1"
    local icon color

    if [[ "${COMP_STATUS[$comp]}" == "SUCCESS" ]]; then
        icon="✓"; color="good"
    else
        icon="✕"; color="attention"
    fi

    cat <<EOF
{
    "type": "Container",
    "spacing": "Medium",
    "items": [
        {
            "type": "ColumnSet",
            "columns": [
                {
                    "type": "Column",
                    "width": "auto",
                    "items": [
                        { "type": "TextBlock", "text": "$icon", "weight": "Bolder", "color": "$color" }
                    ]
                },
                {
                    "type": "Column",
                    "width": "stretch",
                    "items": [
                        { "type": "TextBlock", "text": "$comp", "weight": "Bolder", "wrap": true },
                        { "type": "TextBlock", "text": "${COMP_SUMMARY[$comp]}", "size": "Small", "wrap": true, "isSubtle": true, "spacing": "None" }
                    ]
                }
            ]
        }
    ]
}
EOF
}

# Builds the entire Adaptive Card and sends it. Called exactly ONCE,
# after every component has been attempted (see on_exit at the bottom).
send_teams_alert()
{
    local END_TIME
    local GRAND_TOTAL
    local STATUS_ICON STATUS_COLOR STATUS_TEXT
    local COMP_BLOCKS="" comp first=1
    local JSON

    END_TIME=$(date '+%F %T')
    GRAND_TOTAL=$((TOTAL_ARCHIVES_CREATED + TOTAL_ARCHIVES_RECOVERED))

    if [[ "$RUN_STATUS" == "SUCCESS" ]]; then
        STATUS_ICON="✓"; STATUS_COLOR="good"; STATUS_TEXT="SUCCESS"
    else
        STATUS_ICON="✕"; STATUS_COLOR="attention"; STATUS_TEXT="FAILED"
    fi

    for entry in "${COMPONENTS[@]}"; do
        local comp block
        comp="${entry%%|*}"
        block=$(build_component_fact_block "$comp")
        if [[ $first -eq 1 ]]; then
            COMP_BLOCKS="$block"
            first=0
        else
            COMP_BLOCKS="${COMP_BLOCKS},${block}"
        fi
    done

    JSON=$(cat <<EOF
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
                                            { "type": "TextBlock", "text": "$STATUS_ICON", "size": "ExtraLarge", "weight": "Bolder" }
                                        ]
                                    },
                                    {
                                        "type": "Column",
                                        "width": "stretch",
                                        "items": [
                                            { "type": "TextBlock", "text": "Multi-Component Log Archive Execution", "size": "Large", "weight": "Bolder", "wrap": true },
                                            { "type": "TextBlock", "text": "$START_TIME → $END_TIME", "size": "Small", "isSubtle": true, "wrap": true, "spacing": "None" }
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
                            { "type": "TextBlock", "text": "Overall Summary", "weight": "Bolder", "size": "Medium" },
                            {
                                "type": "ColumnSet",
                                "spacing": "Medium",
                                "columns": [
                                    {
                                        "type": "Column", "width": "stretch",
                                        "items": [
                                            { "type": "TextBlock", "text": "${#COMPONENTS[@]}", "size": "ExtraLarge", "weight": "Bolder", "horizontalAlignment": "Center" },
                                            { "type": "TextBlock", "text": "Components", "size": "Small", "wrap": true, "horizontalAlignment": "Center" }
                                        ]
                                    },
                                    {
                                        "type": "Column", "width": "stretch",
                                        "items": [
                                            { "type": "TextBlock", "text": "$COMPONENTS_OK", "size": "ExtraLarge", "weight": "Bolder", "horizontalAlignment": "Center", "color": "good" },
                                            { "type": "TextBlock", "text": "Succeeded", "size": "Small", "wrap": true, "horizontalAlignment": "Center" }
                                        ]
                                    },
                                    {
                                        "type": "Column", "width": "stretch",
                                        "items": [
                                            { "type": "TextBlock", "text": "$COMPONENTS_FAILED", "size": "ExtraLarge", "weight": "Bolder", "horizontalAlignment": "Center", "color": "attention" },
                                            { "type": "TextBlock", "text": "Failed", "size": "Small", "wrap": true, "horizontalAlignment": "Center" }
                                        ]
                                    },
                                    {
                                        "type": "Column", "width": "stretch",
                                        "items": [
                                            { "type": "TextBlock", "text": "$TOTAL_LOG_FILES_PROCESSED", "size": "ExtraLarge", "weight": "Bolder", "horizontalAlignment": "Center" },
                                            { "type": "TextBlock", "text": "Log Files Processed", "size": "Small", "wrap": true, "horizontalAlignment": "Center" }
                                        ]
                                    }
                                ]
                            },
                            {
                                "type": "FactSet",
                                "spacing": "Medium",
                                "facts": [
                                    { "title": "Archives Created",   "value": "$TOTAL_ARCHIVES_CREATED" },
                                    { "title": "Archives Recovered", "value": "$TOTAL_ARCHIVES_RECOVERED" },
                                    { "title": "Already Existing",   "value": "$TOTAL_ARCHIVES_EXISTING" },
                                    { "title": "Total Archives Handled", "value": "$GRAND_TOTAL" },
                                    { "title": "Cutoff Date", "value": "$CUTOFF_DATE" }
                                ]
                            }
                        ]
                    },

                    {
                        "type": "Container",
                        "spacing": "Large",
                        "items": [
                            { "type": "TextBlock", "text": "Per-Component Breakdown", "weight": "Bolder", "size": "Medium" }
                        ]
                    },

                    ${COMP_BLOCKS},

                    {
                        "type": "Container",
                        "style": "$STATUS_COLOR",
                        "bleed": true,
                        "spacing": "Large",
                        "items": [
                            { "type": "TextBlock", "text": "Overall Status: $STATUS_TEXT", "weight": "Bolder", "size": "Medium", "horizontalAlignment": "Center", "wrap": true }
                        ]
                    }
                ]
            }
        }
    ]
}
EOF
)

    if ! send_notification "$JSON"; then
        log "WARNING: Teams notification failed. Archive execution status is unchanged."
    else
        log "Teams notification sent successfully."
    fi
}

#==================================================
# Core per-component archiving logic
# (same algorithm as the original single-component
#  script — grouped by date, verify, move, recover)
#==================================================

process_component()
{
    local COMPONENT="$1"
    local SRC_DIR="$2"
    local DEST_DIR="$3"

    local LOG_FILES_PROCESSED=0
    local ARCHIVES_CREATED=0
    local ARCHIVES_RECOVERED=0
    local ARCHIVES_EXISTING=0
    local RESULT="Execution completed successfully."

    echo
    log "=================================================="
    log "COMPONENT: $COMPONENT"
    log "=================================================="

    if [[ ! -d "$SRC_DIR" ]]; then
        log "ERROR: Source directory not found: $SRC_DIR"
        COMP_STATUS[$COMPONENT]="FAILED"
        COMP_SUMMARY[$COMPONENT]="Source directory not found: $SRC_DIR"
        return 1
    fi
    log "Source directory found."

#    if [[ ! -d "$DEST_DIR" ]]; then
#        log "Destination directory does not exist. Creating..."
#        mkdir -p "$DEST_DIR"
#    else
#        log "Destination directory exists."
#    fi

    #--------------------------------------------------------
    #Alternate Version
    #--------------------------------------------------------
    
    if [[ ! -d "$DEST_DIR" ]]; then
        log "ERROR: Destination directory does not exist: $DEST_DIR"
        COMP_STATUS[$COMPONENT]="FAILED"
        COMP_SUMMARY[$COMPONENT]="Destination directory does not exist: $DEST_DIR"
        return 1
    else
        log "Destination directory exists."
    fi

    log "Searching logs using filename date (older than or equal to $DAYS days)..."

    declare -A FILE_GROUPS=()
    shopt -s nullglob

    local file
    for file in "${SRC_DIR}/${COMPONENT}"-INST_*-*.log.gz
    do
        [[ -f "$file" ]] || continue

        local bn
        bn="$(basename "$file")"

        if [[ $bn =~ ([0-9]{4}-[0-9]{2}-[0-9]{2}) ]]; then
            local FILE_DATE="${BASH_REMATCH[1]}"
            if [[ "$FILE_DATE" < "$CUTOFF_DATE" || "$FILE_DATE" == "$CUTOFF_DATE" ]]; then
                FILE_GROUPS["$FILE_DATE"]+="$file"$'\n'
            fi
        else
            log "WARNING: Filename doesn't match expected date pattern, skipping: $bn"
        fi
    done
    shopt -u nullglob

    if [[ ${#FILE_GROUPS[@]} -eq 0 ]]; then
        log "No logs found older than or equal to $DAYS days."
        COMP_STATUS[$COMPONENT]="SUCCESS"
        COMP_SUMMARY[$COMPONENT]="No eligible logs found older than or equal to $DAYS days."
        COMP_CREATED[$COMPONENT]=0
        COMP_RECOVERED[$COMPONENT]=0
        COMP_EXISTING[$COMPONENT]=0
        COMP_PROCESSED[$COMPONENT]=0
        return 0
    fi

    local DATE
    while IFS= read -r DATE
    do
        local ARCHIVE_NAME="${COMPONENT}-${DATE}.tar.gz"
        local ARCHIVE_PATH="${SRC_DIR}/${ARCHIVE_NAME}"

        echo
        log "--------------------------------------------------"
        log "Processing Date : $DATE"
        log "Archive         : $ARCHIVE_NAME"
        log "--------------------------------------------------"

        if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then
            ARCHIVES_EXISTING=$((ARCHIVES_EXISTING + 1))
            log "Archive already exists in destination. Skipping..."
            continue
        fi

        local -a FILES
        mapfile -t FILES < <(printf '%s' "${FILE_GROUPS[$DATE]}" | sort -V)

        echo
        log "Files being archived (${#FILES[@]}):"
        printf '  %s\n' "${FILES[@]}" | tee -a "$LOG_FILE"

        # ---- Recovery case: archive already built in SRC_DIR ----
        if [[ -f "$ARCHIVE_PATH" ]]; then
            echo
            log "Archive already exists in source."

            local ARCHIVE_COUNT SRC_COUNT
            ARCHIVE_COUNT=$(tar -tzf "$ARCHIVE_PATH" | wc -l)
            SRC_COUNT=${#FILES[@]}

            if [[ "$ARCHIVE_COUNT" -ne "$SRC_COUNT" ]]; then
                log "ERROR: Verification failed for existing archive $ARCHIVE_NAME"
                log "       Archive has $ARCHIVE_COUNT files, expected $SRC_COUNT."
                COMP_STATUS[$COMPONENT]="FAILED"
                COMP_SUMMARY[$COMPONENT]="Verification failed for existing archive $ARCHIVE_NAME. Source logs were NOT deleted."
                return 1
            fi

            log "Verification OK ($ARCHIVE_COUNT files). Moving archive to destination..."
            mv -f "$ARCHIVE_PATH" "$DEST_DIR/"

            if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then
                chmod 777 "$DEST_DIR/$ARCHIVE_NAME"
                log "Removing source log files..."
                rm -f "${FILES[@]}"

                LOG_FILES_PROCESSED=$((LOG_FILES_PROCESSED + ${#FILES[@]}))
                ARCHIVES_RECOVERED=$((ARCHIVES_RECOVERED + 1))
                log "Recovery completed."
            else
                log "ERROR: Destination verification failed. Archive not found: $DEST_DIR/$ARCHIVE_NAME"
                COMP_STATUS[$COMPONENT]="FAILED"
                COMP_SUMMARY[$COMPONENT]="Destination archive verification failed. Source logs were NOT deleted."
                return 1
            fi

            continue
        fi

        # ---- Normal case: create a new archive ----
        echo
        log "Creating archive..."
        tar -czf "$ARCHIVE_PATH" -C "$SRC_DIR" $(printf '%s\n' "${FILES[@]}" | xargs -I{} basename {})
        log "Archive created."

        log "Verifying archive contents..."
        local ARCHIVE_COUNT SRC_COUNT
        ARCHIVE_COUNT=$(tar -tzf "$ARCHIVE_PATH" | wc -l)
        SRC_COUNT=${#FILES[@]}

        if [[ "$ARCHIVE_COUNT" -ne "$SRC_COUNT" ]]; then
            log "ERROR: Verification failed for $ARCHIVE_NAME"
            log "       Archive has $ARCHIVE_COUNT files, expected $SRC_COUNT."
            rm -f "$ARCHIVE_PATH"
            COMP_STATUS[$COMPONENT]="FAILED"
            COMP_SUMMARY[$COMPONENT]="Archive verification failed for $ARCHIVE_NAME. Source logs were NOT deleted."
            return 1
        fi
        log "Verification OK ($ARCHIVE_COUNT files matched)."

        mv -f "$ARCHIVE_PATH" "$DEST_DIR/"

        if [[ -f "$DEST_DIR/$ARCHIVE_NAME" ]]; then
            chmod 777 "$DEST_DIR/$ARCHIVE_NAME"
            log "Archive moved successfully. Deleting source log files..."
            rm -f "${FILES[@]}"

            LOG_FILES_PROCESSED=$((LOG_FILES_PROCESSED + ${#FILES[@]}))
            ARCHIVES_CREATED=$((ARCHIVES_CREATED + 1))
            log "Completed for $DATE"
        else
            log "ERROR: Failed to move archive to destination."
            COMP_STATUS[$COMPONENT]="FAILED"
            COMP_SUMMARY[$COMPONENT]="Failed to move archive $ARCHIVE_NAME to destination."
            return 1
        fi

    done < <(printf '%s\n' "${!FILE_GROUPS[@]}" | sort)

    # ---- Build this component's summary line ----
    if [[ "$ARCHIVES_CREATED" -gt 0 && "$ARCHIVES_RECOVERED" -gt 0 && "$ARCHIVES_EXISTING" -gt 0 ]]; then
        RESULT="$ARCHIVES_CREATED created, $ARCHIVES_RECOVERED recovered, $ARCHIVES_EXISTING already existed."
    elif [[ "$ARCHIVES_CREATED" -gt 0 && "$ARCHIVES_RECOVERED" -gt 0 ]]; then
        RESULT="$ARCHIVES_CREATED created and $ARCHIVES_RECOVERED recovered."
    elif [[ "$ARCHIVES_CREATED" -gt 0 && "$ARCHIVES_EXISTING" -gt 0 ]]; then
        RESULT="$ARCHIVES_CREATED created. $ARCHIVES_EXISTING already existed."
    elif [[ "$ARCHIVES_RECOVERED" -gt 0 && "$ARCHIVES_EXISTING" -gt 0 ]]; then
        RESULT="$ARCHIVES_RECOVERED recovered. $ARCHIVES_EXISTING already existed."
    elif [[ "$ARCHIVES_CREATED" -gt 0 ]]; then
        RESULT="$ARCHIVES_CREATED archive(s) created."
    elif [[ "$ARCHIVES_RECOVERED" -gt 0 ]]; then
        RESULT="$ARCHIVES_RECOVERED archive(s) recovered."
    elif [[ "$ARCHIVES_EXISTING" -gt 0 ]]; then
        RESULT="All eligible archives already existed. Processing skipped."
    else
        RESULT="No eligible logs found older than or equal to $DAYS days."
    fi

    COMP_STATUS[$COMPONENT]="SUCCESS"
    COMP_SUMMARY[$COMPONENT]="$RESULT"
    COMP_CREATED[$COMPONENT]=$ARCHIVES_CREATED
    COMP_RECOVERED[$COMPONENT]=$ARCHIVES_RECOVERED
    COMP_EXISTING[$COMPONENT]=$ARCHIVES_EXISTING
    COMP_PROCESSED[$COMPONENT]=$LOG_FILES_PROCESSED

    # NOTE: run-level totals are rolled up by the caller (main loop),
    # not here, to avoid double counting.
    return 0
}

#==================================================
# Exit trap — sends the ONE consolidated Teams alert
#==================================================

on_exit()
{
    send_teams_alert || true
}

trap on_exit EXIT

#==================================================
# Main
#==================================================

exec 200>"$LOCK_FILE"
flock -n 200 || {
    log "Another instance is running."
    exit 1
}

log "Starting multi-component archive run. Components: ${#COMPONENTS[@]}"

for ENTRY in "${COMPONENTS[@]}"; do

    IFS='|' read -r COMPONENT SRC_DIR DEST_DIR <<< "$ENTRY"

    if [[ -z "$COMPONENT" || -z "$SRC_DIR" || -z "$DEST_DIR" ]]; then
        log "ERROR: Malformed COMPONENTS entry, expected 'name|src_dir|dest_dir': $ENTRY"
        COMPONENTS_FAILED=$((COMPONENTS_FAILED + 1))
        RUN_STATUS="FAILED"
        continue
    fi

    if process_component "$COMPONENT" "$SRC_DIR" "$DEST_DIR"; then
        COMPONENTS_OK=$((COMPONENTS_OK + 1))
        # Roll totals for the "no eligible logs" early-return path too
        TOTAL_LOG_FILES_PROCESSED=$((TOTAL_LOG_FILES_PROCESSED + ${COMP_PROCESSED[$COMPONENT]:-0}))
        TOTAL_ARCHIVES_CREATED=$((TOTAL_ARCHIVES_CREATED + ${COMP_CREATED[$COMPONENT]:-0}))
        TOTAL_ARCHIVES_RECOVERED=$((TOTAL_ARCHIVES_RECOVERED + ${COMP_RECOVERED[$COMPONENT]:-0}))
        TOTAL_ARCHIVES_EXISTING=$((TOTAL_ARCHIVES_EXISTING + ${COMP_EXISTING[$COMPONENT]:-0}))
    else
        COMPONENTS_FAILED=$((COMPONENTS_FAILED + 1))
        RUN_STATUS="FAILED"
        log "Component $COMPONENT FAILED: ${COMP_SUMMARY[$COMPONENT]:-unknown error}"
    fi

done

echo
log "======================================================================"
log "All components processed. OK=$COMPONENTS_OK FAILED=$COMPONENTS_FAILED"
log "======================================================================"

if [[ "$RUN_STATUS" == "FAILED" ]]; then
    exit 1
fi

exit 0

```



