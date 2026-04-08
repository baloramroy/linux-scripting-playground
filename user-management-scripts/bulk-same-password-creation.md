```bash

#!/bin/bash
PASS="password123"

for comp in user1 user2; do 
    if id "$comp" &>/dev/null; then
        echo "${comp}:${PASS}" | chpasswd && \
        echo "✓ password changed successfully for ${comp}"
    else
        echo "✗ user ${comp} does not exist"
    fi
done

```
