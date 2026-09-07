```bash
#!/bin/bash

OUTPUT_FILE="/home/scripts/redis_properties_all_components.txt"

COMPONENTS=(
    bds
    cms
    cp
    cs
    davs
    dfs
    drs
    dmscore
    extch
    extchSMS
    kms
    knotify
    kod
    map
    pcs
    tms
    tsp
    ecs
    rpg
    rms
    bkofc
    utilityservice
    ias
    mps
    spg
    npsb_recon
)

# Temporary file for summary
SUMMARY_FILE="/tmp/redis_summary_$$.tmp"

# Clear old output file
> "$OUTPUT_FILE"
> "$SUMMARY_FILE"

echo "Redis properties for all components" >> "$OUTPUT_FILE"
echo "Generated on: $(date)" >> "$OUTPUT_FILE"
echo "============================================================" >> "$OUTPUT_FILE"
echo >> "$OUTPUT_FILE"


for COMPONENT in "${COMPONENTS[@]}"; do

    CONFIG_FILE="/home/$COMPONENT/cfg/application.properties"

    echo "============================================================" >> "$OUTPUT_FILE"
    echo "COMPONENT: $COMPONENT" >> "$OUTPUT_FILE"
    echo "CONFIG:    $CONFIG_FILE" >> "$OUTPUT_FILE"
    echo "============================================================" >> "$OUTPUT_FILE"

    if [[ -f "$CONFIG_FILE" ]]; then

        # Check whether Redis configuration exists
        if grep -qi "redis" "$CONFIG_FILE"; then

            echo "YES" >> "$SUMMARY_FILE"

            # Print Redis properties
            grep -i "redis" "$CONFIG_FILE" >> "$OUTPUT_FILE"

        else

            echo "NO" >> "$SUMMARY_FILE"
            echo "No Redis properties found." >> "$OUTPUT_FILE"

        fi

    else

        echo "NO" >> "$SUMMARY_FILE"
        echo "CONFIG FILE NOT FOUND." >> "$OUTPUT_FILE"

    fi

    echo >> "$OUTPUT_FILE"

done


###############################################################################
# Summary
###############################################################################

echo >> "$OUTPUT_FILE"
echo >> "$OUTPUT_FILE"
echo "============================================================" >> "$OUTPUT_FILE"
echo "                    REDIS CONFIGURATION SUMMARY" >> "$OUTPUT_FILE"
echo "============================================================" >> "$OUTPUT_FILE"
echo >> "$OUTPUT_FILE"

printf "%-20s %-20s\n" "COMPONENT" "REDIS CONFIGURATION" >> "$OUTPUT_FILE"
printf "%-20s %-20s\n" "--------------------" "--------------------" >> "$OUTPUT_FILE"

COUNT=0

for COMPONENT in "${COMPONENTS[@]}"; do

    COUNT=$((COUNT + 1))

    STATUS=$(sed -n "${COUNT}p" "$SUMMARY_FILE")

    printf "%-20s %-20s\n" "$COMPONENT" "$STATUS" >> "$OUTPUT_FILE"

done

echo >> "$OUTPUT_FILE"
echo "============================================================" >> "$OUTPUT_FILE"

# Remove temporary file
rm -f "$SUMMARY_FILE"

echo "Completed."
echo "Output saved to: $OUTPUT_FILE"

```