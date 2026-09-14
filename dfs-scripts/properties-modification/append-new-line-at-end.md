```bash
#!/bin/bash

COMPONENTS=(
    ussdgwgp
    ussdgwblink
    ussdgwrobi
    ussdgwttalk
    knotifydmz
    outboundproxy
)

DATE=$(date +%d%m%Y)

for COMPONENT in "${COMPONENTS[@]}"
do
    FILE="/home/${COMPONENT}/cfg/application.properties"
    BACKUP="${FILE}.bkp.${DATE}"

    if [[ -f "$FILE" ]]; then

        # Create backup before modification
        cp -p "$FILE" "$BACKUP"
        echo "Backup created: $BACKUP"

        # Append configuration block
        cat >> "$FILE" <<'EOF'

#################13082026#######################
#DMS
http.client.route.dmscore.location=dmscore.kpp.com:10911
http.client.route.dmscore.context-path=/dms-portal-core-1.0
http.client.route.dmscore.max-per-route-connections=20


########### Face Verification ####################
ussd.face-verification.enabled=false
ussd.face-verification.endpoint-list=


##########20260906##########
mno.airtel = CIRKLE
EOF

        echo "Updated: $FILE"
        echo

    else
        echo "File not found: $FILE"
        echo
    fi
done
```