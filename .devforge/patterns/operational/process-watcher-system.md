# Process Watcher System

## Intent
Provide automated monitoring and restart capabilities for long-running processes, ensuring system reliability through intelligent failure detection and recovery.

## Problem Statement
- **Silent process failures** - Processes crash or hang without notification
- **Manual intervention required** - Someone must manually restart failed services
- **Progress stalls undetected** - Processes appear running but make no progress
- **Resource waste** - Failed processes consume resources without producing results
- **Service unreliability** - Systems become unreliable due to unmonitored failures

What happens when this pattern is NOT used:
- Processes fail silently and stay failed
- Manual monitoring required to detect issues
- Data processing pipelines stall indefinitely
- System appears operational but produces no output
- Manual restart procedures for every failure

## Context & Applicability
- **Long-running data processing** pipelines
- **Background service management** for web applications
- **Batch processing systems** with progress tracking
- **Development environments** with multiple services
- **Any system requiring high availability**

Most beneficial for:
- Data processing pipelines running for hours/days
- Services that must maintain high uptime
- Development environments with complex service dependencies
- Systems where manual monitoring is impractical

## Solution Overview
Create an intelligent process watcher that:
- **Monitors process health** using multiple indicators
- **Detects various failure modes** (crashes, hangs, no progress)
- **Automatically restarts failed processes** with configurable policies
- **Tracks performance metrics** and failure patterns
- **Integrates with existing process management** (tmux, systemd, etc.)

Core principles:
- Multiple health indicators (process existence, progress, resource usage)
- Configurable restart policies and limits
- Comprehensive logging and metrics
- Non-intrusive monitoring approach
- Integration with standard process management tools

## Structure & Components

### High-level Architecture
```
Process Watcher
    ↓
Health Checkers → Process Existence
                → Progress Detection  
                → Resource Monitoring
    ↓
Restart Manager → Graceful Stop
               → Process Restart
               → Failure Tracking
    ↓
Monitored Processes
```

### Key Components
1. **Health Checkers** - Multiple ways to assess process health
2. **Progress Tracker** - Monitors work output or database changes
3. **Restart Manager** - Handles process lifecycle and restart policies
4. **Metrics Collector** - Tracks performance and failure data
5. **Configuration Manager** - Timeout settings, restart limits, etc.
6. **Integration Layer** - Works with tmux, systemd, containers

### Implementation Example
```python
from utils.process_watcher import ProcessWatcher

# Create watcher for a data processing script
watcher = ProcessWatcher(
    session_name="data_processor",
    script_command="python process_data.py --batch-size 1000",
    check_interval=60,        # Check every minute
    progress_timeout=300,     # Restart if no progress for 5 minutes
    max_restarts=5,          # Maximum automatic restarts
    auto_restart=True
)

# Start monitoring
watcher.run_monitoring()

# Custom progress checker
def check_database_progress():
    # Check database for new records
    return get_processed_record_count()

watcher.set_progress_checker(check_database_progress)
```

## Implementation Variations

### **Tmux-Based Monitoring**
- Monitors tmux sessions and processes
- Good for development environments
- Easy to implement and debug

### **Systemd Integration**
- Uses systemd for process management
- Production-ready monitoring
- Integrates with system logging

### **Container-Based Monitoring**
- Monitors Docker containers or Kubernetes pods
- Cloud-native approach
- Scales with orchestration systems

### **Database Progress Tracking**
- Uses database record counts for progress detection
- Accurate for data processing pipelines
- Works across process restarts

## Benefits & Trade-offs

**Advantages:**
- **Automatic recovery** - Systems self-heal from failures
- **Early failure detection** - Catch issues before they become critical
- **Reduced manual intervention** - Less need for human monitoring
- **Improved reliability** - Higher system uptime and availability
- **Performance insights** - Metrics help optimize processes
- **Development productivity** - Reliable services during development

**Disadvantages:**
- **Additional complexity** - Another system component to maintain
- **Resource overhead** - Monitoring consumes CPU and memory
- **False positives** - May restart healthy but slow processes
- **Configuration complexity** - Many parameters to tune properly

