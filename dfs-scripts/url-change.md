# url change script
```bash
#!/bin/bash

#----------------------------------------------------------#
# Author: Baloram Roy
# Description: Replace config line using safe method
#----------------------------------------------------------#

target_file="application.properties"

targets=(
    "/home/bkofc/cfg"
    "/home/dfs/cfg"
    "/home/ecs/cfg"
    "/home/extch/cfg"
    "/home/ias/cfg"
    "/home/knotify/cfg"
    "/home/kod/cfg"
    "/home/map/cfg"
    "/home/rms/cfg"
    "/home/spg/cfg"
    "/home/tms/cfg"
)

declare -A config_map

config_map["spring.datasource.password="]="spring.datasource.password=bkofc"
config_map["spring.datasource.password="]="spring.datasource.password=dfs"
config_map["spring.datasource.password="]="spring.datasource.password=ecs"
config_map["spring.datasource.password="]="spring.datasource.password=extch"
config_map["spring.datasource.password="]="spring.datasource.password=ias"
config_map["spring.datasource.password="]="spring.datasource.password=knotify"
config_map["spring.datasource.password="]="spring.datasource.password=kod"
config_map["spring.datasource.password="]="spring.datasource.password=map"
config_map["spring.datasource.password="]="spring.datasource.password=rms"
config_map["spring.datasource.password="]="spring.datasource.password=spg"
config_map["spring.datasource.password="]="spring.datasource.password=tms"


backup_date=$(date +%d%m%Y)

echo ""
echo "Starting update..."
echo ""

for location in "${targets[@]}"; do

    echo "----------------------------------------------------------"
    echo "Processing : $location"
    echo "----------------------------------------------------------"

    file="$location/$target_file"
    backup="$file.bak.$backup_date"

    if [ ! -d "$location" ]; then
        echo "[WARN] Directory not found: $location"
        continue
    fi

    if [ ! -f "$file" ]; then
        echo "[WARN] File not found: $file"
        continue
    fi

    update=0
    backup_created=0

    for pattern in "${!config_map[@]}"; do

        new_line="${config_map[$pattern]}"

        # check ANY occurrence (commented or not)
        if grep -q "^[#]*$pattern" "$file"; then

            # backup once
            if [ "$backup_created" -eq 0 ]; then
                cp -p "$file" "$backup"
                echo "[INFO] Backup created: $backup"
                backup_created=1
            fi

            # 1. remove ONLY old commented occurrences
            sed -i "/^#$pattern/d" "$file"

            # 2. find ACTIVE line and process it once
            if grep -q "^$pattern" "$file"; then

                # comment ONLY the active line (first match)
                sed -i "/^$pattern/s/^/#/" "$file"

                # insert new line after it
                sed -i "/^#$pattern/a $new_line" "$file"

            fi

            echo "[SUCCESS] Updated:"
            echo "  Pattern: $pattern"
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
```


## new

```bash
#!/bin/bash

#----------------------------------------------------------#
# Author: Baloram Roy
# Description: Replace config line using safe method
#----------------------------------------------------------#

targets=(
    "bkofc"
    "dfs"
    "ecs"
    "extch"
    "ias"
    "knotify"
    "kod"
    "map"
    "rms"
    "spg"
    "tms"
)

base_path="/home"
dest_path="cfg"
target_file="application.properties"
backup_date=$(date +%d%m%Y)

echo ""
echo "Starting update..."
echo ""

for name in "${targets[@]}"; do

    location="$base_path/$name/$dest_path"
    file="$base_path/$name/$dest_path/$target_file"
    backup="$file.bak.$backup_date"

    echo "----------------------------------------------------------"
    echo "Processing : $file"
    echo "----------------------------------------------------------"

    if [ ! -d "$location" ]; then
        echo "[WARN] Directory not found: $location"
        continue
    fi

    if [ ! -f "$file" ]; then
        echo "[WARN] File not found: $file"
        continue
    fi

    update=0
    backup_created=0
    pattern="spring.datasource.password="
```
