# Stage Runner Management Pattern

**Pattern Type**: Workflow  
**Category**: Process Management  
**Maturity**: Stable  
**Last Updated**: 2025-01-10  

## Intent

Provide standardized process lifecycle management for multi-stage data processing pipelines through comprehensive shell script wrappers that support start/stop/status/logs commands, PID file management, and crash-resistant tmux session handling.

## Problem Statement

### Core Problems
- **Process Management Chaos**: Inconsistent ways to start, stop, and monitor long-running processes
- **Orphaned Processes**: Processes continue running after crashes with no way to track or stop them
- **Status Invisibility**: No standardized way to check if processes are running or stopped
- **Resource Leaks**: Failed processes leave behind tmux sessions and resource locks
- **Recovery Complexity**: Difficult to restart failed processes or recover from crashes

### Business Impact
- **Resource Waste**: Orphaned processes consume CPU, memory, and I/O resources
- **Operational Overhead**: Manual process hunting and cleanup across development sessions
- **Deployment Fragility**: Inconsistent process management breaks automated deployments
- **Debugging Difficulty**: No standardized way to access process logs or status
- **System Instability**: Accumulated orphaned processes can destabilize development environments

## Context

### When to Apply
- **Multi-Stage Pipelines**: Projects with sequential processing stages
- **Long-Running Processes**: Data processing, web servers, background workers
- **Development Environments**: AI agent development prone to crashes
- **Team Environments**: Shared development systems with multiple processes
- **Production Systems**: Any system requiring reliable process management

### When NOT to Apply
- **Simple Scripts**: Single-execution utilities with no lifecycle management needs
- **System Services**: Processes managed by systemd, supervisor, or similar
- **Container Environments**: Processes managed by orchestration systems

## Solution Overview

The Stage Runner Management Pattern provides standardized shell script wrappers that:

1. **Lifecycle Management**: Consistent start/stop/status/logs commands
2. **PID File Tracking**: Store process and tmux session PIDs for reliable management
3. **Crash Resistance**: Tmux sessions survive terminal and IDE crashes
4. **Resource Cleanup**: Proper cleanup of PIDs and sessions on stop
5. **Status Visibility**: Clear reporting of process and session states

## Structure & Components

### Core Implementation

