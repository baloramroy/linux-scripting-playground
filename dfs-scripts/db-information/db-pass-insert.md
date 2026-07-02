#!/bin/bash

declare -A PASSWORDS=(
    ["tsp"]="ENC(E0B19B0515E0D45B1FA0D483325370C0AEA51AA66BB2CE74)"
    ["bds"]="ENC(A0B19D41A826D1869BF5C68BAC07FA97112CDED555F40F81AF3CB639BE63CF6B)"
    ["ecs"]="ENC(8E77DEB15DF9B4190696C79E890E9E2957D2021A38BABDC8)"
    ["rms"]="ENC(647EC348BB94D5741A7AE66D7E08A72A116F0B3E308732AF)"
    ["rpg"]="gFnvkMWKLtBsdgshfgdfdghX"

)

BASE_DIR="/home"

for app in "${!PASSWORDS[@]}"
do
    FILE="${BASE_DIR}/${app}/cfg/application.properties"
    NEWPASS="${PASSWORDS[$app]}"

    if [[ ! -f "$FILE" ]]; then
        echo "[ERROR] File not found: $FILE"
        continue
    fi

    echo "Processing $FILE ..."

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