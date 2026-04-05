## Fake Log File Generator with Date Range including `Today`

**The Purpose of this Scripts:**
- If you pass `4`
- It should generate logs for:

  ```
  Today
  1 day ago
  2 days ago
  3 days ago
  ```

Total = **4 days only**

---

## SCRIPT: Exactly X Days Including Today

```bash
#!/bin/bash
set -euo pipefail

# Usage:
#   ./script.sh [total_days] [component] [hostname] [instance] [output_dir]
#
# Examples:
#   ./script.sh 4 extch nagad-app1 INST_1 ./fake_logs
#   ./script.sh 4 weblog web-server PROD_2 /var/logs

total_days="${1:-1}"
component="${2:-extch}"
hostname_var="${3:-nagad-app1}"
instance="${4:-INST_1}"
output_dir="${5:-./fake_logs}"

# Validate numeric input
if ! [[ "$total_days" =~ ^[0-9]+$ ]] || [ "$total_days" -eq 0 ]; then
    echo "Error: total_days must be a positive number"
    exit 1
fi

mkdir -p "$output_dir"

# Loop exactly X days including today
for ((d=total_days-1; d>=0; d--))
do
    current_date=$(date -d "-${d} day" +%Y-%m-%d)

    for hour in {00..23}
    do
        file_name="${component}-${hostname_var}-${instance}-${current_date}-${hour}-0.log.gz"
        file_path="${output_dir}/${file_name}"

        touch "$file_path"

        # Set modification time
        touch -t "$(date -d "${current_date} ${hour}:00:00" +%Y%m%d%H%M.%S)" "$file_path"
    done
done

echo "Fake log files created successfully."
```

---

## How It Works

### Suppose Today = 2026-04-05

**You run:**

```bash
./script.sh extch nagad-app1 INST_1 4 ./fake_logs
```

**Then:**

```
total_days = 4
```

#

### If `total_days=4`

**Then loop becomes:**

```text
for ((d=3; d>=0; d--))
```

**So values of `d`:**

```
3
2
1
0
```

#

### What This Line Does

```
date -d "-${d} day"
```

**Generates:**

```
d=3 → 3 days ago
d=2 → 2 days ago
d=1 → 1 day ago
d=0 → today
```

**So final dates:**

```
2026-04-02
2026-04-03
2026-04-04
2026-04-05
```

Exactly 4 days.

---

## File Count Calculation

If you pass `4`:

- 4 days
- 24 hours per day
- 4 × 24 = 96 files

---

## Why We Use `days_total-1`

**Because:**

If you want **exactly X days including today**,
You must start counting from:

```
X - 1
```

**Otherwise it becomes:**

```
X + 1 days
```

This is a very common off-by-one logic issue in scripting.

---