```bash
#!/bin/bash
# run_stage_template.sh - Template for all stage runners

# MANDATORY: Change to script directory
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
cd "$SCRIPT_DIR"

# Configuration
STAGE_NAME="stage1"
PYTHON_SCRIPT="stage1_processor.py"
TMUX_SESSION="${STAGE_NAME}_processing"
PROCESS_PID_FILE="${STAGE_NAME}.pid"
TMUX_PID_FILE="tmux_${STAGE_NAME}.pid"
LOG_FILE="../logs/${STAGE_NAME}.log"

# MANDATORY: Virtual environment setup
setup_venv() {
    if [ ! -f "requirements.txt" ]; then
        echo "ERROR: requirements.txt not found in $SCRIPT_DIR"
        exit 1
    fi
    
    if [ ! -d "venv" ]; then
        echo "Creating virtual environment..."
        python3 -m venv venv
    fi
    
    source venv/bin/activate
    pip install -r requirements.txt
}

# Start process in tmux session
start_process() {
    if is_running; then
        echo "Process already running (PID: $(cat $PROCESS_PID_FILE))"
        return 0
    fi
    
    setup_venv
    
    echo "Starting $STAGE_NAME process..."
    tmux new-session -d -s "$TMUX_SESSION" \
        "source venv/bin/activate; python3 $PYTHON_SCRIPT 2>&1 | tee $LOG_FILE"
    
    # Store tmux session PID
    tmux list-sessions -F "#{session_name} #{session_id}" | \
        grep "^$TMUX_SESSION " | \
        cut -d' ' -f2 | \
        sed 's/\$//' > "$TMUX_PID_FILE"
    
    # Get Python process PID (may take a moment to start)
    sleep 2
    PYTHON_PID=$(tmux list-panes -t "$TMUX_SESSION" -F "#{pane_pid}")
    echo "$PYTHON_PID" > "$PROCESS_PID_FILE"
    
    echo "Started $STAGE_NAME in tmux session: $TMUX_SESSION"
    echo "Process PID: $PYTHON_PID"
    echo "Log file: $LOG_FILE"
}

# Stop process and cleanup
stop_process() {
    if [ ! -f "$PROCESS_PID_FILE" ] && [ ! -f "$TMUX_PID_FILE" ]; then
        echo "No PID files found - process may not be running"
        return 0
    fi
    
    echo "Stopping $STAGE_NAME process..."
    
    # Kill Python process if PID file exists
    if [ -f "$PROCESS_PID_FILE" ]; then
        PID=$(cat "$PROCESS_PID_FILE")
        if kill -0 "$PID" 2>/dev/null; then
            kill "$PID"
            echo "Killed process PID: $PID"
        fi
        rm -f "$PROCESS_PID_FILE"
    fi
    
    # Kill tmux session
    if [ -f "$TMUX_PID_FILE" ]; then
        tmux kill-session -t "$TMUX_SESSION" 2>/dev/null
        rm -f "$TMUX_PID_FILE"
        echo "Killed tmux session: $TMUX_SESSION"
    fi
    
    echo "$STAGE_NAME process stopped"
}

# Check if process is running
is_running() {
    if [ ! -f "$PROCESS_PID_FILE" ]; then
        return 1
    fi
    
    PID=$(cat "$PROCESS_PID_FILE")
    kill -0 "$PID" 2>/dev/null
    return $?
}

# Show process status
show_status() {
    echo "=== $STAGE_NAME Status ==="
    
    if is_running; then
        PID=$(cat "$PROCESS_PID_FILE")
        echo "Status: RUNNING"
        echo "Process PID: $PID"
        echo "Memory usage: $(ps -o rss= -p $PID 2>/dev/null | awk '{print $1/1024 " MB"}' || echo 'N/A')"
        echo "CPU usage: $(ps -o %cpu= -p $PID 2>/dev/null || echo 'N/A')"
    else
        echo "Status: STOPPED"
    fi
    
    # Tmux session status
    if tmux has-session -t "$TMUX_SESSION" 2>/dev/null; then
        echo "Tmux session: ACTIVE ($TMUX_SESSION)"
    else
        echo "Tmux session: INACTIVE"
    fi
    
    # Log file status
    if [ -f "$LOG_FILE" ]; then
        echo "Log file: $LOG_FILE ($(wc -l < "$LOG_FILE") lines)"
        echo "Last log entry: $(tail -n 1 "$LOG_FILE" 2>/dev/null || echo 'N/A')"
    else
        echo "Log file: NOT FOUND"
    fi
}

# Show recent logs
show_logs() {
    local lines=${1:-50}
    if [ -f "$LOG_FILE" ]; then
        echo "=== Last $lines lines of $STAGE_NAME logs ==="
        tail -n "$lines" "$LOG_FILE"
    else
        echo "Log file not found: $LOG_FILE"
    fi
}

# Main command dispatch
case "${1:-}" in
    start)
        start_process
        ;;
    stop)
        stop_process
        ;;
    restart)
        stop_process
        sleep 2
        start_process
        ;;
    status)
        show_status
        ;;
    logs)
        show_logs "${2:-50}"
        ;;
    start-foreground)
        setup_venv
        python3 "$PYTHON_SCRIPT"
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status|logs [lines]|start-foreground}"
        echo ""
        echo "Commands:"
        echo "  start            - Start process in tmux session"
        echo "  stop             - Stop process and cleanup"
        echo "  restart          - Stop then start process"
        echo "  status           - Show process status and resource usage"
        echo "  logs [lines]     - Show recent log entries (default: 50)"
        echo "  start-foreground - Run process in foreground (no tmux)"
        exit 1
        ;;
esac
```

### PID File Management

```bash
# PID file naming convention
PROCESS_PID_FILE="${STAGE_NAME}.pid"        # Main process PID
TMUX_PID_FILE="tmux_${STAGE_NAME}.pid"      # Tmux session PID

# PID file operations
store_pid() {
    local pid=$1
    local pid_file=$2
    echo "$pid" > "$pid_file"
}

read_pid() {
    local pid_file=$1
    if [ -f "$pid_file" ]; then
        cat "$pid_file"
    fi
}

cleanup_pid_files() {
    rm -f "$PROCESS_PID_FILE" "$TMUX_PID_FILE"
}

# Process validation
is_process_running() {
    local pid_file=$1
    if [ ! -f "$pid_file" ]; then
        return 1
    fi
    
    local pid=$(cat "$pid_file")
    kill -0 "$pid" 2>/dev/null
    return $?
}
```

### Integration with Virtual Environments

```bash
# Virtual environment management
setup_stage_venv() {
    local stage_dir=$1
    local requirements_file="$stage_dir/requirements.txt"
    local venv_dir="$stage_dir/venv"
    
    if [ ! -f "$requirements_file" ]; then
        echo "ERROR: $requirements_file not found"
        exit 1
    fi
    
    if [ ! -d "$venv_dir" ]; then
        echo "Creating virtual environment in $venv_dir..."
        python3 -m venv "$venv_dir"
    fi
    
    source "$venv_dir/bin/activate"
    pip install -r "$requirements_file"
}
```

## Implementation Variations

### Basic Implementation
- Simple start/stop commands
- Basic PID file management
- Minimal status reporting

### Enhanced Implementation
- Comprehensive status reporting with resource usage
- Log rotation and management
- Health check integration
- Automatic restart on failure

### Enterprise Implementation
- Integration with monitoring systems
- Automated deployment support
- Service discovery integration
- Centralized logging and metrics

## Benefits & Trade-offs

