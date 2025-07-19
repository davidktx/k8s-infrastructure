# Centralized Resource Management

## Intent
Provide a single source of truth for all resource locations (paths, URLs, connections) in a project, eliminating hardcoded references and ensuring consistent access across all modules.

## Problem Statement
- **Hardcoded paths scattered throughout codebase** - difficult to maintain and modify
- **Environment-specific paths** - code breaks when moved between development, testing, and production
- **Inconsistent resource access** - different modules use different approaches to find the same resources
- **Difficult refactoring** - changing directory structure requires updating multiple files
- **Testing challenges** - hard to redirect resources for testing without modifying production code

What happens when this pattern is NOT used:
- 50+ hardcoded paths scattered across 20+ files
- Breaking changes when moving between environments
- Difficult to relocate files or change directory structure
- Testing requires complex mocking or temporary file manipulation

## Context & Applicability
- **Multi-module projects** with shared resources
- **Cross-environment deployment** (dev/test/prod)
- **Projects with complex directory structures**
- **Applications requiring testing isolation**
- **Teams needing consistent development practices**

Most beneficial for:
- Projects with 5+ modules accessing shared resources
- Applications deployed in multiple environments
- Teams with multiple developers working on same codebase

## Solution Overview
Create a centralized resource registry that:
- **Maps logical names to actual locations** (e.g., 'logs' → '/project/local/logs/')
- **Supports nested resource hierarchies** (e.g., 'venv/main', 'venv/test')
- **Automatically creates directories** when needed
- **Provides environment-independent access** 
- **Offers both programmatic and command-line interfaces**

Core principles:
- Single source of truth for all resource locations
- Logical naming independent of physical structure
- Environment-agnostic resource resolution
- Fail-fast validation with clear error messages

## Structure & Components

### High-level Architecture
```
Application Modules
       ↓
Resource Manager (get_path)
       ↓
Path Mapping Registry
       ↓
Physical File System
```

### Key Components
1. **Path Mapping Registry** - Central dictionary of resource definitions
2. **Resource Resolver** - Function to retrieve paths by logical name
3. **Directory Manager** - Ensures required directories exist
4. **Validation Layer** - Validates resource requests and provides clear errors
5. **Helper Functions** - Convenience methods for common operations

### Pseudocode Example
```python
# Central resource mapping
RESOURCE_MAP = {
    'logs': PROJECT_ROOT / 'local' / 'logs',
    'databases': PROJECT_ROOT / 'local' / 'databases',
    'config': PROJECT_ROOT / 'config',
    'venv': {
        'main': PROJECT_ROOT / 'venv',
        'test': PROJECT_ROOT / 'test' / 'venv'
    }
}

def get_resource(resource_type, subtype=None):
    if resource_type not in RESOURCE_MAP:
        raise ValueError(f"Unknown resource: {resource_type}")
    
    resource = RESOURCE_MAP[resource_type]
    
    if isinstance(resource, dict):
        if not subtype:
            raise ValueError(f"Resource {resource_type} requires subtype")
        if subtype not in resource:
            raise ValueError(f"Unknown subtype: {subtype}")
        return resource[subtype]
    
    return resource

# Usage
log_path = get_resource('logs') / 'application.log'
test_venv = get_resource('venv', 'test')
```

## Implementation Variations

### **Simple Dictionary Approach**
- Basic key-value mapping
- Suitable for small projects
- Easy to understand and implement

### **Class-Based Registry**
- Object-oriented resource management
- Supports inheritance and composition
- Better for complex hierarchies

### **Configuration-File Driven**
- Resources defined in external config
- Runtime configuration changes
- Environment-specific resource files

### **Framework Integration**
- Built into application framework
- Automatic environment detection
- Integration with dependency injection

## Benefits & Trade-offs

**Advantages:**
- **Centralized maintenance** - Change directory structure in one place
- **Environment independence** - Same code works dev/test/prod
- **Consistent access patterns** - All modules use same approach
- **Testing isolation** - Easy to redirect resources for tests
- **Clear error messages** - Immediate feedback for invalid resource requests
- **Self-documenting** - Resource mapping serves as architecture documentation

