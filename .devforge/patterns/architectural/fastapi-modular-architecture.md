# FastAPI Modular Architecture Pattern

**Pattern Type**: Architectural  
**Category**: Web Application Architecture  
**Maturity**: Stable  
**Last Updated**: 2025-01-10  

## Intent

Structure FastAPI applications using modular architecture that separates concerns through organized API endpoints, route handlers, template management, and database context handling to create maintainable, scalable web applications.

## Problem Statement

### Core Problems
- **Monolithic Structure**: Single large application files become unmaintainable
- **Tight Coupling**: Mixed concerns make testing and modification difficult
- **Route Scatter**: API endpoints and web routes mixed without organization
- **Template Chaos**: HTML templates scattered without logical grouping
- **Database Coupling**: Database access tightly coupled to route handlers
- **Testing Complexity**: Monolithic structure makes unit testing difficult

### Business Impact
- **Development Velocity**: Slower feature development due to code complexity
- **Maintenance Overhead**: Difficult to modify or extend existing functionality
- **Bug Introduction**: Changes in one area break unrelated functionality
- **Team Collaboration**: Multiple developers stepping on each other's work
- **Code Quality**: Degraded code quality due to mixed responsibilities

## Context

### When to Apply
- **Complex Web Applications**: Applications with multiple types of endpoints
- **Team Development**: Multiple developers working on the same application
- **Long-term Projects**: Applications requiring ongoing maintenance and extension
- **API + Web Hybrid**: Applications serving both API endpoints and web pages
- **Testing Requirements**: Applications requiring comprehensive test coverage

### When NOT to Apply
- **Simple APIs**: Single-purpose APIs with few endpoints
- **Prototype Development**: Quick prototypes or proof-of-concept applications
- **Single Developer**: Small applications maintained by one person

## Solution Overview

The FastAPI Modular Architecture Pattern organizes applications into:

1. **Core Application**: Central FastAPI app configuration and initialization
2. **API Modules**: RESTful API endpoints grouped by functionality
3. **Route Modules**: Web page routes and form handlers
4. **Template Organization**: Hierarchical template structure with components
5. **Database Context**: Centralized database connection and context management
6. **Utility Modules**: Shared functionality and helper functions

## Structure & Components

### Directory Structure

```
stage0/
├── app.py                      # Legacy monolithic (deprecated)
├── stage0.py                   # Main FastAPI application
├── run_stage0.py              # Application entry point
├── api/                       # RESTful API endpoints
│   ├── __init__.py
│   ├── corrected_records_api.py
│   ├── database_health_api.py
│   ├── export_api.py
│   ├── manual_validation_api.py
│   ├── stage4_api.py
│   └── validation_api.py
├── routes/                    # Web page routes
│   ├── __init__.py
│   ├── main_routes.py
│   ├── report_routes.py
│   ├── stage4_routes.py
│   ├── validation_routes.py
│   └── web_routes.py
├── templates/                 # HTML templates
│   ├── base_header.html
│   ├── base_footer.html
│   ├── components/
│   │   ├── stage4_status_indicator.html
│   │   └── stage4_throughput_scaling.html
│   ├── api.html
│   ├── stage4.html
│   └── validation.html
├── utils/                     # Shared utilities
│   ├── __init__.py
│   ├── database_context.py
│   ├── database.py
│   └── filters.py
└── requirements.txt
```

### Core Application Structure

```python
# stage0.py - Main FastAPI application
from fastapi import FastAPI, Request
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from fastapi.middleware.cors import CORSMiddleware

# Import route modules
from .routes import main_routes, report_routes, stage4_routes, validation_routes, web_routes
from .api import (
    corrected_records_api, database_health_api, export_api,
    manual_validation_api, stage4_api, validation_api
)

# Create FastAPI application
app = FastAPI(
    title="Texas RRC Lease Name Index",
    description="Multi-stage data processing pipeline for Texas Railroad Commission lease data",
    version="1.0.0"
)

# Configure CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Configure static files and templates
app.mount("/static", StaticFiles(directory="static"), name="static")
templates = Jinja2Templates(directory="templates")

# Include route modules
app.include_router(main_routes.router, prefix="", tags=["main"])
app.include_router(report_routes.router, prefix="/reports", tags=["reports"])
app.include_router(stage4_routes.router, prefix="/stage4", tags=["stage4"])
app.include_router(validation_routes.router, prefix="/validation", tags=["validation"])
app.include_router(web_routes.router, prefix="/web", tags=["web"])

# Include API modules
app.include_router(corrected_records_api.router, prefix="/api/corrected-records", tags=["api"])
app.include_router(database_health_api.router, prefix="/api/health", tags=["api"])
app.include_router(export_api.router, prefix="/api/export", tags=["api"])
app.include_router(manual_validation_api.router, prefix="/api/manual-validation", tags=["api"])
app.include_router(stage4_api.router, prefix="/api/stage4", tags=["api"])
app.include_router(validation_api.router, prefix="/api/validation", tags=["api"])

# Global exception handler
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    return templates.TemplateResponse(
        "error.html",
        {"request": request, "error": str(exc)},
        status_code=500
    )
```

