```bash
#!/bin/bash

# Clean up any previous temporary files
rm -f ps.tmp pstat.tmp awk_$$.tmp
touch ps.tmp


# Define process display names and their matching search patterns
cat <<'EOF' > ps.tmp
#KPP-apigw|kpp-apigw)
#KPP-apigwext|(kpp-apigwext|api-gatewayext)
KPP-cms|KPP-cms
KPP-cp|KPP-cp
KPP-cs|KPP-cs
KPP-davs|KPP-davs
KPP-dfs|KPP-dfs
KPP-drs|KPP-drs
KPP-dmscore|KPP-dmscore
KPP-extch|KPP-extch-INST
KPP-extchSMS|KPP-extchSMS
KPP-ias|KPP-ias
KPP-kms|kms
KPP-knotify|KPP-knotify
KPP-kod|KPP-kod
KPP-map|KPP-map
KPP-pcs|KPP-pcs
KPP-spg|KPP-spg
KPP-tms|KPP-tms
KPP-tsp|KPP-tsp
KPP-bds|KPP-bds
KPP-ecs|KPP-ecs
KPP-rms|KPP-rms
KPP-rpg|KPP-rpg
KPP-mps|KPP-mps
KPP-bkofc|KPP-bkofc
KPP-utilityservice|KPP-utilityservice
KPP-npsb_recon|KPP-npsb_recon
EOF

# Print Header
printf "\033[32m================================================================================\033[0m\n" >> pstat.tmp
printf "\033[32mProcess Status Monitoring Script. [ KONA SOFTWARE LAB LTD.]            \033[0m\n" >> pstat.tmp
printf "\033[32m================================================================================\033[0m\n" >> pstat.tmp
printf "\033[32m%-16s%8s %8s %8s %2s %8s %8s %10s \033[0m\n" "PROCESS" "UID" "PID" "PPID" "C" "STIME" "TTY" "TIME" >> pstat.tmp
printf "\033[32m================================================================================\033[0m\n" >> pstat.tmp

found=0
total=0


# Read each service name and process it to present on the screen

while IFS='|' read display_name search_name
do
    [[ "$display_name" =~ ^# ]] && continue

    total=$((total + 1))
    rm -f awk_$$.tmp

    ps -ef | grep -iw "$search_name" | grep -v grep | \
    awk '{ printf "%8s %8d %8d %2d %8s %8s %10s\n", $1, $2, $3, $4, $5, $6, $7 }' > awk_$$.tmp

    if [ ! -s awk_$$.tmp ]
    then
        printf "\033[31m%-16s\033[0m" "$display_name" >> pstat.tmp
        printf "\033[31m%8s %5s %5s %2s %8s %8s %10s\033[0m\n" "-" "-" "-" "-" "-" "-" "-" > awk_$$.tmp
        cat awk_$$.tmp >> pstat.tmp
        found=$((found + 1))
    else
        printf "\033[31m%-16s\033[0m" "$display_name" >> pstat.tmp
        sed '2,$s/^/                /' awk_$$.tmp >> pstat.tmp
    fi

done < ps.tmp

# Print Footer with summary
alive=$((total - found))

if [ -e pstat.tmp ]
then
        printf "\033[32m================================================================================\033[0m\n" >> pstat.tmp
        printf "\033[33mProcess Total = %d , Alive = %d , Down = %d \033[0m\n" "$total" "$alive" "$found" >> pstat.tmp
        printf "\033[32m================================================================================\033[0m\n" >> pstat.tmp
        cat pstat.tmp
else
        echo No Manager process !!!
fi

# Final cleanup
rm -f ps.tmp awk_$$.tmp pstat.tmp

```