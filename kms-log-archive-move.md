
# KMS Log Archiving Script – Brief Documentation

## 1. Purpose of the Script

This script is designed to:

1. Archive **KMS application** log files older than a specified number of days.
2. **Compress** hourly `.log.gz` files into a single daily `.tar.gz` archive.
3. Move the archive to a **centralized log** storage directory.
4. **Delete original hourly logs** after successful archival.
5. Prevent **duplicate execution** using a lock file.
6. Handle **recovery scenarios** safely.

---

## 2. Log File Format Assumption

**The script assumes log files follow this naming format:**

```
kms-YYYY-MM-DD-HH-0.log.gz
```

**Example:**

```
kms-2026-03-28-14-0.log.gz
kms-2026-03-28-15-0.log.gz
```

**Where:**

* `kms` → Component name
* `YYYY-MM-DD` → Date
* `HH` → Hour
* `0` → Instance number

---

## 3. Configurable Variables

**At the top of the script:**

```
SRC_DIR="/home/kms/bin/log"
DEST_DIR="/LOGS/app6/kms/kmslog"
COMPONENT="kms"
DAYS=3
LOCK_FILE="/tmp/kms_archive.lock"
```

**Meaning:**

1. `SRC_DIR`
   Location where hourly logs are generated.

2. `DEST_DIR`
   Destination where daily compressed archives will be stored.

3. `COMPONENT`
   Prefix of log files (must match filename format).

4. `DAYS`
   Logs older than this number of days will be archived.

5. `LOCK_FILE`
   Prevents multiple instances of this script from running at the same time.

---

## 4. Script Workflow Overview

The script performs the following steps:

### Step 1: Safety & Locking

* `set -euo pipefail` ensures:

  * Exit on any error.
  * No use of undefined variables.
  * Pipeline failure detection.
* Uses `flock` to prevent concurrent execution.

If another instance is running:

```
Another instance is running.
```

---

### Step 2: Source Directory Validation

* Checks if source directory exists.
* If not found, exits with error.

---

### Step 3: Find Eligible Log Dates

**The script:**

1. Finds files:

   ```
   kms-*.log.gz
   ```
2. Filters logs older than `$DAYS`
3. Extracts unique dates (YYYY-MM-DD)
4. Stores them in an array `DATES`

**If no eligible logs:**

```
No logs older than X days found.
```

---

### Step 4: Process Each Date

**For each eligible date:**

```
kms-YYYY-MM-DD.tar.gz
```

The script handles 3 scenarios:

---

### Case 1: Archive Already Exists in Destination

**If archive already exists in destination:**

* Script skips that date.
* Prevents duplicate archiving.

---

### Case 2: Recovery Scenario

If archive exists in source directory (previous run interrupted):

* Move archive to destination.
* Delete hourly logs.
* Continue to next date.

This ensures recovery from partial failures.

---

### Case 3: Normal Archive Creation

**If:**

* Archive does not exist
* Hourly logs exist

**Then:**

1. Create archive:

   ```bash
   tar -czf kms-YYYY-MM-DD.tar.gz
   ```
2. Move archive to destination.
3. Verify archive moved successfully.
4. Delete original hourly logs.

**If archive move fails:**

* Script exits with error.
* Logs are NOT deleted.

---

## 5. Example Cron Entry

Example: Run daily at 1:30 AM

```
30 1 * * * /path/to/kms_archive.sh >> /var/log/kms_archive.log 2>&1
```

---

## 6. Important Notes for Administrator

Before using:

1. Verify log filename format matches expected pattern.
2. Ensure destination path has enough disk space.
3. Ensure script has execution permission:

   ```bash
   chmod +x kms_archive.sh
   ```
4. Test in a lower environment before production use.
5. Confirm that retention policy aligns with `DAYS` variable.

---

## 7. Expected Result

**For hourly logs like:**

```
kms-2026-03-28-00-0.log.gz
kms-2026-03-28-01-0.log.gz
...
kms-2026-03-28-23-0.log.gz
```

**After execution:**

- Destination:

    ```
    kms-2026-03-28.tar.gz
    ```

- Source:\
  `Hourly logs removed.`

---

## The Scipt Itself

```bash
#!/bin/bash
set -euo pipefail

SRC_DIR="/home/kms/bin/log"
DEST_DIR="/LOGS/app6/kms/kmslog"
COMPONENT="kms"
DAYS=3
LOCK_FILE="/tmp/kms_archive.lock"

exec 200>"$LOCK_FILE"
flock -n 200 || { echo "Another instance is running."; exit 1; }

# Validate source directory
if [ ! -d "$SRC_DIR" ]; then
    echo "ERROR: Source directory not found: $SRC_DIR"
    exit 1
fi

cd "$SRC_DIR"

#Create destination if it is doesnot exist
mkdir -p "$DEST_DIR"

echo "Archiving logs older than $DAYS days..."

# Enable safe glob handling
shopt -s nullglob

# Collect unique dates older than N days
mapfile -t DATES < <(
    find . -maxdepth 1 -type f \
        -name "${COMPONENT}-*.log.gz" \
        -mtime +"$DAYS" \
    | awk -F'[-]' '{print $2"-"$3"-"$4}' \
    | sort -u
)

if [ ${#DATES[@]} -eq 0 ]; then
    echo "No logs older than $DAYS days found."
    exit 0
fi

# --------------------------------------------------
# Process each date
# --------------------------------------------------
for DATE in "${DATES[@]}"; do

    ARCHIVE_NAME="${COMPONENT}-${DATE}.tar.gz"

    echo ""
    echo "Processing date: $DATE"

    # 1️⃣ If archive already exists in destination → skip safely
    if [ -f "$DEST_DIR/$ARCHIVE_NAME" ]; then
        echo "Archive already exists in destination. Skipping."
        continue
    fi

    # 2️⃣ Recovery case: archive exists in source
    if [ -f "$ARCHIVE_NAME" ]; then
        echo "Archive exists in source. Recovering by moving it to destination..."
        mv -f "$ARCHIVE_NAME" "$DEST_DIR/"
        echo "Archive in the source, successfully move to the destination."
        rm -f ${COMPONENT}-${DATE}-*.log.gz
        echo "Log Remove Successfuly from the Source directory."
        continue
    fi

    FILES=(${COMPONENT}-${DATE}-*.log.gz)

    if [ ${#FILES[@]} -eq 0 ]; then
        echo "No hourly log files found for $DATE"
        continue
    else
        echo "Still hourly log exist in the source, after archive move to destination."
    fi

    # 3️⃣ Normal archive creation flow
    echo "Creating archive: $ARCHIVE_NAME"
    tar -czf "$ARCHIVE_NAME" "${FILES[@]}"
    echo "Archive created successfully."

    mv -f "$ARCHIVE_NAME" "$DEST_DIR/"
    if [ -f "$DEST_DIR/$ARCHIVE_NAME" ]; then
        echo "Archive moved successfully"
        echo "Deleting hourly log files.."
        rm -f "${FILES[@]}"
    else
        echo "ERROR: Archive move failed!" 
        exit 1
    fi
done

echo ""
echo "All eligible logs processed successfully."
```
