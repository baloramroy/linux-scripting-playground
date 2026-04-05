# **Fake Log Generator Script with `Dry Run`**

## **Purpose**

This script generates **fake hourly log files** for a specified number of past days, **including today**.
It is useful for testing, simulations, or preparing a mock log environment.

* **Date range handling:**

  * `date_range` specifies how many days **in the past** to generate logs **in addition to today**.
  * Example: If `date_range=4` and today is `2026-04-05`, the script generates logs for:

    ```
    2026-04-01  (4 days ago)
    2026-04-02  (3 days ago)
    2026-04-03  (2 days ago)
    2026-04-04  (1 day ago)
    2026-04-05  (today)
    ```

* **Total days generated = `date_range + 1`** (includes today).

* **Hourly logs:** 24 log files are created per day, one per hour.

---
## SCRIPT: `generate_fake_logs.sh`

```bash
#!/bin/bash
set -euo pipefail

# Usage:
#   ./script.sh [date_range] [dry_run] [component] [hostname] [instance] [output_dir]
#
# Examples:
#   ./script.sh 4 false extch nagad-app1 INST_1 ./fake_logs
#   ./script.sh 7 true weblog web-server PROD_2 /var/logs

date_range="${1:-0}"
dry_run="${2:-false}"
component="${3:-extch}"
hostname_var="${4:-nagad-app1}"
instance="${5:-INST_1}"
output_dir="${6:-./fake_logs}"

# Normalize dry_run value to lowercase
dry_run="${dry_run,,}"

# Validate numeric input
if ! [[ "$date_range" =~ ^[0-9]+$ ]]; then
    printf "[ERROR] date_range must be a non-negative number.\n"
    exit 1
fi

# Validate dry_run value
if [[ "$dry_run" != "true" && "$dry_run" != "false" ]]; then
    printf "[ERROR] dry_run must be 'true' or 'false'.\n"
    exit 1
fi

# Create output directory (only if not dry run)
if [[ "$dry_run" == "false" ]]; then
    mkdir -p "$output_dir"
else
    printf "[DRY RUN] Output dir would be: %s\n" "$output_dir"
fi

total_files=0

# Loop from X days ago down to today
for ((d=date_range; d>=0; d--))
do
    current_date=$(date -d "-${d} day" +%Y-%m-%d)

    for hour in $(seq -w 0 23)
    do
        file_name="${component}-${hostname_var}-${instance}-${current_date}-${hour}-0.log.gz"
        file_path="${output_dir}/${file_name}"

        if [[ "$dry_run" == "true" ]]; then
            printf "[DRY RUN] Would create: %s\n" "$file_name"
        else
            touch "$file_path"
            # Set modification time properly
            touch -t "$(date -d "${current_date} ${hour}:00:00" +%Y%m%d%H%M.%S)" "$file_path"
        fi
        ((total_files++))
    done
done

# Final message block
printf "\n[INFO] Execution completed.\n"

if [[ "$dry_run" == "true" ]]; then
    printf "[INFO] Mode: DRY RUN\n"
    printf "[INFO] Dry run completed successfully. No files were created.\n"
    printf "[INFO] Files that would have been created: %d\n" "$total_files"
else
    printf "[INFO] Mode: LIVE\n"
    printf "[INFO] Total files created: %d\n" "$total_files"
    printf "[INFO] Output directory: %s\n" "$output_dir"
fi

```

---

## **How to Use**

### **Make the script executable**

```bash
chmod +x generate_fake_logs.sh
```

### **Run with defaults**

```bash
./generate_fake_logs.sh
```

* Defaults:

  * `date_range = 0` → generates **only today’s logs**
  * `dry_run = false`
  * `component = extch`
  * `hostname = nagad-app1`
  * `instance = INST_1`
  * `output_dir = ./fake_logs`

### **Run with custom values**

```bash
./generate_fake_logs.sh 4 false extch nagad-app6 INST_2 /tmp/logs
```


### **Arguments Order**

| Position | Variable   | Description                                                 | Example    |
| -------- | ---------- | ----------------------------------------------------------- | ---------- |
| $1       | date_range | Number of past days to generate logs (in addition to today) | 4          |
| $2       | dry_run    | Dry run mode (`true` or `false`)                            | false      |
| $3       | component  | Name of the log component                                   | extch      |
| $4       | hostname   | Hostname for the log                                        | nagad-app6 |
| $5       | instance   | Instance identifier                                         | INST_2     |
| $6       | output_dir | Directory to store generated logs                           | /tmp/logs  |

---

## **How It Works**

1. **Date Loop**

```bash
for ((d=date_range; d>=0; d--))
do
    current_date=$(date -d "-${d} day" +%Y-%m-%d)
```

* Loops from **`date_range` days ago down to today**.
* Example with `date_range=4` (today = 2026-04-05):


  | d | Generated Date |
  | - | -------------- |
  | 4 | 2026-04-01     |
  | 3 | 2026-04-02     |
  | 2 | 2026-04-03     |
  | 1 | 2026-04-04     |
  | 0 | 2026-04-05     |


#

2. **Hourly Log Loop**

```bash
for hour in $(seq -w 0 23)
```

* Generates **24 files per day**, one per hour.
* File format:

  ```text
  component-hostname-instance-YYYY-MM-DD-HH-0.log.gz
  ```

* Example:

  ```text
  extch-nagad-app6-INST_1-2026-04-01-00-0.log.gz
  extch-nagad-app6-INST_1-2026-04-01-01-0.log.gz
  ...
  extch-nagad-app6-INST_1-2026-04-01-23-0.log.gz
  ```

* File modification timestamps match the **actual date and hour**:

  ```bash
  touch -t "$(date -d "${current_date} ${hour}:00:00" +%Y%m%d%H%M.%S)" "$file_path"
  ```

#

3. **Dry Run Mode**

* If `dry_run=true`, files are **not created**.
* Script prints **what would have been created**:

  ```text
  [DRY RUN] Would create: extch-nagad-app6-INST_1-2026-04-01-00-0.log.gz
  ...
  [INFO] Mode: DRY RUN
  [INFO] Dry run completed successfully. No files were created
  [INFO] Files that would have been created: 120
  ```

* Total files = `(date_range + 1) × 24` (hours per day)

#

4. **Live Mode**

* If `dry_run=false`, files are **actually created** in `output_dir`.
* After execution, summary:

  ```text
  [INFO] Mode: LIVE
  [INFO] Total files created: 120
  [INFO] Output directory: /tmp/logs
  ```

---

## **Output Example**

* For `date_range=4` (4 days ago to today) and hourly logs:

  ```text
  extch-nagad-app6-INST_1-2026-04-01-00-0.log.gz
  extch-nagad-app6-INST_1-2026-04-01-01-0.log.gz
  ...
  extch-nagad-app6-INST_1-2026-04-05-23-0.log.gz
  ```

* 5 days × 24 hours = **120 log files**

---

## **Final Notes**

* **Total days generated = `date_range + 1`** (includes today).
* Each day produces **24 hourly logs**.
* Supports **dry-run mode** for testing without creating files.
* Easy to customize: `component`, `hostname`, `instance`, or `output_dir`.

---

