#!/bin/bash

declare -A PASSWORDS=(
    ["bkofc"]="ENC(E0B19B0515E0D45B05F43F7E2E79E14BF62FB03E7942B3283C26199D8A114A24)"
    ["utilityservice"]="VS5G9PSjTTddfdy8z8kUTS"
)

BASE_DIR="/home"
TODAY=$(date +%d%m%Y)

for app in "${!PASSWORDS[@]}"
do
    FILE="${BASE_DIR}/${app}/cfg/application.properties"
    BACKUP="${FILE}.bkp.${TODAY}"
    NEWPASS="${PASSWORDS[$app]}"

    if [[ ! -f "$FILE" ]]; then
        echo "[ERROR] File not found: $FILE"
        continue
    fi

    echo "Processing $FILE ..."

    # Take backup (only once per day)
    if [[ ! -f "$BACKUP" ]]; then
        cp -p "$FILE" "$BACKUP"

        if [[ $? -ne 0 ]]; then
            echo "[ERROR] Failed to create backup: $BACKUP"
            echo "[ERROR] Skipping $FILE"
            continue
        fi

        echo "Backup created: $BACKUP"
    else
        echo "Backup already exists: $BACKUP"
    fi

    # Skip if already updated
    if grep -q "^spring.datasource.password=${NEWPASS}$" "$FILE"; then
        echo "Already updated. Skipping."
        continue
    fi

    awk -v newpass="$NEWPASS" '
    BEGIN {done=0}

    {
        if (!done && $0 ~ /^spring\.datasource\.password=/) {
            print "#" $0
            print "spring.datasource.password=" newpass
            done=1
        } else {
            print
        }
    }
    ' "$FILE" > "${FILE}.tmp" && mv "${FILE}.tmp" "$FILE"

    echo "Done."
done