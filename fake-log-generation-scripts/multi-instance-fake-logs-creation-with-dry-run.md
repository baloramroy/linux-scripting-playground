#!/bin/bash
set -euo pipefail

###############################################################################
# Fake Log Generator
#
# Usage:
#   ./fake-log-generator.sh <component> <hostname> <instances> <days> [dry_run] [output_dir]
#
# Examples:
#   ./fake-log-generator.sh dmscore nagad-app7 2 4
#   ./fake-log-generator.sh dmscore nagad-app7 2 4 true
#   ./fake-log-generator.sh dmscore nagad-app7 2 4 false /tmp/fake_logs
#------------------------------------------------------------------------------

# Variable Declaration and Argument Capture
component="${1:-}"
hostname="${2:-}"
total_instance="${3:-}"
date_range="${4:-}"
dry_run="${5:-false}"
output_dir="${6:-./fake_logs}"

# Normalize dry_run value
dry_run="${dry_run,,}"

#------------------------------------------------------------------------------
# Validation
#------------------------------------------------------------------------------

if [[ $# -lt 4 || $# -gt 6 ]]; then
    printf "Usage:\n"
    printf "  %s <component> <hostname> <instances> <days> [dry_run] [output_dir]\n" "$0"
    printf "\nExamples:\n"
    printf "  %s dmscore nagad-app7 2 4\n" "$0"
    printf "  %s dmscore nagad-app7 2 4 true\n" "$0"
    printf "  %s dmscore nagad-app7 2 4 false /tmp/fake_logs\n" "$0"
    exit 1
fi

# Validate instances
if ! [[ "$total_instance" =~ ^[1-9][0-9]*$ ]]; then
    printf "[ERROR] instances must be a positive integer.\n"
    exit 1
else
    printf "Found Instance Number is = %s\n" "$total_instance"
fi

# Validate days
if ! [[ "$date_range" =~ ^[0-9]+$ ]]; then
    printf "[ERROR] days must be a non-negative integer.\n"
    exit 1
else
    printf "[INFO] Found date range: %s days ago\n" "$date_range"
    printf "[INFO] It will create log from %s to %s.\n" \
        "$(date -d "$date_range days ago" +%Y-%m-%d)" \
        "$(date +%Y-%m-%d)"
    printf "\n"
fi

# Validate dry_run
if [[ "$dry_run" != "true" && "$dry_run" != "false" ]]; then
    printf "[ERROR] dry_run must be 'true' or 'false'.\n"
    exit 1
fi

#------------------------------------------------------------------------------
# Create output directory
#------------------------------------------------------------------------------

if [[ "$dry_run" == "false" ]]; then
    mkdir -p "$output_dir"
else
    printf "[DRY RUN] Output directory would be: %s\n" "$output_dir"
fi

#------------------------------------------------------------------------------
# Function
#------------------------------------------------------------------------------

generate_logs() {

    local component="$1"
    local hostname="$2"
    local instances="$3"
    local date_range="$4"

    for ((day=0; day<=date_range; day++))
    do
        DATE=$(/bin/date -d "$day day ago" +%Y-%m-%d)

        for ((hour=0; hour<24; hour++))
        do
            for ((inst=1; inst<=instances; inst++))
            do
                file="${component}-${hostname}-INST_${inst}-${DATE}-${hour}-0.log.gz"

                if [[ "$dry_run" == "true" ]]; then
                    printf "[DRY RUN] Would create: %s\n" "$file"
                else
                    touch "${output_dir}/${file}"
                fi
            done
        done
    done
}

#------------------------------------------------------------------------------
# Run
#------------------------------------------------------------------------------

generate_logs "$component" "$hostname" "$total_instance" "$date_range"

total=$(( (date_range + 1) * 24 * total_instance ))

#------------------------------------------------------------------------------
# Summary
#------------------------------------------------------------------------------

printf "\n[INFO] Execution completed.\n"

if [[ "$dry_run" == "true" ]]; then
    printf "[INFO] Mode              : DRY RUN\n"
    printf "[INFO] Files to create   : %d\n" "$total"
    printf "[INFO] Output directory  : %s\n" "$output_dir"
    printf "[INFO] No files were created.\n"
else
    printf "[INFO] Mode              : LIVE\n"
    printf "[INFO] Files created     : %d\n" "$total"
    printf "[INFO] Output directory  : %s\n" "$output_dir"
fi

#------------------------------------------------------------------------------
# END
# Author: Baloram Roy
# Department: DevOps
#------------------------------------------------------------------------------