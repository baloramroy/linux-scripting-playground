## Restart DFS component one by one:
With the help of this script we can restart all the component one by one with 1 minitue interval. Worked for both Single instance

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

# Single-instance components (stop/start scripts: fkill / start)
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

# Multi-instance components (stop/start scripts: fkillall / startall)
MULTI_INSTANCE_COMPONENTS=(
    davs
    dfs
    portal_davs
    portal_dfs
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

#process_running() {
#    local user="$1"
#
#    pgrep -u "$user" >/dev/null 2>&1
#}


# Processes to ignore when checking if a component is "running" —
# these are login/session/monitoring processes, not the actual app.

#IGNORE_PATTERN='(^|[[:space:]]|/)(-?bash|sh|sshd-session|systemd|\(sd-pam\)|tail|monitorall|monitor|pstat|su)([: ]|$)'

#process_running() {
#    local user="$1"
#
#    pgrep -u "$user" -a 2>/dev/null | grep -Ev "$IGNORE_PATTERN" | grep -q .
#}


IGNORE_PATTERN='(^|/)(-?bash|sh|sshd-session|systemd|\(sd-pam\)|tail|monitorall|monitor|pstat|su)([: ]|$)'

process_running() {
    local user="$1"
    pgrep -u "$user" -a 2>/dev/null \
      | awk '{$1=""; print substr($0,2)}' \
      | grep -Ev "$IGNORE_PATTERN" \
      | grep -q .
}

###############################################################################

is_multi_instance() {
    local user="$1"
    local m

    for m in "${MULTI_INSTANCE_COMPONENTS[@]}"
    do
        if [[ "$m" == "$user" ]]; then
            return 0
        fi
    done

    return 1
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

    local stop_cmd="fkill"
    local start_cmd="start"

    if is_multi_instance "$user"
    then
        stop_cmd="fkillall"
        start_cmd="startall"
    fi

    log "------------------------------------------------------------"
    log "Processing component : ${user} (stop=${stop_cmd}, start=${start_cmd})"

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

    if [[ ! -x "${HOME_DIR}/bin/${stop_cmd}" ]]
    then
        log "[ERROR] ${stop_cmd} not found."

        FAILED+=("$user (${stop_cmd} Missing)")
        return
    fi

    if [[ ! -x "${HOME_DIR}/bin/${start_cmd}" ]]
    then
        log "[ERROR] ${start_cmd} not found."

        FAILED+=("$user (${start_cmd} Missing)")
        return
    fi

    #############################################
    # Stop
    #############################################

    log "Executing ${stop_cmd}..."

    su - "$user" -c "cd ~/bin && ./${stop_cmd}" >>"$LOG_FILE" 2>&1

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

    log "Executing ${start_cmd}..."

    su - "$user" -c "cd ~/bin && ./${start_cmd}" >>"$LOG_FILE" 2>&1

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

for component in "${COMPONENTS[@]}" "${MULTI_INSTANCE_COMPONENTS[@]}"
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