### Entry Point Pattern

```python
# run_stage0.py - Application entry point
import uvicorn
from pathlib import Path

# Import the FastAPI app
from stage0 import app

def main():
    """Run the Stage 0 web server"""
    # Configure uvicorn
    uvicorn.run(
        "stage0:app",
        host="0.0.0.0",
        port=5580,
        reload=True,
        log_level="info"
    )

if __name__ == "__main__":
    main()
```

### API Module Pattern

```python
# api/stage4_api.py - Example API module
from fastapi import APIRouter, HTTPException, Depends
from typing import List, Optional
from pydantic import BaseModel

from ..utils.database_context import get_database_context

router = APIRouter()

# Request/Response models
class ProcessingRequest(BaseModel):
    processor_type: str
    batch_size: Optional[int] = 100
    force_refresh: Optional[bool] = False

class ProcessingResponse(BaseModel):
    status: str
    records_processed: int
    success_rate: float
    processing_time: float

# API endpoints
@router.post("/start-processing", response_model=ProcessingResponse)
async def start_processing(
    request: ProcessingRequest,
    db_context=Depends(get_database_context)
):
    """Start Stage 4 processing with specified parameters"""
    try:
        # Processing logic here
        result = await process_records(
            processor_type=request.processor_type,
            batch_size=request.batch_size,
            force_refresh=request.force_refresh,
            db_context=db_context
        )
        
        return ProcessingResponse(
            status="success",
            records_processed=result["processed"],
            success_rate=result["success_rate"],
            processing_time=result["time"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/status")
async def get_processing_status(db_context=Depends(get_database_context)):
    """Get current processing status"""
    try:
        status = await get_current_status(db_context)
        return status
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Route Module Pattern

```python
# routes/stage4_routes.py - Example route module
from fastapi import APIRouter, Request, Depends, Form
from fastapi.templating import Jinja2Templates
from fastapi.responses import HTMLResponse

from ..utils.database_context import get_database_context

router = APIRouter()
templates = Jinja2Templates(directory="templates")

@router.get("/", response_class=HTMLResponse)
async def stage4_dashboard(
    request: Request,
    db_context=Depends(get_database_context)
):
    """Render Stage 4 dashboard page"""
    try:
        # Get dashboard data
        processing_stats = await get_processing_statistics(db_context)
        processor_status = await get_processor_status(db_context)
        
        return templates.TemplateResponse(
            "stage4.html",
            {
                "request": request,
                "processing_stats": processing_stats,
                "processor_status": processor_status,
                "page_title": "Stage 4 - LLM Processing"
            }
        )
    except Exception as e:
        return templates.TemplateResponse(
            "error.html",
            {"request": request, "error": str(e)},
            status_code=500
        )

@router.post("/start-processor")
async def start_processor(
    request: Request,
    processor_type: str = Form(...),
    batch_size: int = Form(100),
    db_context=Depends(get_database_context)
):
    """Handle processor start form submission"""
    try:
        # Start processor
        result = await start_processing(
            processor_type=processor_type,
            batch_size=batch_size,
            db_context=db_context
        )
        
        # Redirect with success message
        return templates.TemplateResponse(
            "stage4.html",
            {
                "request": request,
                "success_message": f"Started {processor_type} processor",
                "processing_result": result
            }
        )
    except Exception as e:
        return templates.TemplateResponse(
            "stage4.html",
            {
                "request": request,
                "error_message": str(e)
            },
            status_code=500
        )
```

### Database Context Pattern

```python
# utils/database_context.py - Centralized database context
from contextlib import asynccontextmanager
from typing import AsyncGenerator
import sqlite3
from pathlib import Path

from utils.get_path import get_path

