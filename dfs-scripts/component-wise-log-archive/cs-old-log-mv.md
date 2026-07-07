```bash
#!/bin/bash

cd /home/scripts

compname24="cs"

# Archive logs older than this many days
START_FROM=2

CUTOFF_DATE=$(/bin/date --date="${START_FROM} days ago" "+%Y-%m-%d")

find -P /home/cs/logs/archive/ -type f -name "*.log.gz" \
| grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}' \
| sort -u \
| while read date
do
    # Skip cutoff date and anything newer
    if [[ "$date" < "$CUTOFF_DATE" || "$date" == "$CUTOFF_DATE" ]]; then
        echo "Processing logs for date: $date"
    else
        continue
    fi

    TARGET_DIR="/LOGS/app6/cs/$compname24-${HOSTNAME}-${INSTANCE_NAME}-$date"

    mkdir -p "$TARGET_DIR"

    find -P /home/cs/logs/archive/ -type f -iname "*$date*.log.gz" \
        | xargs -I {} mv {} "$TARGET_DIR"

    cd /LOGS/app6/cs/ || exit 1

    tar -cvzf "$compname24-${HOSTNAME}-${INSTANCE_NAME}-$date.tar.gz" \
        "$compname24-${HOSTNAME}-${INSTANCE_NAME}-$date"

    chmod 777 "$compname24-${HOSTNAME}-${INSTANCE_NAME}-$date.tar.gz"

    rm -rf "$compname24-${HOSTNAME}-${INSTANCE_NAME}-$date"

done

```