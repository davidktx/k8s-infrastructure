# PYTHONPATH Management in Shell Scripts

## Intent
Provide standardized PYTHONPATH management for shell scripts and development environments, preventing common issues like duplicate paths, improper formatting, and import conflicts across multi-stage project architectures.

## Problem Statement
- **Duplicate paths** - Same directory added multiple times to PYTHONPATH
- **Leading/trailing colons** - Malformed PYTHONPATH causing import issues
- **Inconsistent path resolution** - Relative vs absolute paths causing confusion
- **Multi-stage conflicts** - Different project stages overriding each other's paths
- **Development vs production** - Different PYTHONPATH needs for different environments

What happens when this pattern is NOT used:
- Import errors in shell scripts and Python processes
- Inconsistent behavior between development and production
- Duplicate paths causing performance issues
- Manual path management leading to errors
- Hard-to-debug import resolution problems

## Context & Applicability
- **Multi-stage projects** with complex directory structures
- **Development environments** using shell scripts for automation
- **Virtual environments** needing proper path management
- **CI/CD pipelines** requiring consistent Python imports
- **Projects with stage-based architectures** (stage0, stage1, etc.)

Most beneficial for:
- Projects with 3+ stages or modules
- Development teams using shell scripts for process management
- Applications requiring consistent import paths across environments
- Projects mixing relative and absolute imports

## Solution Overview
Create a standardized PYTHONPATH management system that:
- **Eliminates duplicate paths** through deduplication logic
- **Handles relative path resolution** consistently
- **Provides environment-specific configurations** (dev, prod, test)
- **Integrates with virtual environments** seamlessly
- **Offers debugging and validation** capabilities

Core principles:
- Deduplicate paths automatically
- Use absolute paths for consistency
- Environment-aware configuration
- Fail-safe error handling
- Comprehensive logging and debugging

## Structure & Components

### High-level Architecture
```
Shell Script Entry Point
    ↓
PYTHONPATH Manager
    ↓
Path Deduplication → Path Validation → Environment Setup
    ↓
Python Process Launch
```

### Key Components
1. **Path Deduplication** - Remove duplicate entries automatically
2. **Path Validation** - Ensure paths exist and are accessible
3. **Environment Detection** - Adapt to dev/prod/test environments
4. **Relative Path Resolution** - Convert relative to absolute paths
5. **Virtual Environment Integration** - Work with venv/conda
6. **Debug Mode** - Comprehensive logging and validation

### Implementation Example
```bash
#!/bin/bash
# PYTHONPATH management for multi-stage projects

# Source the PYTHONPATH manager
source "$(dirname "$0")/utils/pythonpath_manager.sh"

# Initialize PYTHONPATH for this project
init_pythonpath() {
    local project_root="$1"
    local stage="$2"
    
    # Clear existing PYTHONPATH to start fresh
    export PYTHONPATH=""
    
    # Add project root
    add_to_pythonpath "$project_root"
    
    # Add stage-specific paths
    if [[ -d "$project_root/$stage" ]]; then
        add_to_pythonpath "$project_root/$stage"
    fi
    
    # Add common utilities
    add_to_pythonpath "$project_root/utils"
    
    # Validate final PYTHONPATH
    validate_pythonpath
}

# Usage
PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
init_pythonpath "$PROJECT_ROOT" "stage0"
```

## Implementation Variations

### **Basic Shell Function Approach**
```bash
# Simple PYTHONPATH utilities
add_to_pythonpath() {
    local path="$1"
    path="$(readlink -f "$path")"  # Convert to absolute
    
    if [[ -d "$path" ]]; then
        if [[ ":$PYTHONPATH:" != *":$path:"* ]]; then
            export PYTHONPATH="${PYTHONPATH:+$PYTHONPATH:}$path"
        fi
    fi
}

remove_from_pythonpath() {
    local path="$1"
    path="$(readlink -f "$path")"
    export PYTHONPATH="$(echo "$PYTHONPATH" | sed "s|:$path||g" | sed "s|$path:||g" | sed "s|^:||" | sed "s|:$||")"
}
```

### **Advanced Manager Class**
```bash
# Full-featured PYTHONPATH manager
pythonpath_manager() {
    local action="$1"
    shift
    
    case "$action" in
        "init")
            _pythonpath_init "$@"
            ;;
        "add")
            _pythonpath_add "$@"
            ;;
        "remove")
            _pythonpath_remove "$@"
            ;;
        "validate")
            _pythonpath_validate "$@"
            ;;
        "debug")
            _pythonpath_debug "$@"
            ;;
        *)
            echo "Usage: pythonpath_manager {init|add|remove|validate|debug}"
            return 1
            ;;
    esac
}
```

### **Environment-Specific Configuration**
```bash
# Different PYTHONPATH for different environments
setup_environment_pythonpath() {
    local env="${1:-development}"
    local project_root="$2"
    
    case "$env" in
        "development")
            add_to_pythonpath "$project_root"
            add_to_pythonpath "$project_root/stage0"
            add_to_pythonpath "$project_root/utils"
            add_to_pythonpath "$project_root/tests"
            ;;
        "production")
            add_to_pythonpath "$project_root"
            add_to_pythonpath "$project_root/stage0"
            add_to_pythonpath "$project_root/utils"
            ;;
        "testing")
            add_to_pythonpath "$project_root"
            add_to_pythonpath "$project_root/tests"
            add_to_pythonpath "$project_root/mocks"
            ;;
    esac
}
```

