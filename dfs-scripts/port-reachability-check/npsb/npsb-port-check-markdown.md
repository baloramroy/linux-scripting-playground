```bash
#!/bin/bash

###########################################
# Configuration
###########################################

SERVICE_NAME="gp"
HOST="192.168.231.213"
PORT="22"

WEBHOOK_URL="https://default1fdbc3071c9d4e679cc689ac4325a3.17.environment.api.powerplatform.com:443/powerautomate/automations/direct/cu/02/workflows/ea4f98fcb23545f793cae2a9c90c37b4/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=8lQSDr_3NWCILFTMOmoPW3OtyQxExTj67pNFMxvAsas"

STATE_FILE="/tmp/${SERVICE_NAME}_service.state"
TIMEOUT=5

###########################################
# Check Service
###########################################

START_TIME=$(date +%s.%N)

if timeout ${TIMEOUT} bash -c "</dev/tcp/${HOST}/${PORT}" 2>/dev/null
then
    CURRENT_STATUS="UP"
else
    CURRENT_STATUS="DOWN"
fi

END_TIME=$(date +%s.%N)

RESPONSE_TIME=$(awk "BEGIN {printf \"%.2fs\", ${END_TIME}-${START_TIME}}")

###########################################
# Read Previous Status
###########################################

if [ -f "$STATE_FILE" ]; then
    PREVIOUS_STATUS=$(cat "$STATE_FILE")
else
    PREVIOUS_STATUS="UNKNOWN"
fi

###########################################
# Send Teams Notification
###########################################

send_notification() {

    MESSAGE="$1"

    if [ "$CURRENT_STATUS" = "DOWN" ]; then
        COLOR="FF0000"
        STATUS_HTML="<font color='red'><b>DOWN</b></font>"
    else
        COLOR="00AA00"
        STATUS_HTML="<font color='green'><b>UP</b></font>"
    fi

    # Escape message for JSON
    MESSAGE_JSON=$(printf '%s' "$MESSAGE" | \
        python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))')

    JSON=$(cat <<EOF
{
  "@type":"MessageCard",
  "@context":"http://schema.org/extensions",
  "summary":"Service Status",
  "themeColor":"$COLOR",
  "title":"Service Status Changed",
  "text":$MESSAGE_JSON
}
EOF
)
    echo "$JSON"

    curl -v \
        -H "Content-Type: application/json" \
        -d "$JSON" \
        "$WEBHOOK_URL" >/dev/null
}

###########################################
# Send only if Status Changed
###########################################

if [ "$CURRENT_STATUS" != "$PREVIOUS_STATUS" ]; then

    if [ "$CURRENT_STATUS" = "DOWN" ]; then
        STATUS="🔴 DOWN"
    else
        STATUS="🟢 UP"
    fi

    MESSAGE=$(cat <<EOF
**Service Status Changed**

| Service | Status | Response | Host | Port |
|---------|--------|----------|------|------|
| ${SERVICE_NAME} | ${STATUS} | ${RESPONSE_TIME} | ${HOST} | ${PORT} |

**Time:** $(date)
EOF
)

    send_notification "$MESSAGE"

    echo "$CURRENT_STATUS" > "$STATE_FILE"

fi

```