# Centralized PID Management

## Intent
Provide standardized PID file management for multi-process applications, enabling reliable process tracking, monitoring, and cleanup across development and production environments.

## Problem Statement
- **Scattered PID files** - Different processes create PID files in various locations
- **Inconsistent naming** - No standard for PID file naming conventions
- **Manual cleanup** - Stale PID files accumulate when processes crash
- **Process tracking complexity** - Hard to monitor which services are running
- **Service management chaos** - No unified way to start/stop/status check services

What happens when this pattern is NOT used:
- PID files scattered across `/tmp`, `/var/run`, project directories
- Stale PID files pointing to dead processes
- No reliable way to check if services are actually running
- Manual process cleanup when things go wrong
- Difficulty orchestrating multiple related processes

## Context & Applicability
- **Multi-service applications** with several long-running processes
- **Development environments** needing reliable service management
- **Production deployments** requiring process monitoring
- **CI/CD pipelines** needing clean service lifecycle management
- **Any application using tmux, background processes, or daemons**

Most beneficial for:
- Applications with 3+ long-running processes
- Development teams needing reliable service start/stop
- Projects using tmux sessions for process management
- Applications requiring process health monitoring

## Solution Overview
Create a centralized PID management system that:
- **Standardizes PID file locations** in a single directory (e.g., `local/pid/`)
- **Provides consistent naming** using service names and types
- **Tracks multiple PID types** per service (main process, tmux session, workers)
- **Offers process lifecycle management** (start, stop, status, cleanup)
- **Automatically handles cleanup** of stale PID files

Core principles:
- Single directory for all PID files
- Consistent naming convention: `{{service_name}}.pid`, `tmux_{{service_name}}.pid`
- Process existence validation
- Automatic stale PID cleanup
- Service-oriented management

## Structure & Components

### High-level Architecture
```
Application Services
       ↓
PID Manager
       ↓
Centralized PID Directory
       ↓
File System
```

### Key Components
1. **PID Directory** - Single location for all PID files (`local/pid/`)
2. **PID Manager Class** - Core management functionality
3. **Service Registry** - Track multiple services and their PID types
4. **Process Validator** - Check if processes are actually running
5. **Cleanup Manager** - Remove stale PID files automatically
6. **Service Controller** - Start/stop/status operations

### Implementation Example
```python
from utils.pid_management import PIDManager, write_pid, read_pid

# Initialize manager
pid_manager = PIDManager()

# Write PID for a service
pid = os.getpid()
pid_manager.write_pid("web_server", pid, "process")
pid_manager.write_pid("web_server", tmux_pid, "tmux")

# Check service status
status = pid_manager.get_process_status("web_server")
print(f"Service running: {{status['process_running']}}")

# Stop service and cleanup
pid_manager.stop_service("web_server")

# List all active services
active_services = pid_manager.list_active_services()
```

## Implementation Variations

### **Simple File-Based Approach**
- Basic PID files in centralized directory
- Manual process checking
- Suitable for small applications

### **Class-Based Management**
- Full PID manager with automatic cleanup
- Process validation and health checking
- Better for complex multi-service applications

### **Service-Oriented Architecture**
- Named services with multiple PID types
- Hierarchical process management
- Advanced orchestration capabilities

### **Integration with Process Managers**
- Works with systemd, supervisor, pm2
- Hybrid approach using both standard tools and custom PID management
- Production-ready service management

## Benefits & Trade-offs

**Advantages:**
- **Centralized management** - All PID files in one location
- **Consistent interface** - Same API for all services
- **Automatic cleanup** - No stale PID files accumulating
- **Process validation** - Know if processes are actually running
- **Service orchestration** - Easy to manage multiple related processes
- **Development productivity** - Reliable service start/stop during development

**Disadvantages:**
- **Additional abstraction** - One more layer to understand
- **Dependency creation** - Services depend on PID management system
- **Initial setup** - Requires migrating existing PID handling
- **Platform specifics** - Signal handling varies between OS

