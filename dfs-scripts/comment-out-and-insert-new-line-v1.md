
# Bulk Config file Modification Script

## Overview
This script searches for an **exact line** in `terget-file` files, **comments** it out, and **adds a modified line immediately after**.

### Example Transformation

- **Before:**
	```properties
	java_home=/opt/jdk-1.8.0_161
	```

- **After:**
	```properties
	#java_home=/opt/jdk-1.8.0_161
	JAVA_HOME=/opt/jdk-1.8.0_491
	```
>[!Note]
>Does not work for this `old_line="jdbc.url=jdbc:mysql://10.10.10.1/db"` type of expression.
---

## Script Features

| Feature | Description |
|---------|-------------|
| **Exact Matching** | Uses `grep -Fxq` for precise line matching |
| **Backup Creation** | Creates timestamped `bak.DDMMYYYY` files |
| **Multiple Directories** | Accepts multiple directory paths as arguments |
| **Inline Commenting** | Comments old line without removing it |
| **Line Insertion** | Adds new line immediately after commented line |

---


## Script file

### Create the Script
```bash
vim line-replace.sh
```

### Script Content
```bash
#!/bin/bash

# Configuration - Modify these variables as needed
old_line="java_home=/opt/jdk-1.8.0_161"
new_line="JAVA_HOME=/opt/jdk-1.8.0_491"
backup_suffix="bak.$(date +%d%m%Y)"  # Dynamic backup suffix
target_file_name="application.properties"

#Argument passing check
if [ $# -eq 0 ]; then
    echo "This script needs directory path as argument to run:"
    echo "Usage: $0 [dir1] [dir2] [dir3] ..."
    exit 1
fi

# Process each directory passed as argument
for dir in "$@"
do
    #Check if directory is valid
    if [ ! -d "$dir" ]; then
        echo "[WARN] Directory not found: $dir"
        continue
    fi

    file="$dir/$target_file_name"

    # Check if file exists
    if [ ! -f "$file" ]; then
        echo "file not found: $file"
        continue
    fi

    # Check if the old line exists exactly
    if grep -Fxq "$old_line" "$file"; then
        # Create timestamped backup
        cp -p "$file" "$file.$backup_suffix"
        
        # Comment out the old line
        sed -i "\|$old_line|s|^|#|" "$file"
        
        # Add the new line after the commented line
        sed -i "\|#$old_line|a $new_line" "$file"
        
        echo "[SUCCESS] Updated: $file"
				echo "[INFO] Backup: $file.$backup_suffix"
    else
        echo "No matching entry found in: $file"
    fi
done
```

---

## Execution Guide

### 1. Make Script Executable
```bash
chmod +x line-replace.sh
```

### 2. Run the Script
```bash
./line-replace.sh /path/path/path /path/path/path /path/path/path
```

### 3. Verify Changes
```bash
# Check the updated file
cat /path/path/path/application.properties

# Compare with backup if needed
diff /path/path/path/application.properties /path/path/path/application.properties.bak.08022026
```


---

## Customization Tips

### Change Search/Replace Values

- Edit these lines in the script:

	```bash
	old_line="your_exact_line_to_replace"
	new_line="your_new_line_content"
	```

### Change Target filename

- Modify this line:

	```bash
	file="$dir/your-config-file-name.properties"
	```

### Disable Backups
- Remove or comment out:
  
	```bash
	# cp -p "$file" "$file$backup_suffix"
	```

---

## Safety Notes

- ⚠️ **Always test on a single directory first:**
	```bash
	./line-replace.sh /root/dfs/sys
	```

- ⚠️ **Verify backups exist before trusting changes**

- ⚠️ **Use absolute paths to avoid ambiguity**

---

