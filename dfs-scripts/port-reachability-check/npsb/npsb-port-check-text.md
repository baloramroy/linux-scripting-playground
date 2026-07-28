```bash
#!/bin/bash

###########################################
# Configuration
###########################################

SERVICE_NAME="GP"
HOST="10.10.10.10"
PORT="2233"

WEBHOOK_URL="https://your-powerautomate-webhook"

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
    else
        COLOR="00AA00"
    fi

    # Escape message for JSON
    MESSAGE_JSON=$(printf '%s' "$MESSAGE" | \
        python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))')

    JSON=$(cat <<EOF
{
  "@type": "MessageCard",
  "@context": "http://schema.org/extensions",
  "summary": "TCP Service Monitoring",
  "themeColor": "$COLOR",
  "title": "TCP Service Monitoring Alert",
  "text": $MESSAGE_JSON
}
EOF
)

    curl -s \
        -H "Content-Type: application/json" \
        -d "$JSON" \
        "$WEBHOOK_URL" >/dev/null
}

###########################################
# Send only if Status Changed
###########################################

if [ "$CURRENT_STATUS" != "$PREVIOUS_STATUS" ]; then

    if [ "$CURRENT_STATUS" = "DOWN" ]; then
        STATUS_ICON="🔴"
        ACTION=$(cat <<EOF

<b>Action Required</b>

• Verify network connectivity.
• If connectivity is confirmed and the issue persists, contact the service owner.
EOF
)
    else
        STATUS_ICON="🟢"
        ACTION=""
    fi

    MESSAGE=$(cat <<EOF
<b>Service Status Changed</b>

<b>Service :</b> ${SERVICE_NAME}

<b>Status  :</b> ${STATUS_ICON} ${CURRENT_STATUS}

<b>Response:</b> ${RESPONSE_TIME}

<b>Host    :</b> ${HOST}

<b>Port    :</b> ${PORT}

<b>Time    :</b> $(date)

${ACTION}
EOF
)

    send_notification "$MESSAGE"

    echo "$CURRENT_STATUS" > "$STATE_FILE"

fi
```