class DatabaseContext:
    def __init__(self):
        self.lease_db_path = get_path("lease_db")
        self.llm_db_path = get_path("llm_db")
    
    @asynccontextmanager
    async def get_lease_connection(self) -> AsyncGenerator[sqlite3.Connection, None]:
        """Get connection to main lease database"""
        conn = sqlite3.connect(self.lease_db_path)
        conn.row_factory = sqlite3.Row
        try:
            yield conn
        finally:
            conn.close()
    
    @asynccontextmanager
    async def get_llm_connection(self) -> AsyncGenerator[sqlite3.Connection, None]:
        """Get connection to LLM results database"""
        conn = sqlite3.connect(self.llm_db_path)
        conn.row_factory = sqlite3.Row
        try:
            yield conn
        finally:
            conn.close()
    
    async def execute_query(self, query: str, params: tuple = (), db_type: str = "lease"):
        """Execute query against specified database"""
        db_path = self.lease_db_path if db_type == "lease" else self.llm_db_path
        
        with sqlite3.connect(db_path) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.cursor()
            cursor.execute(query, params)
            
            if query.strip().upper().startswith("SELECT"):
                return cursor.fetchall()
            else:
                conn.commit()
                return cursor.rowcount

# Dependency injection
async def get_database_context() -> DatabaseContext:
    """FastAPI dependency for database context"""
    return DatabaseContext()
```

### Template Organization

```html
<!-- templates/base_header.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Texas RRC Lease Name Index{% endblock %}</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <link href="/static/css/style.css" rel="stylesheet">
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
        <div class="container">
            <a class="navbar-brand" href="/">RRC Lease Index</a>
            <div class="navbar-nav">
                <a class="nav-link" href="/">Dashboard</a>
                <a class="nav-link" href="/stage4">Stage 4</a>
                <a class="nav-link" href="/validation">Validation</a>
                <a class="nav-link" href="/api">API</a>
            </div>
        </div>
    </nav>
    <div class="container mt-4">
        {% block content %}{% endblock %}
    </div>

<!-- templates/base_footer.html -->
    </div>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>
    <script src="/static/js/app.js"></script>
</body>
</html>

<!-- templates/stage4.html -->
{% include 'base_header.html' %}

{% block title %}Stage 4 - LLM Processing{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-12">
        <h1>Stage 4 - LLM Processing</h1>
        
        {% if success_message %}
        <div class="alert alert-success">{{ success_message }}</div>
        {% endif %}
        
        {% if error_message %}
        <div class="alert alert-danger">{{ error_message }}</div>
        {% endif %}
        
        <!-- Include reusable components -->
        {% include 'components/stage4_status_indicator.html' %}
        {% include 'components/stage4_throughput_scaling.html' %}
        
        <!-- Processing controls -->
        <div class="card mt-4">
            <div class="card-header">
                <h3>Processing Controls</h3>
            </div>
            <div class="card-body">
                <form method="post" action="/stage4/start-processor">
                    <div class="row">
                        <div class="col-md-6">
                            <select name="processor_type" class="form-select" required>
                                <option value="">Select Processor</option>
                                <option value="enhanced_multi">Enhanced Multi</option>
                                <option value="blazing_fast">Blazing Fast</option>
                                <option value="turbo">Turbo</option>
                            </select>
                        </div>
                        <div class="col-md-4">
                            <input type="number" name="batch_size" class="form-control" 
                                   placeholder="Batch Size" value="100" min="1" max="1000">
                        </div>
                        <div class="col-md-2">
                            <button type="submit" class="btn btn-primary">Start</button>
                        </div>
                    </div>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}

{% include 'base_footer.html' %}
```

## Implementation Variations

### Basic Implementation
- Simple route separation
- Basic template organization
- Minimal database abstraction

### Enhanced Implementation
- Comprehensive API documentation
- Advanced error handling
- Database connection pooling
- Middleware integration

### Enterprise Implementation
- Authentication and authorization
- Rate limiting and caching
- Monitoring and logging integration
- Microservice architecture support

## Benefits & Trade-offs

### Benefits
- **Maintainability**: Clear separation of concerns makes code easier to maintain
- **Testability**: Modular structure enables comprehensive unit testing
- **Scalability**: Easy to add new features without affecting existing code
- **Team Collaboration**: Multiple developers can work on different modules simultaneously
- **Code Reusability**: Shared utilities and components reduce duplication

### Trade-offs
- **Initial Complexity**: More complex setup than monolithic approach
- **Learning Curve**: Developers must understand the modular structure
- **File Management**: More files to manage and navigate
- **Import Complexity**: More complex import relationships

### Performance Impact
- **Positive**: Better code organization can improve performance through optimization
- **Negative**: Slight overhead from module imports and dependency injection
- **Net Result**: Negligible performance impact with significant maintainability gains

## Real-world Examples

### Texas RRC Lease Name Index Project

#### Before Refactoring (Monolithic)
```python
# stage0/app.py - 800+ lines of mixed concerns
from fastapi import FastAPI, Request, Form
import sqlite3

