```bash

#!/bin/bash
set -euo pipefail

PASS="password123"

# Define users as an array
users=(
    user1
    user2
    user3
)


for user in "${users[@]}"; do 
    if id "$user" &>/dev/null; then
        
        # Change password
        echo "${user}:${PASS}" | chpasswd
        
        # Set password to never expire
        chage -M 99999 -I -1 "$user"

        echo "✓ Password changed and set to never expire for ${user}"
    else
        echo "✗ User ${user} does not exist"
    fi
done


```
