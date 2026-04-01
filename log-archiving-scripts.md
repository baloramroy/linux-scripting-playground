```bash
#!/bin/bash
set -euo pipefail

# Configurable Variables
component=${1:-"spg"}
hostname=${2:-"nagad-dc1-app1"}
keep_days=${3:-3}
lock_file="/tmp/${component}_archive.lock"

# Log Rotation Directories
src_dir="/home/lab/fake_logs"
dst_dir="/home/lab/dst_logs"

exec 200 > "$lock_file"
flock -n 200 || { echo "Another instance is running."; exit 1; }


# Validate source directory
if [[ ! -d $src_dir ]]; then
    echo "Error: Source Directory Not Found: $src_dir"
    exit 1
else
    echo "Source Directory Exist. Changing Directory to ${src_dir}"
    cd "$src_dir"
fi

# Change Directory to Source
#cd "$src_dir"

# Ensure destination exists
if [[ ! -d $dst_dir ]]; then
    echo "Destination directory not found, creating: ${dst_dir}"
    mkdir -p "${dst_dir}"
else
    echo "Destination Directory Exist: ${dst_dir}"
fi

# Message
echo "Archiving logs older than $keep_days days..."

# Enable safe glob handling
shopt -s nullglob

# Collect unique dates older than N days
mapfile -t all_dates < <(
    find "${src_dir}" -maxdepth 1 -type f -name "*.log.gz" -print 2>/dev/null \
    | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}' \
    | sort -u
)

# --------------------------------------------------
# Process each date
# --------------------------------------------------

for date in ${all_dates[@]}; do

    archive_name="${component}-${hostname}-${date}.tar.gz"
    echo "Processing Log for this ${date}"

    # 1️⃣ If archive already exists in destination → skip safely
    if [[ -f $dst_dir/$archive_name ]]; then
        echo "Archive already exists in destination. Skipping."
        continue
    fi

    if [[ -f $archive_name ]]; then
        echo "Archive exists in source. Recovering by moving it to destination..."

        # Moving archive to destination
        mv -f "${archive_name}" "${dst_dir}/"
        echo "Recovery move successful."

        # Removing Raw log from Source
        rm -f ${component}-${date}-*.log.gz
        echo "Source Log Remove Successful."
        continue

    fi

done





```