app = FastAPI()

@app.get("/")
async def dashboard(request: Request):
    # Mixed database access, template rendering, business logic
    pass

@app.get("/api/stage4/status")
async def stage4_status():
    # API endpoint mixed with web routes
    pass

@app.post("/stage4/start")
async def start_stage4(processor_type: str = Form(...)):
    # Form handling mixed with API logic
    pass
```

#### After Refactoring (Modular)
```python
# stage0/stage0.py - Clean application structure
from fastapi import FastAPI
from .routes import main_routes, stage4_routes
from .api import stage4_api

app = FastAPI()
app.include_router(main_routes.router)
app.include_router(stage4_routes.router, prefix="/stage4")
app.include_router(stage4_api.router, prefix="/api/stage4")
```

#### Migration Results
- **Code Organization**: 800+ line monolith split into 12 focused modules
- **Testing**: Enabled comprehensive unit testing of individual components
- **Team Development**: Multiple developers can work on different modules
- **Maintenance**: Easy to modify specific functionality without affecting others

## Implementation Guidance

### Step 1: Analyze Existing Structure
```bash
# Identify current structure and dependencies
find . -name "*.py" -exec wc -l {} + | sort -n
grep -r "from.*import" . --include="*.py" | head -20
```

### Step 2: Plan Module Organization
```python
# Define module boundaries
API_MODULES = ["stage4_api", "validation_api", "export_api"]
ROUTE_MODULES = ["main_routes", "stage4_routes", "validation_routes"]
UTILITY_MODULES = ["database_context", "filters", "helpers"]
```

### Step 3: Create Directory Structure
```bash
mkdir -p api routes utils templates/components
touch api/__init__.py routes/__init__.py utils/__init__.py
```

### Step 4: Migrate Functionality
```python
# Extract API endpoints to separate modules
# Extract route handlers to route modules
# Create shared utilities
# Organize templates hierarchically
```

### Step 5: Update Imports and Dependencies
```python
# Update main application to use new modules
# Fix import paths
# Test all functionality
```

### Common Pitfalls
- **Circular Imports**: Avoid circular dependencies between modules
- **Over-Modularization**: Don't create too many small modules
- **Import Paths**: Ensure relative imports work correctly
- **Database Connections**: Properly manage database connections across modules

## Related Patterns

### Complementary Patterns
- **Centralized Path Management**: Used for template and static file paths
- **Mandatory Development Environment**: Provides foundation for web server deployment
- **Stage Runner Management**: Manages web server lifecycle

### Alternative Patterns
- **Microservices Architecture**: Separate services instead of modules
- **Monolithic Architecture**: Single large application file
- **Plugin Architecture**: Dynamic loading of functionality

## DevForge Integration

### Pattern Dependencies
- **Prerequisites**: FastAPI, Python 3.8+, Jinja2 templates
- **Complements**: Database management patterns
- **Enables**: API testing and documentation patterns

### Automation Support
- **Code Generation**: Automated creation of API and route modules
- **Testing Framework**: Automated testing of modular components
- **Documentation**: Automated API documentation generation

### Monitoring Integration
- **Health Checks**: Module-specific health monitoring
- **Performance Metrics**: Per-module performance tracking
- **Error Tracking**: Module-specific error reporting

## Metrics & Success Criteria

### Quantitative Metrics
- **Module Count**: Number of focused modules vs. monolithic files
- **Code Complexity**: Cyclomatic complexity per module
- **Test Coverage**: Percentage of code covered by tests
- **Development Velocity**: Time to implement new features

### Qualitative Indicators
- **Code Maintainability**: Ease of making changes to specific functionality
- **Team Productivity**: Reduced conflicts and improved collaboration
- **Bug Isolation**: Easier to identify and fix issues
- **Feature Development**: Faster implementation of new features

---

**Implementation Priority**: HIGH - Essential for maintainable web applications with multiple developers.

**Success Pattern**: Start with API/route separation, then add database context and template organization as the application grows. 