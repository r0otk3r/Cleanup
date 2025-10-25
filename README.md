# Debian/Ubuntu Full System Cleanup Script

This script provides a comprehensive cleanup solution for Debian and Ubuntu-based systems. It automates various tasks to free up disk space, remove unnecessary files, and optimize your system's performance.

**Please read the entire README and understand the script before running it.**

## ⚠️ WARNING: Use with Caution ⚠️

This script performs **extensive system modifications**, including package removal and file deletion.
* **Always review the script's contents** before execution to understand what it does.
* **It is highly recommended to back up your system** before running any system cleanup script, especially on a production machine or if you are unsure.
* The script includes a confirmation prompt, but once confirmed, it will proceed with cleanup.

## Features

*   **Package Management:**
    *   Updates package lists.
    *   Removes unused packages and dependencies (`autoremove`, `autoclean`, `clean`).
    *   Removes old kernel versions, keeping the current one.
    *   Cleans orphaned packages (requires `deborphan` to be installed).
*   **Log & Cache Management:**
    *   Cleans crash reports.
    *   Vacuums journal logs (older than 7 days).
    *   Rotates and compresses old logs.
    *   Clears `apt` list caches.
    *   Cleans thumbnail caches for all users.
    *   Cleans temporary files (`/tmp`, `/var/tmp`).
    *   Cleans Snap application cache (removes old revisions).
*   **User Data Cleanup:**
    *   Empties the Trash for all users.
*   **System Information:**
    *   Displays disk space usage at the end.

## Prerequisites

*   A Debian or Ubuntu-based operating system.
*   `sudo` privileges or root access.
*   (Optional but recommended for orphaned package cleanup) `deborphan` utility:
    ```bash
    sudo apt install deborphan
    ```

## How to Use

1.  **Save the Script:**
    Save the provided script content into a file, for example, `cleanup.sh`.

    ```bash
    #!/bin/bash
    # Full system cleanup script for Debian/Ubuntu-based systems
    # Run as root or with sudo

    set -euo pipefail  # Exit on error, undefined variables, pipe failures

    echo "=== Full System Cleanup ==="

    # Confirm running as root
    if [[ "$EUID" -ne 0 ]]; then
        echo "[!] Please run this script with sudo or as root."
        exit 1
    fi

    # Safety confirmation
    read -p "This will perform extensive system cleanup. Continue? (y/N): " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        echo "Aborted."
        exit 0
    fi

    echo "[+] Updating package list..."
    apt update -qq

    echo "[+] Removing unused packages and dependencies..."
    apt autoremove --purge -y
    apt autoclean -y
    apt clean -y

    echo "[+] Removing old kernels (except the current one)..."
    current_kernel=$(uname -r)
    # Safer kernel removal with better filtering
    kernels_to_remove=$(dpkg -l 'linux-image-*' 'linux-headers-*' | grep ^ii | awk '{print $2}' | grep -vE "${current_kernel%-generic}|dbg" | grep -v linux-image-unsigned)
    if [[ -n "$kernels_to_remove" ]]; then
        echo "$kernels_to_remove" | xargs apt -y purge --auto-remove
    else
        echo "No old kernels found to remove."
    fi

    echo "[+] Cleaning orphaned packages..."
    if command -v deborphan >/dev/null 2>&1; then
        orphaned=$(deborphan)
        if [[ -n "$orphaned" ]]; then
            echo "$orphaned" | xargs -r apt -y remove --purge
        else
            echo "No orphaned packages found."
        fi
    else
        echo "deborphan not installed, skipping orphaned package cleanup."
        echo "Install with: apt install deborphan"
    fi

    echo "[+] Cleaning up crash reports..."
    if [[ -d /var/crash ]]; then
        find /var/crash -type f -name "*.crash" -delete
    fi

    echo "[+] Cleaning journal logs (older than 7 days)..."
    if command -v journalctl >/dev/null 2>&1; then
        journalctl --vacuum-time=7d
    else
        echo "journalctl not available, skipping journal cleanup."
    fi

    echo "[+] Rotating and compressing old logs..."
    if command -v logrotate >/dev/null 2>&1; then
        logrotate -f /etc/logrotate.conf
    fi

    # Clean log files more safely
    echo "[+] Cleaning log files..."
    find /var/log -type f -name "*.log" -exec sh -c 'echo -n > "$1"' _ {} \; 2>/dev/null || true

    echo "[+] Cleaning thumbnail cache..."
    for home in /home/* /root; do
        if [[ -d "$home/.cache/thumbnails" ]]; then
            rm -rf "$home/.cache/thumbnails"/*
        fi
    done

    echo "[+] Cleaning apt lists..."
    apt update -qq  # Refresh after cleanup
    rm -rf /var/lib/apt/lists/*
    mkdir -p /var/lib/apt/lists/partial

    echo "[+] Cleaning temporary files..."
    # Safer temp file cleanup - don't remove the directories themselves
    find /tmp -mindepth 1 -maxdepth 1 -type f -atime +1 -delete 2>/dev/null || true
    find /var/tmp -mindepth 1 -maxdepth 1 -type f -atime +7 -delete 2>/dev/null || true

    echo "[+] Emptying all users' Trash..."
    for home in /home/* /root; do
        trash_dir="$home/.local/share/Trash"
        if [[ -d "$trash_dir/files" ]]; then
            rm -rf "$trash_dir/files"/*
        fi
        if [[ -d "$trash_dir/info" ]]; then
            rm -rf "$trash_dir/info"/*
        fi
    done

    echo "[+] Cleaning snap cache..."
    if command -v snap >/dev/null 2>&1; then
        # Remove old snap revisions
        snap list --all | awk '/disabled/{print $1, $3}' | while read snapname revision; do
            snap remove "$snapname" --revision="$revision"
        done 2>/dev/null || true
    else
        echo "snap not available, skipping snap cleanup."
    fi

    echo "[+] Checking disk space..."
    df -h / /home /var /tmp

    echo "[+] Cleanup complete. Recommended to reboot if kernels were removed."
    ```

2.  **Make it Executable:**
    ```bash
    chmod +x cleanup.sh
    ```

3.  **Run the Script:**
    ```bash
    sudo ./cleanup.sh
    ```
    The script will prompt you for confirmation before proceeding.

## Explanation of Key Sections

*   `set -euo pipefail`: A robust bash setting.
    *   `-e`: Exits immediately if a command exits with a non-zero status.
    *   `-u`: Treats unset variables as an error and exits.
    *   `-o pipefail`: The return value of a pipeline is the status of the last command to exit with a non-zero status, or zero if all commands exit successfully.
*   **Kernel Removal:** The script intelligently identifies and removes old `linux-image-*` and `linux-headers-*` packages, specifically excluding the currently running kernel and debug/unsigned versions.
*   **Orphaned Packages:** Utilizes `deborphan` to find and remove packages that are no longer required by any other installed package.
*   **Log Cleaning:** Empties (truncates) `.log` files in `/var/log` rather than deleting them, which is safer as some applications expect log files to exist.
*   **Temporary Files:** Cleans files in `/tmp` older than 1 day and in `/var/tmp` older than 7 days. It avoids removing the directories themselves to prevent issues.
*   **Snap Cache:** Iterates through all disabled (old) Snap revisions and removes them to free up space.

## Recommended Action After Cleanup

If the script removed old kernel versions, it is **highly recommended to reboot your system** to ensure you are running on a clean, updated kernel and to finalize any related changes.

```bash
sudo reboot
```
### Contributing

Feel free to suggest improvements or report issues by opening an issue or pull request.

## License

This script is provided under the MIT License.
