To reduce or increase the gap between "PROCESS" and "UID" in the header, you need to adjust the column widths in the `printf` format specifiers.

## The Current Format:

```bash
printf "\033[1;34m%-16s%8s %8s %8s %2s %8s %8s %10s \033[0m\n" "PROCESS" "UID" "PID" "PPID" "C" "STIME" "TTY" "TIME" >> pstat.tmp
```

The `%-16s` for "PROCESS" creates a 16-character wide column, which is causing the gap.

## To Reduce the Gap, Change:

### Option 1: Reduce the Width of PROCESS Column
Change `%-16s` to a smaller width:

```bash
# Change from 16 to 12 characters
printf "\033[1;34m%-12s%8s %8s %8s %2s %8s %8s %10s \033[0m\n" "PROCESS" "UID" "PID" "PPID" "C" "STIME" "TTY" "TIME" >> pstat.tmp
```

### Option 2: Match the Width Used in Data Display
Look at how you're displaying the data:

```bash
printf "\033[31m%-16s\033[0m" "$display_name" >> pstat.tmp
```

You need to make both the header and data use the SAME width:

```bash
# Change both to 12 characters
printf "\033[1;34m%-12s%8s %8s %8s %2s %8s %8s %10s \033[0m\n" "PROCESS" "UID" "PID" "PPID" "C" "STIME" "TTY" "TIME" >> pstat.tmp

# And change the data display to match
printf "\033[31m%-12s\033[0m" "$display_name" >> pstat.tmp
```



## Complete Updated Section:

```bash
# Print Header
printf "\033[34m================================================================================\033[0m\n" >> pstat.tmp
printf "\033[34mProcess Status Monitoring Script. [ KONA SOFTWARE LAB LTD.]            \033[0m\n" >> pstat.tmp
printf "\033[34m================================================================================\033[0m\n" >> pstat.tmp

# ✅ CHANGED: Reduced PROCESS column width from 16 to 12
printf "\033[1;34m%-12s%8s %8s %8s %2s %8s %8s %10s \033[0m\n" "PROCESS" "UID" "PID" "PPID" "C" "STIME" "TTY" "TIME" >> pstat.tmp
printf "\033[34m================================================================================\033[0m\n" >> pstat.tmp

found=0
total=0

# Read each service name and process it to present on the screen
while IFS='|' read display_name search_name
do
    #Trim leading and trailing white space
    display_name=$(echo "$display_name" | xargs)
    search_name=$(echo "$search_name" | xargs)

    #Ignore Hashed (#kpp-spg) component name
    [[ "$display_name" =~ ^# ]] && continue

    total=$((total + 1))
    rm -f awk_$$.tmp

    ps -ef | grep -w "$search_name" | \
    awk '{ if ($8!="grep" && $8!="tail") printf "%8s %8d %8d %2d %8s %8s %10s\n", $1, $2, $3, $4, $5, $6, $7 }' > awk_$$.tmp

    if [ ! -s awk_$$.tmp ]
    then
        # ✅ CHANGED: Match the header width
        printf "\033[31m%-12s\033[0m" "$display_name" >> pstat.tmp
        printf "\033[31m%8s %5s %5s %2s %8s %8s %10s\033[0m\n" "-" "-" "-" "-" "-" "-" "-" > awk_$$.tmp
        cat awk_$$.tmp >> pstat.tmp
        found=$((found + 1))
    else
        # ✅ CHANGED: Match the header width
        printf "\033[31m%-12s\033[0m" "$display_name" >> pstat.tmp
        sed '2,$s/^/                /' awk_$$.tmp >> pstat.tmp
    fi

done < ps.tmp

# Print Footer with summary
alive=$((total - found))

if [ -e pstat.tmp ]
then
        printf "\033[34m================================================================================\033[0m\n" >> pstat.tmp
        printf "\033[1;34mProcess Total = %d , Alive = %d , Down = %d \033[0m\n" "$total" "$alive" "$found" >> pstat.tmp
        printf "\033[34m================================================================================\033[0m\n" >> pstat.tmp
        cat pstat.tmp
fi
```

## Visual Comparison:

### Before (Gap is 16 characters):
```
PROCESS           UID     PID    PPID  C    STIME     TTY       TIME
---
KPP-cms           cms   2052665       1  0    Jul04       ?   00:57:23
```

### After (Gap is 12 characters):
```
PROCESS         UID     PID    PPID  C    STIME     TTY       TIME
---
KPP-cms         cms   2052665       1  0    Jul04       ?   00:57:23
```

## Summary of Changes:

| Line | Before | After | Effect |
|---|---|---|---|
| Header | `%-16s` | `%-12s` | PROCESS column width reduced by 4 characters |
| Data display | `%-16s` | `%-12s` | Data column width reduced to match |
| Alignment lines | `sed '2,$s/^/                /'` | **Need to update this too!** | See note below |

## ⚠️ Important: Also Update the Alignment Line!

In the `else` block, you have:
```bash
sed '2,$s/^/                /' awk_$$.tmp >> pstat.tmp
```

The spaces here (`                `) should match the PROCESS column width. If you change to 12 characters, change to:
```bash
sed '2,$s/^/            /' awk_$$.tmp >> pstat.tmp
```

**Note:** 12 spaces = 12 characters.