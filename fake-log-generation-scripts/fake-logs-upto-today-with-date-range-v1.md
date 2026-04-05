## Fake Log File Generator with date range

**The Purpose of this Scripts:**
- If you pass `4`
- It should generate logs from **4 days ago up to today**
- Including today

**So if today = `2026-04-05`**

- It should generate:

  ```text
  2026-04-01
  2026-04-02
  2026-04-03
  2026-04-04
  2026-04-05
  ```

Total = 5 days (4 days ago + today)

---

## Script (From X Days Ago → Today)

```bash
#!/bin/bash
set -euo pipefail

# Usage:
#   ./script.sh [date_range] [component] [hostname] [instance] [output_dir]
#
# Examples:
#   ./script.sh 4 extch nagad-app1 INST_1 ./fake_logs
#   ./script.sh 4 weblog web-server PROD_2 /var/logs

date_range="${1:-0}"
component="${2:-extch}"
hostname_var="${3:-nagad-app1}"
instance="${4:-INST_1}"
output_dir="${5:-./fake_logs}"


# Validate numeric input
if ! [[ "$date_range" =~ ^[0-9]+$ ]]; then
    echo "Error: date_range must be a non-negative number"
    exit 1
fi

mkdir -p "$output_dir"

# Loop from X days ago down to today
for ((d=date_range; d>=0; d--))
do
    current_date=$(date -d "-${d} day" +%Y-%m-%d)

    for hour in {00..23}
    do
        file_name="${component}-${hostname_var}-${instance}-${current_date}-${hour}-0.log.gz"
        file_path="${output_dir}/${file_name}"

        touch "$file_path"

        # Set modification time properly
        touch -t "$(date -d "${current_date} ${hour}:00:00" +%Y%m%d%H%M.%S)" "$file_path"
    done
done

echo "Fake log files created successfully."
```

---

## How It Works

**If you run:**

```bash
./script.sh extch nagad-app1 INST_1 4 ./fake_logs
```

**Then loop runs:**

```text
d=4
d=3
d=2
d=1
d=0
```

**And this line:**

```bash
date -d "-${d} day"
```

**Generates:**

```text
4 days ago
3 days ago
2 days ago
1 day ago
today
```

---

## Total Files Created

**If you pass `4`:**

- Total days = 5
- Each day = 24 files
- Total = 5 × 24 = 120 files

---