## Benefits & Trade-offs

**Advantages:**
- **Eliminates duplicate paths** - No performance impact from redundant entries
- **Consistent behavior** - Same imports work across all environments
- **Debugging support** - Easy to validate and troubleshoot PYTHONPATH issues
- **Environment flexibility** - Different setups for dev/prod/test
- **Integration friendly** - Works with virtual environments and CI/CD

**Disadvantages:**
- **Additional complexity** - More setup code required
- **Shell script dependency** - Requires bash/shell scripting knowledge
- **Path resolution overhead** - Extra processing for path validation
- **Environment assumptions** - May not work in all shell environments

**Alternatives:**
- **Python setup.py** - Package-based path management
- **Virtual environment only** - Rely on venv/conda paths
- **IDE configuration** - Let development tools handle paths
- **Docker containers** - Containerized environments with fixed paths

## Real-world Examples

### **Multi-Stage Data Processing Pipeline**
- **Challenge**: 5 stages each needing to import from parent and utils
- **Solution**: Stage-aware PYTHONPATH management
- **Result**: Zero import conflicts, consistent behavior across stages
- **Source**: Texas lease data processing project

### **Development Environment Setup**
- **Problem**: Different developers having different import issues
- **Implementation**: Standardized PYTHONPATH initialization script
- **Benefits**: Consistent development environment across team

### **CI/CD Pipeline Integration**
- **Context**: Automated testing requiring proper Python paths
- **Solution**: Environment-specific PYTHONPATH configuration
- **Outcome**: Reliable automated testing with proper imports

## Implementation Guidance

### **Step 1: Create PYTHONPATH Utilities**
```bash
# Create utils/pythonpath_manager.sh
mkdir -p utils
cat > utils/pythonpath_manager.sh << 'EOF'
#!/bin/bash
# PYTHONPATH management utilities

add_to_pythonpath() {
    local path="$1"
    path="$(readlink -f "$path" 2>/dev/null || echo "$path")"
    
    if [[ -d "$path" ]]; then
        if [[ ":$PYTHONPATH:" != *":$path:"* ]]; then
            export PYTHONPATH="${PYTHONPATH:+$PYTHONPATH:}$path"
        fi
    fi
}

validate_pythonpath() {
    echo "PYTHONPATH validation:"
    IFS=':' read -ra PATHS <<< "$PYTHONPATH"
    for path in "${PATHS[@]}"; do
        if [[ -n "$path" ]]; then
            if [[ -d "$path" ]]; then
                echo "  ✅ $path"
            else
                echo "  ❌ $path (not found)"
            fi
        fi
    done
}
EOF
```

### **Step 2: Update Shell Scripts**
```bash
# Add to existing shell scripts
source "$(dirname "$0")/utils/pythonpath_manager.sh"

# Initialize PYTHONPATH
PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
add_to_pythonpath "$PROJECT_ROOT"
add_to_pythonpath "$PROJECT_ROOT/stage0"
add_to_pythonpath "$PROJECT_ROOT/utils"
```

### **Step 3: Environment-Specific Setup**
```bash
# Create environment-specific scripts
# dev_environment.sh
source utils/pythonpath_manager.sh
setup_environment_pythonpath "development" "$PROJECT_ROOT"

# prod_environment.sh
source utils/pythonpath_manager.sh
setup_environment_pythonpath "production" "$PROJECT_ROOT"
```

### **Common Pitfalls**
- **Relative paths** - Always convert to absolute paths
- **Missing validation** - Check if paths exist before adding
- **Environment conflicts** - Clear PYTHONPATH before setting
- **Debugging difficulty** - Always provide validation/debug commands

## Related Patterns

### **Complementary Patterns**
- **Centralized Resource Management** - Consistent directory structure
- **Process Watcher System** - Monitor processes with proper PYTHONPATH
- **Configuration Management** - Environment-specific settings

### **Conflicting Patterns**
- **Package-based Imports** - Using setup.py instead of PYTHONPATH
- **Container-based Isolation** - Docker managing all paths

### **Effective Combinations**
- **PYTHONPATH + Virtual Environments** - Best of both worlds
- **PYTHONPATH + Process Management** - Consistent paths for all processes
- **PYTHONPATH + CI/CD** - Reproducible testing environments

## DevForge Integration

### **Template Implementation**
- **`templates/pythonpath-management/`** - Complete shell script utilities
- **Features**:
  - Ready-to-use PYTHONPATH manager functions
  - Environment-specific configuration templates
  - Validation and debugging utilities
  - Integration examples for common use cases

### **Related DevForge Patterns**
- **Centralized Resource Management** - Consistent directory structure foundation
- **Process Watcher System** - Ensures processes have proper PYTHONPATH
- **Configuration Management** - Environment-specific path management

### **Future Template Opportunities**
- **Python-based PYTHONPATH manager** - More advanced path management
- **Docker integration** - Container-aware path management
- **IDE integration** - Development environment configuration

## References

- **Python Path Management** - Official Python documentation on sys.path
- **Shell Scripting Best Practices** - Proper variable handling and path management
- **Virtual Environment Integration** - Working with venv and conda environments
- **CI/CD Path Management** - Reproducible builds and testing

---

*Pattern extracted from real-world multi-stage data processing pipeline where PYTHONPATH duplication and formatting issues caused import conflicts and system instability. Proven solution for complex Python project architectures.* 