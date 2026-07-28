```bash
#!/bin/bash

###########################################
# Configuration
###########################################
SERVER_NAME=$(hostname)
SERVICE_NAME="NPSB_Switch"
#HOST="10.210.10.37"
HOST="10.237.16.141"
PORT="52380"

#WEBHOOK_URL=""

WEBHOOK_URL="https://default1-hVb4xMbs"

SCRIPT_DIR="/home/script/npsb-swich-check"
STATE_DIR="${SCRIPT_DIR}/state-file"
STATE_FILE="${STATE_DIR}/${SERVICE_NAME}.state"
#STATE_FILE="/tmp/${SERVICE_NAME}_service.state"
TIMEOUT=5

###########################################
# Proxy Configuration
###########################################

HTTP_PROXY="http://10.210.10.173:4899"
HTTPS_PROXY="http://10.210.10.173:4899"

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
  "title":"Bangladesh Bank NPSB Switch Monitoring Alert",
  "text":$MESSAGE_JSON
}
EOF
)

    curl -s \
        -x "$HTTPS_PROXY" \
        -H "Content-Type: application/json" \
        -d "$JSON" \
        "$WEBHOOK_URL" >/dev/null
}

###########################################
# Send only if Status Changed
###########################################

if [ "$CURRENT_STATUS" != "$PREVIOUS_STATUS" ]; then

    if [ "$CURRENT_STATUS" = "DOWN" ]; then
        STATUS="<font color='red'><b>DOWN</b></font>"
    else
        STATUS="<font color='green'><b>UP</b></font>"
    fi

    MESSAGE=$(cat <<EOF
<b>Server:</b> ${SERVER_NAME}<br>

<b>Time:</b> $(date "+%a %b %d %I:%M:%S %p")<br>

<br>

<table border="1" cellspacing="0" cellpadding="10" width="900">

<tr>
<th width="180" align="left">Service</th>
<th width="120" align="left">Status</th>
<th width="120" align="left">Response</th>
<th width="350" align="left">Host</th>
<th width="130" align="left">Port</th>
</tr>

<tr>
<td width="180">${SERVICE_NAME}</td>
<td width="120">${STATUS}</td>
<td width="120">${RESPONSE_TIME}</td>
<td width="350">${HOST}</td>
<td width="130">${PORT}</td>
</tr>

</table>
<br>
EOF
)
    send_notification "$MESSAGE"

    echo "$CURRENT_STATUS" > "$STATE_FILE"

fi

```