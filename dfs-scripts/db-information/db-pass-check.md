#!/bin/bash

# List of components
COMPONENTS=(
	rms
	rpg
	mps
	bkofc
	utilityservice
)

printf "%-15s | %s\n" "Component" "Password"
printf "%-15s-+-%s\n" "---------------" "------------------------------"

for component in "${COMPONENTS[@]}"; do
    dir="/home/${component}/cfg"
    file="${dir}/application.properties"

    if [[ -f "$file" ]]; then
        password=$(grep -m1 '^spring\.datasource\.password=' "$file" | cut -d'=' -f2-)

        if [[ -n "$password" ]]; then
            printf "%-15s | %s\n" "$component" "$password"
        else
            printf "%-15s | %s\n" "$component" "Not Found"
        fi
    else
        printf "%-15s | %s\n" "$component" "application.properties Missing"
    fi
done