**Alternatives:**
- **systemd/supervisor** - Production process managers
- **Docker containers** - Container-based process isolation
- **Process monitoring tools** - External monitoring systems
- **Manual PID files** - Simple but inconsistent approach

## Real-world Examples

### **Data Processing Pipeline**
- **Scale**: 5+ stage processors, tmux sessions, monitoring processes
- **Implementation**: Python PID manager with service categories
- **Results**: 
  - Zero stale PID files during development
  - Reliable service restart capabilities
  - Easy process health monitoring
- **Source**: Data processing pipeline implementations

### **Web Application Stack**
- **Services**: Web server, database, Redis, worker processes
- **Management**: Unified start/stop for entire stack
- **Benefits**: Consistent development environment setup

### **Machine Learning Pipeline**
- **Components**: Training processes, model servers, monitoring
- **Coordination**: PID-based process orchestration
- **Advantage**: GPU resource management and process cleanup

## Implementation Guidance

### **Step 1: Create PID Directory Structure**
```python
# Create centralized PID directory
from pathlib import Path
pid_dir = Path("local/pid")
pid_dir.mkdir(parents=True, exist_ok=True)
```

### **Step 2: Implement Core PID Manager**
```python
class PIDManager:
    def write_pid(self, service_name: str, pid: int, pid_type: str = "process"):
        # Write PID to standardized file location
        
    def read_pid(self, service_name: str, pid_type: str = "process"):
        # Read PID from file with validation
        
    def is_process_running(self, pid: int):
        # Check if process actually exists
```

### **Step 3: Migrate Existing Services**
```python
# Before
with open('/tmp/myapp.pid', 'w') as f:
    f.write(str(os.getpid()))

# After
write_pid("myapp", os.getpid(), "process")
```

### **Step 4: Add Service Management**
```python
# Service lifecycle management
pid_manager.stop_service("myapp")  # Graceful stop
pid_manager.stop_service("myapp", force=True)  # Force kill
status = pid_manager.get_process_status("myapp")
```

### **Common Pitfalls**
- **Forgetting cleanup** - Always remove PID files on exit
- **Not validating PIDs** - Check if process actually exists
- **Signal handling** - Handle different termination signals properly
- **Race conditions** - Use file locking for concurrent access

## Related Patterns

### **Complementary Patterns**
- **Process Watcher System** - Monitors processes using PID management
- **Service Discovery** - Uses PID management for service registration
- **Configuration Management** - Coordinates with service management

### **Conflicting Patterns**
- **Container-based Architecture** - Docker handles process management
- **Serverless Architecture** - No long-running processes to manage

### **Effective Combinations**
- **PID Management + Process Watcher** - Complete process lifecycle management
- **PID Management + Centralized Logging** - Unified service monitoring
- **PID Management + Health Checks** - Comprehensive service reliability

## DevForge Integration

### **Implementing Template**
- **`templates/pid-management/`** - Complete Python implementation
- **Features**: 
  - Ready-to-use PID manager class
  - Service lifecycle scripts
  - Health monitoring utilities
  - Integration examples

### **Related DevForge Patterns**
- **Process Watcher System** - Builds on PID management foundation
- **Centralized Resource Management** - Uses consistent directory structure
- **Service Orchestration** - Enables reliable multi-service coordination

### **Future Template Opportunities**
- **Multi-language implementations** - Go, Node.js, Java versions
- **Container integration** - Docker + PID management hybrid
- **Cloud-native versions** - Kubernetes pod management integration

## References

- **Unix Process Management** - Signal handling and process lifecycle
- **System Administration Best Practices** - PID file conventions
- **Service Management Patterns** - systemd and supervisor approaches
- **Production Deployment** - Process monitoring and reliability patterns

---

*Pattern extracted and generalized from high-reliability data processing pipeline implementations. Proven in production environments processing millions of records with zero manual intervention.*
