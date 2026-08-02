## Restart DFS component one by one:
With the help of this script we can restart all the component one by one with 1 minitue interval.

```bash
#!/bin/bash
###############################################################################
# KPP Component Restart Script
#
# Author  : Baloram
# Purpose : Restart all KPP components sequentially
# Run As  : root
###############################################################################

set -uo pipefail

#############################
# Configuration
#############################

COMPONENTS=(
    cms
    cp
    cs
    davs
    dfs
    drs
    dmscore
    extch
    extchSMS
    ias
    kms
    knotify
    kod
    map
    pcs
    spg
    tms
    tsp
    bds
    ecs
    rms
    rpg
    mps
    bkofc
    utilityservice
    npsb_recon
)

STOP_TIMEOUT=60
START_TIMEOUT=60
CHECK_INTERVAL=2

LOG_DIR="/var/log"
LOG_FILE="${LOG_DIR}/component_restart_$(date +%Y%m%d_%H%M%S).log"

mkdir -p "$LOG_DIR"

SUCCESS=()
FAILED=()

###############################################################################

log() {
    local msg="$1"
    echo "$(date '+%F %T') | ${msg}" | tee -a "$LOG_FILE"
}

###############################################################################

process_running() {
    local user="$1"

    pgrep -u "$user" >/dev/null 2>&1
}

###############################################################################

wait_for_stop() {

    local user="$1"

    local elapsed=0

    while process_running "$user"
    do
        if (( elapsed >= STOP_TIMEOUT )); then
            return 1
        fi

        sleep "$CHECK_INTERVAL"
        ((elapsed+=CHECK_INTERVAL))
    done

    return 0
}

###############################################################################

wait_for_start() {

    local user="$1"

    local elapsed=0

    while ! process_running "$user"
    do
        if (( elapsed >= START_TIMEOUT )); then
            return 1
        fi

        sleep "$CHECK_INTERVAL"
        ((elapsed+=CHECK_INTERVAL))
    done

    return 0
}

###############################################################################

restart_component() {

    local user="$1"

    log "------------------------------------------------------------"
    log "Processing component : ${user}"

    #############################################
    # Verify user
    #############################################

    if ! id "$user" >/dev/null 2>&1
    then
        log "[ERROR] User does not exist."

        FAILED+=("$user (User Missing)")
        return
    fi

    HOME_DIR=$(getent passwd "$user" | cut -d: -f6)

    if [[ ! -d "${HOME_DIR}/bin" ]]
    then
        log "[ERROR] ${HOME_DIR}/bin not found."

        FAILED+=("$user (bin Missing)")
        return
    fi

    if [[ ! -x "${HOME_DIR}/bin/fkill" ]]
    then
        log "[ERROR] fkill not found."

        FAILED+=("$user (fkill Missing)")
        return
    fi

    if [[ ! -x "${HOME_DIR}/bin/start" ]]
    then
        log "[ERROR] start not found."

        FAILED+=("$user (start Missing)")
        return
    fi

    #############################################
    # Stop
    #############################################

    log "Executing fkill..."

    su - "$user" -c "cd ~/bin && ./fkill" >>"$LOG_FILE" 2>&1

    if wait_for_stop "$user"
    then
        log "Component stopped successfully."
    else
        log "[ERROR] Timeout waiting for component to stop."

        FAILED+=("$user (Stop Timeout)")
        return
    fi

    #############################################
    # Start
    #############################################

    log "Executing start..."

    su - "$user" -c "cd ~/bin && ./start" >>"$LOG_FILE" 2>&1

    if wait_for_start "$user"
    then
        log "Component started successfully."

        SUCCESS+=("$user")
    else
        log "[ERROR] Timeout waiting for component to start."

        FAILED+=("$user (Start Timeout)")
    fi
}

###############################################################################

echo
echo "==============================================================="
echo " KPP Component Restart"
echo "==============================================================="

log "Restart operation started."

for component in "${COMPONENTS[@]}"
do
    restart_component "$component"
done

log "Restart operation completed."

echo
echo "==================== SUMMARY ===================="

echo
echo "Successful : ${#SUCCESS[@]}"

for item in "${SUCCESS[@]}"
do
    echo "  ✔ $item"
done

echo
echo "Failed : ${#FAILED[@]}"

for item in "${FAILED[@]}"
do
    echo "  ✖ $item"
done

echo
echo "Log File : $LOG_FILE"

echo "================================================="

if (( ${#FAILED[@]} > 0 ))
then
    exit 1
else
    exit 0
fi
```

## Complete Workflow
Here's the overall flow in one diagram:

```
                Start Script
                     │
                     ▼
          Read COMPONENTS array
                     │
                     ▼
        ┌─────────────────────┐
        │ Next component (cms)│
        └──────────┬──────────┘
                   │
                   ▼
            Does user exist?
                   │
          Yes ─────┴───── No
           │               │
           ▼               ▼
   Check ~/bin exists   Mark FAILED
           │
           ▼
 Check fkill & start exist
           │
           ▼
      Run ./fkill
           │
           ▼
 Wait until process stops
           │
           ▼
      Run ./start
           │
           ▼
 Wait until process starts
           │
           ▼
 Add to SUCCESS
           │
           ▼
      Next component
           │
           ▼
      Print summary
           │
           ▼
            End
```
