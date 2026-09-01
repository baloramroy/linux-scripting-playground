#!/bin/bash

#----------------------------------------------------------#

# Author: Baloram Roy
# Date Created: 09/06/2026
# Last Modified: 09/06/2026

# Description: 
# Run `sed` operation, find target lines and replace it.

# Usage:
# ./replace-line.sh


#--------------------- CONFIGURATION ----------------------#

target_file="application.properties"

targets=(
    "/home/sysgw/cfg"
    "/home/dmsgw/cfg"
    "/home/callcentergw/cfg"
    "/home/awsgw/cfg"
    "/home/rpgweb/cfg"
)

declare -A config_map

config_map["java_home=/opt/jdk-1.8.0_161"]="JAVA_HOME=/opt/jdk-1.8.0_491"
config_map["java_home=/opt/jdk-1.8.0_291"]="JAVA_HOME=/opt/jdk-1.8.0_491"
config_map["java_home=/opt/jdk-1.8.0_351"]="JAVA_HOME=/opt/jdk-1.8.0_491"


#--------------------- CONFIGURATION ----------------------#

backup_date=$(date +%d%m%Y)

echo ""
echo "Starting update..."
echo ""

for location in "${targets[@]}"; do

    echo "----------------------------------------------------------"
    echo "Processing : $location"
    echo "----------------------------------------------------------"

    # Declare variable inside loop
    file="$location/$target_file"
    backup="$file.bak.$backup_date"

    # Check Directory
    if [ ! -d "$location" ]; then
        echo "[WARN] Directory not found: $location"
        echo ""
        continue
    fi

    # Check file
    if [ ! -f "$file" ]; then
        echo "[WARN] File not found: $file"
        echo ""
        continue
    fi

    # Line replace indicator and file backup indicator
    update=0
    backup_created=0

    for old_line in "${!config_map[@]}"; do
    
        new_line="${config_map[$old_line]}"

        if grep -Fxq "$old_line" "$file"; then

            # Backup once per file
            if [ "$backup_created" -eq 0 ]; then
                cp -p "$file" "$backup"
                echo "[INFO] Backup created: $backup"
                backup_created=1
            fi

            # Comment out the old line
            sed -i "\|$old_line|s|^|#|" "$file"
            
            # Add the new line after the commented line
            sed -i "\|#$old_line|a $new_line" "$file"

            echo "[SUCCESS] Replaced:"
            echo "  Old Line: $old_line"
            echo "  New Line: $new_line"

            update=1
        fi
    done

    if [ "$update" -eq 1 ]; then
        echo "[SUCCESS] File updated: $file"
    else
        echo "[INFO] No matching entry found in: $file"
    fi

    echo ""

done

echo "----------------------------------------------------------"
echo "Update completed."
echo "----------------------------------------------------------"
