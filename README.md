# Step-2-Start-the-Bash-script
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
CRITICAL_PROCESSES=("systemd" "init" "kthreadd" "sshd")
# Create required directories/files if they do not exist
mkdir -p "$SENSOR_LOG_DIR"
mkdir -p "$ARCHIVE_DIR"
touch "$SYSTEM_LOG"
# ----------------------------------------------------------
# Function: Write action to system log
# ----------------------------------------------------------
log_action() {
    local message="$1"
    echo "$(date '+%Y-%m-%d %H:%M:%S') - message">>"SYSTEM_LOG"
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
    log_action "Memory usage was displayed."
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
    echo "3. Exit"
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
            echo
            read -p "Are you sure you want to exit? (Y/N): " confirm
            if [[ "confirm"=="Y"||"confirm" == "y" ]]
            then
                log_action "Administrator exited the system."
                echo "Goodbye!"
                break
            else
                echo "Exit cancelled."
            fi
            ;;
        *)
            echo "Invalid choice. Please select 1, 2, or 3."
            ;;
    esac
Done
