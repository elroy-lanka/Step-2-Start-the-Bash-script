[smart_campus.sh.sh](https://github.com/user-attachments/files/31436850/smart_campus.sh.sh)
#!/bin/bash
# ==========================================================
# Smart Campus IoT Device Management System
# Operating Systems Assessment - Task 1
# ==========================================================

# Log file
SYSTEM_LOG="system_monitor_log.txt"

# Default sensor log directory
SENSOR_LOG_DIR="$HOME/sensor_logs"

# Archive directory
ARCHIVE_DIR="$SENSOR_LOG_DIR/ArchiveLogs"

# Critical system processes that must not be terminated
CRITICAL_PROCESSES=("systemd" "init" "kthreadd" "sshd" "kworker" "rcu_sched")

# Create required directories/files if they do not exist
mkdir -p "$SENSOR_LOG_DIR"
mkdir -p "$ARCHIVE_DIR"
touch "$SYSTEM_LOG"

# ----------------------------------------------------------
# Function: Write action to system log
# ----------------------------------------------------------
log_action() {
    local message="$1"
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $message" >> "$SYSTEM_LOG"
}

# ----------------------------------------------------------
# Function: Display CPU usage
# ----------------------------------------------------------
show_cpu_usage() {
    echo
    echo "=========================================="
    echo "          CURRENT CPU USAGE"
    echo "=========================================="
    top -bn1 | grep "Cpu(s)" | head -n 1
    echo
    echo "=========================================="
    echo "       TOP 5 CPU CONSUMING PROCESSES"
    echo "=========================================="
    ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head -n 6
    log_action "CPU usage was displayed."
}

# ----------------------------------------------------------
# Function: Display memory usage
# ----------------------------------------------------------
show_memory_usage() {
    echo
    echo "=========================================="
    echo "          CURRENT MEMORY USAGE"
    echo "=========================================="
    free -h
    echo
    echo "=========================================="
    echo "       TOP 5 MEMORY CONSUMING PROCESSES"
    echo "=========================================="
    ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%mem | head -n 6
    log_action "Memory usage was displayed."
}

# ----------------------------------------------------------
# Function: Display top processes
# ----------------------------------------------------------
show_top_processes() {
    echo
    echo "=========================================="
    echo "      TOP 10 PROCESSES BY MEMORY USAGE"
    echo "=========================================="
    echo "PID     USER     %CPU   %MEM   COMMAND"
    echo "------------------------------------------"
    ps aux --sort=-%mem | head -n 11 | awk '{printf "%-7s %-8s %-6s %-6s %s\n", $2, $1, $3, $4, $11}'
    log_action "Top processes were displayed."
}

# ----------------------------------------------------------
# Function: Terminate a process
# ----------------------------------------------------------
terminate_process() {
    echo
    echo "=========================================="
    echo "         PROCESS TERMINATION"
    echo "=========================================="
    
    # Show current processes
    ps -eo pid,comm,%cpu,%mem --sort=-%mem | head -n 15
    echo
    echo "------------------------------------------"
    echo "Critical processes protected: ${CRITICAL_PROCESSES[*]}"
    echo "------------------------------------------"
    
    read -p "Enter the PID of the process to terminate: " pid
    
    if [[ -z "$pid" || ! "$pid" =~ ^[0-9]+$ ]]; then
        echo "Error: Invalid PID. Please enter a numeric value."
        log_action "Failed termination attempt: Invalid PID entered."
        return
    fi
    
    # Check if process exists
    if ! ps -p "$pid" > /dev/null 2>&1; then
        echo "Error: Process with PID $pid does not exist."
        log_action "Failed termination attempt: PID $pid not found."
        return
    fi
    
    # Get process name and check if critical
    proc_name=$(ps -p "$pid" -o comm= 2>/dev/null)
    proc_name_short=$(basename "$proc_name")
    
    for critical in "${CRITICAL_PROCESSES[@]}"; do
        if [[ "$proc_name_short" == "$critical" || "$proc_name" == *"$critical"* ]]; then
            echo "Error: Cannot terminate critical system process: $proc_name"
            log_action "Blocked termination attempt on critical process: $proc_name (PID: $pid)"
            return
        fi
    done
    
    echo "Process to terminate: $proc_name (PID: $pid)"
    read -p "Are you sure you want to terminate this process? (Y/N): " confirm
    
    if [[ "$confirm" == "Y" || "$confirm" == "y" ]]; then
        kill "$pid" 2>/dev/null
        if [[ $? -eq 0 ]]; then
            echo "Process $pid ($proc_name) terminated successfully."
            log_action "Terminated process: $proc_name (PID: $pid)"
        else
            echo "Error: Failed to terminate process $pid. You may need superuser privileges."
            log_action "Failed termination attempt: PID $pid with sudo required"
        fi
    else
        echo "Termination cancelled."
        log_action "Termination cancelled for PID $pid"
    fi
}

# ----------------------------------------------------------
# Function: Check disk usage and archive logs
# ----------------------------------------------------------
manage_logs() {
    echo
    echo "=========================================="
    echo "       LOG FILE MANAGEMENT"
    echo "=========================================="
    
    # Check sensor log directory disk usage
    if [[ -d "$SENSOR_LOG_DIR" ]]; then
        echo "Sensor log directory: $SENSOR_LOG_DIR"
        echo "Total size: $(du -sh "$SENSOR_LOG_DIR" 2>/dev/null | cut -f1)"
        echo
        echo "------------------------------------------"
        echo "   LARGE LOG FILES (> 50 MB)"
        echo "------------------------------------------"
        
        # Find files larger than 50MB in sensor_logs
        large_files=$(find "$SENSOR_LOG_DIR" -type f -size +50M -name "*.log" 2>/dev/null)
        
        if [[ -n "$large_files" ]]; then
            echo "$large_files" | while read -r file; do
                size=$(du -h "$file" 2>/dev/null | cut -f1)
                echo "File: $(basename "$file")"
                echo "  Size: $size"
                echo "  Path: $file"
                
                # Archive the file
                timestamp=$(date '+%Y%m%d_%H%M%S')
                archive_name="$(basename "$file" .log)_${timestamp}.tar.gz"
                archive_path="$ARCHIVE_DIR/$archive_name"
                
                echo "  Archiving to: $archive_path"
                tar -czf "$archive_path" -C "$(dirname "$file")" "$(basename "$file")" 2>/dev/null
                
                if [[ $? -eq 0 ]]; then
                    # Remove original file after successful archive
                    rm -f "$file"
                    echo "  Status: Archived and removed original"
                    log_action "Archived log file: $(basename "$file") to $archive_name"
                else
                    echo "  Status: Archive failed"
                    log_action "Archive failed for: $(basename "$file")"
                fi
                echo
            done
        else
            echo "No log files larger than 50 MB found."
        fi
        
        # Check archive directory size
        if [[ -d "$ARCHIVE_DIR" ]]; then
            archive_size=$(du -sb "$ARCHIVE_DIR" 2>/dev/null | cut -f1)
            if [[ "$archive_size" -gt 1073741824 ]]; then  # 1GB in bytes
                echo
                echo "=========================================="
                echo "  WARNING: Archive directory exceeds 1GB!"
                echo "=========================================="
                echo "Archive directory: $ARCHIVE_DIR"
                echo "Current size: $(du -sh "$ARCHIVE_DIR" 2>/dev/null | cut -f1)"
                echo "Please review and clean up old archives."
                log_action "WARNING: Archive directory exceeded 1GB"
            else
                echo
                echo "Archive directory size: $(du -sh "$ARCHIVE_DIR" 2>/dev/null | cut -f1)"
            fi
        fi
    else
        echo "Sensor log directory does not exist: $SENSOR_LOG_DIR"
    fi
}

# ----------------------------------------------------------
# Function: View system log
# ----------------------------------------------------------
view_log() {
    echo
    echo "=========================================="
    echo "         SYSTEM MONITOR LOG"
    echo "=========================================="
    if [[ -f "$SYSTEM_LOG" && -s "$SYSTEM_LOG" ]]; then
        tail -n 20 "$SYSTEM_LOG"
        echo
        echo "Log file: $SYSTEM_LOG"
        echo "Total entries: $(wc -l < "$SYSTEM_LOG")"
    else
        echo "No log entries found."
    fi
}

# ----------------------------------------------------------
# Main program
# ----------------------------------------------------------
while true
do
    echo
    echo "=================================================="
    echo "       SMART CAMPUS IoT DEVICE MANAGEMENT"
    echo "=================================================="
    echo "1. Display CPU Usage"
    echo "2. Display Memory Usage"
    echo "3. Show Top Memory-Consuming Processes"
    echo "4. Terminate a Process"
    echo "5. Manage Sensor Log Files (Archive >50MB)"
    echo "6. View System Activity Log"
    echo "7. Exit"
    echo "=================================================="
    read -p "Enter your choice: " choice
    
    case "$choice" in
        1)
            show_cpu_usage
            ;;
        2)
            show_memory_usage
            ;;
        3)
            show_top_processes
            ;;
        4)
            terminate_process
            ;;
        5)
            manage_logs
            ;;
        6)
            view_log
            ;;
        7)
            echo
            read -p "Are you sure you want to exit? (Y/N): " confirm
            if [[ "$confirm" == "Y" || "$confirm" == "y" ]]; then
                log_action "Administrator exited the system."
                echo "Goodbye!"
                break
            else
                echo "Exit cancelled."
            fi
            ;;
        *)
            echo "Invalid choice. Please select 1, 2, 3, 4, 5, 6, or 7."
            ;;
    esac
done