### Benefits
- **Operational Consistency**: Standardized commands across all stages
- **Resource Management**: Proper cleanup prevents resource leaks
- **Crash Resistance**: Tmux sessions survive development environment crashes
- **Debugging Support**: Easy access to logs and status information
- **Team Collaboration**: Consistent process management across developers

### Trade-offs
- **Complexity**: More complex than simple script execution
- **Learning Curve**: Developers must learn the command patterns
- **Resource Overhead**: Tmux sessions consume additional memory
- **File Management**: PID files require cleanup and maintenance

### Performance Impact
- **Positive**: Prevents resource leaks and orphaned processes
- **Negative**: Slight overhead from tmux and PID file management
- **Net Result**: Improved system stability and resource utilization

## Real-world Examples

### Texas RRC Lease Name Index Project

#### Stage Runner Implementation
```bash
# All stage runners follow the pattern
stage0/run_stage0.sh     # Web server management
stage1a/run_stage1a.sh   # URL discovery
stage1b/run_stage1b.sh   # PDF URL extraction
stage2/run_stage2.sh     # PDF download
stage3/run_stage3.sh     # PDF processing
stage4/run_stage4.sh     # LLM processing
stage5/run_stage5.sh     # Data consolidation
```

#### Usage Examples
```bash
# Start Stage 0 web server
./stage0/run_stage0.sh start

# Check status of all stages
for stage in stage0 stage1a stage1b stage2 stage3 stage4 stage5; do
    echo "=== $stage ==="
    ./$stage/run_$stage.sh status
done

# View recent logs
./stage4/run_stage4.sh logs 100

# Stop all processes
for stage in stage0 stage1a stage1b stage2 stage3 stage4 stage5; do
    ./$stage/run_$stage.sh stop
done
```

#### PID File Management
```bash
# Example PID files created
stage0/rrc_web.pid           # Web server process
stage0/tmux_rrc_web.pid      # Tmux session
stage4/stage4_processing.pid # Processing script
stage4/tmux_stage4.pid       # Tmux session
```

## Implementation Guidance

### Step 1: Create Template Script
```bash
# Create template from the core implementation above
cp templates/shell/stage-runner-template.sh stage1/run_stage1.sh
```

### Step 2: Customize for Each Stage
```bash
# Update configuration variables
STAGE_NAME="stage1"
PYTHON_SCRIPT="stage1_processor.py"
TMUX_SESSION="stage1_processing"
```

### Step 3: Implement Required Commands
```bash
# Test all commands work correctly
./run_stage1.sh start
./run_stage1.sh status
./run_stage1.sh logs
./run_stage1.sh stop
```

### Step 4: Add Stage-Specific Features
```bash
# Add custom arguments, health checks, etc.
case "$1" in
    start)
        # Add stage-specific startup logic
        ;;
    custom-command)
        # Add stage-specific commands
        ;;
esac
```

### Step 5: Test Crash Recovery
```bash
# Test tmux session survival
./run_stage1.sh start
# Kill terminal/IDE
# Reconnect and verify process still running
./run_stage1.sh status
```

### Common Pitfalls
- **PID File Cleanup**: Always clean up PID files on stop
- **Process Validation**: Check if PIDs are still valid before using
- **Tmux Session Names**: Use unique names to avoid conflicts
- **Virtual Environment**: Always activate venv before running Python

## Related Patterns

### Complementary Patterns
- **Mandatory Development Environment**: Provides the foundation for stage runners
- **Centralized Path Management**: Used for log files and PID file locations
- **Modular Checklist System**: Tracks stage runner implementation progress

### Alternative Patterns
- **Systemd Services**: System-level service management
- **Docker Containers**: Container-based process isolation
- **Process Managers**: PM2, supervisor, or similar tools

## DevForge Integration

### Pattern Dependencies
- **Prerequisites**: bash, tmux, Python 3.8+
- **Complements**: Development environment patterns
- **Enables**: Pipeline automation and monitoring patterns

### Automation Support
- **Template Generation**: Automated creation of stage runner scripts
- **Health Monitoring**: Integration with monitoring systems
- **Deployment Automation**: Standardized deployment procedures

### Monitoring Integration
- **Status Dashboards**: Centralized view of all stage statuses
- **Log Aggregation**: Centralized logging from all stages
- **Performance Metrics**: Resource usage tracking across stages

## Metrics & Success Criteria

### Quantitative Metrics
- **Process Uptime**: Percentage of time processes run without crashes
- **Resource Utilization**: CPU and memory usage efficiency
- **Recovery Time**: Time to restart failed processes
- **Orphaned Process Count**: Number of processes left running after crashes

### Qualitative Indicators
- **Operational Simplicity**: Easy to start, stop, and monitor processes
- **Team Productivity**: Reduced time spent on process management
- **System Stability**: Fewer resource leaks and system instability
- **Debugging Efficiency**: Faster access to logs and process status

---

**Implementation Priority**: CRITICAL - Essential for any multi-stage processing pipeline or long-running process management.

**Success Pattern**: Implement template first, then systematically apply to all stages with consistent command patterns and PID file management. 