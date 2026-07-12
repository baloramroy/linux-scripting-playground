#!/bin/bash

###############################################################################
# Fake Log Generator
#
# Usage:
#   ./fake-log-generator.sh <component> <hostname> <instances> <days>
#
# Example:
#   ./fake-log-generator.sh dmscore nagad-app7 2 4
#
# This creates fake logs from today back to 4 days ago.
###############################################################################

COMPONENT="${1:-dmscore}"
HOSTNAME="${2:-nagad-app7}"
INSTANCES="${3:-1}"
DAYS="${4:-3}"

OUTPUT_DIR="./fake_logs"

###############################################################################
# Validation
###############################################################################

if [[ $# -ne 4 ]]; then
    echo "Usage:"
    echo "  $0 <component> <hostname> <instances> <days>"
    echo
    echo "Example:"
    echo "  $0 dmscore nagad-app7 2 4"
    exit 1
fi

mkdir -p "$OUTPUT_DIR"

###############################################################################
# Function
###############################################################################

generate_logs() {

    local component="$1"
    local hostname="$2"
    local instances="$3"
    local days="$4"

    for ((day=0; day<=days; day++))
    do
        DATE=$(/bin/date -d "$day day ago" +%Y-%m-%d)

        for ((hour=0; hour<24; hour++))
        do
            for ((inst=1; inst<=instances; inst++))
            do
                FILE="${component}-${hostname}-INST_${inst}-${DATE}-${hour}-0.log.gz"

                touch "${OUTPUT_DIR}/${FILE}"
            done
        done
    done
}

###############################################################################
# Run
###############################################################################

generate_logs "$COMPONENT" "$HOSTNAME" "$INSTANCES" "$DAYS"

TOTAL=$(( (DAYS + 1) * 24 * INSTANCES ))

echo
echo "Done."
echo "Directory : ${OUTPUT_DIR}"
echo "Created   : ${TOTAL} fake log files"