**Disadvantages:**
- **Initial setup overhead** - Requires upfront design of resource hierarchy
- **Indirection complexity** - One more layer between code and files
- **Central dependency** - All modules depend on resource manager
- **Migration effort** - Existing hardcoded paths need to be replaced

**Alternatives:**
- **Environment variables** - Good for deployment-specific paths
- **Configuration frameworks** - More complex but more powerful
- **Relative paths** - Simple but brittle across environments
- **Framework conventions** - Works if using opinionated framework

## Real-world Examples

### **Texas RRC Lease Index Pipeline**
- **Scale**: 20+ resource types across 6 pipeline stages
- **Implementation**: Python with nested dictionaries
- **Results**: 
  - Eliminated 50+ hardcoded paths
  - Seamless deployment across 3 environments
  - Zero path-related bugs in 6 months
- **Location**: `states.tx.lease_name_index/utils/get_path.py`

### **Django Framework**
- **Pattern**: `settings.MEDIA_ROOT`, `settings.STATIC_ROOT`
- **Scale**: Global web framework standard
- **Benefits**: Consistent deployment across millions of projects

### **Node.js Projects**
- **Pattern**: `path.join(__dirname, '..', 'data')`
- **Common approach**: Path resolution relative to module location
- **Limitation**: Still requires path knowledge in each module

## Implementation Guidance

### **Step 1: Inventory Resources**
```python
# Catalog all hardcoded paths in existing code
grep -r "/path/to" src/
grep -r "\.\./data" src/
grep -r "os\.path\.join" src/
```

### **Step 2: Design Resource Hierarchy**
```python
# Group related resources logically
RESOURCES = {
    # Data storage
    'data': LOCAL_DIR / 'data',
    'databases': LOCAL_DIR / 'databases',
    'cache': LOCAL_DIR / 'cache',
    
    # Runtime files
    'logs': LOCAL_DIR / 'logs',
    'pid': LOCAL_DIR / 'pid',
    'temp': LOCAL_DIR / 'temp',
    
    # Project structure
    'config': PROJECT_ROOT / 'config',
    'scripts': PROJECT_ROOT / 'scripts',
}
```

### **Step 3: Implement Core Function**
```python
def get_path(resource_type, subtype=None):
    # Validate request
    # Resolve path
    # Create directories if needed
    # Return Path object
```

### **Step 4: Replace Hardcoded Paths**
```python
# Before
log_file = "/var/log/myapp.log"

# After  
log_file = get_path('logs') / 'myapp.log'
```

### **Common Pitfalls**
- **Over-engineering** - Start simple, add complexity only when needed
- **Inconsistent naming** - Use clear, predictable resource names
- **Missing validation** - Always validate resource requests
- **Forgetting directory creation** - Ensure directories exist before use

## Related Patterns

### **Complementary Patterns**
- **Hybrid Configuration Strategy** - Combines with centralized paths for complete configuration management
- **Factory Pattern** - Can be used to create resource-specific handlers
- **Template Method** - Standardizes resource access patterns across modules

### **Conflicting Patterns**
- **Hardcoded Paths** - Direct opposite of this pattern
- **Convention over Configuration** - May conflict if conventions are unclear

### **Effective Combinations**
- **Centralized Resource Management + Hybrid Configuration** - Complete configuration strategy
- **Centralized Resource Management + Dependency Injection** - Enterprise-level resource management

## DevForge Integration

### **Implementing Template**
- **`templates/path-management/`** - Complete Python implementation
- **Features**: 
  - Ready-to-use `get_path.py` module
  - Project initialization scripts
  - Test suite and examples
  - Integration with project templates

### **Related DevForge Patterns**
- **Hybrid Configuration Strategy** - Uses centralized paths for config file locations
- **Template-Based Project Initialization** - Creates standardized resource hierarchies

### **Future Template Opportunities**
- **Multi-language implementations** - Java, Node.js, Go versions
- **Framework-specific variants** - Django, Flask, Express.js integration
- **Cloud-native versions** - S3, Azure Blob, GCS resource management

## References

- **Gang of Four Design Patterns** - Registry pattern foundations
- **Clean Code (Martin)** - Avoiding hardcoded dependencies
- **The Pragmatic Programmer** - DRY principle and configuration management
- **12-Factor App** - Configuration and environment management best practices
- **states.tx.lease_name_index project** - Original implementation source 