**Alternatives:**
- **External monitoring** - Nagios, Prometheus, DataDog
- **Process managers** - systemd, supervisor, pm2
- **Health check endpoints** - HTTP health checks for web services
- **Manual monitoring** - Human oversight and intervention

## Real-world Examples

### **Data Processing Pipeline**
- **Scale**: Multi-stage ETL processing millions of records
- **Implementation**: Tmux session monitoring with database progress tracking
- **Results**: 
  - 99.9% uptime during processing
  - Automatic recovery from temporary failures
  - Zero manual intervention required
- **Monitoring**: Process existence + database record counts

### **Web Service Stack**
- **Services**: API server, background workers, database
- **Implementation**: systemd integration with HTTP health checks
- **Benefits**: Automatic restart of failed services

### **Machine Learning Training**
- **Process**: Long-running model training jobs
- **Monitoring**: GPU utilization + model checkpoint progress
- **Recovery**: Restart from last checkpoint on failure

## Implementation Guidance

### **Step 1: Identify Health Indicators**
```python
# Define what constitutes "healthy" for your process
health_checks = [
    ProcessExistenceCheck(),
    ProgressCheck(database_table="results", timeout=300),
    ResourceCheck(max_memory_gb=8, max_cpu_percent=90)
]
```

### **Step 2: Implement Progress Detection**
```python
def check_progress():
    # For data processing: check output records
    current_count = get_database_record_count()
    
    # For web services: check HTTP endpoint
    response = requests.get("http://localhost:8080/health")
    
    # For file processing: check output files
    return len(list(output_dir.glob("*.csv")))
```

### **Step 3: Configure Restart Policies**
```python
restart_policy = {
    'max_restarts': 5,
    'backoff_strategy': 'exponential',  # 1s, 2s, 4s, 8s, 16s
    'grace_period': 10,  # Seconds to wait for graceful shutdown
    'failure_threshold': 3  # Consecutive failures before marking unhealthy
}
```

### **Step 4: Add Monitoring and Metrics**
```python
# Track metrics for optimization
metrics = {
    'restart_count': 0,
    'total_downtime': timedelta(0),
    'average_recovery_time': timedelta(seconds=30),
    'failure_patterns': []
}
```

### **Common Pitfalls**
- **Too aggressive restarts** - May restart healthy but slow processes
- **Resource monitoring** - Watch for monitoring overhead
- **Cascade failures** - One restart triggering others
- **State preservation** - Ensure process state isn't lost on restart

## Related Patterns

### **Complementary Patterns**
- **Centralized PID Management** - Process tracking foundation
- **Health Check Endpoints** - Application-level health reporting
- **Circuit Breaker** - Prevents cascade failures
- **Graceful Degradation** - System continues operating with reduced functionality

### **Conflicting Patterns**
- **Fail-Fast** - Immediate failure vs automatic recovery
- **Stateless Architecture** - If processes are truly stateless, simple restart may be sufficient

### **Effective Combinations**
- **Process Watcher + PID Management** - Complete process lifecycle management
- **Process Watcher + Centralized Logging** - Comprehensive failure analysis
- **Process Watcher + Health Checks** - Multi-layer reliability

## DevForge Integration

### **Implementing Template**
- **`templates/process-watcher/`** - Complete Python implementation
- **Features**: 
  - Configurable monitoring system
  - Multiple health check strategies
  - Integration with tmux and systemd
  - Comprehensive metrics and logging

### **Related DevForge Patterns**
- **Centralized PID Management** - Foundation for process tracking
- **Service Orchestration** - Coordinates multiple monitored services
- **Observability** - Monitoring and metrics collection

### **Future Template Opportunities**
- **Cloud-native versions** - Kubernetes liveness/readiness probes
- **Multi-language implementations** - Go, Java, Node.js versions
- **Integration templates** - Prometheus, Grafana, ELK stack integration

## References

- **Site Reliability Engineering** - Google's approach to service monitoring
- **Chaos Engineering** - Testing system resilience
- **Process Management Best Practices** - Unix process lifecycle management
- **Observability Patterns** - Modern monitoring and alerting approaches

---

*Pattern extracted from high-reliability data processing systems handling millions of records with automatic failure recovery and zero manual intervention.*
