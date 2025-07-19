<!--
DevForge Version Metadata:
- Version: 1.0
- Operation: system-update
- Timestamp: 2025-07-16T00:01:37.196488
- Source: author:david,assigned_by:devforge
-->

# Modular Checklist System Pattern

## Intent
Provide a flexible, hierarchical system for managing tasks and checklists across projects and groups, preventing sync conflicts while enabling collaboration.

## Context
Used in multi-project environments like GMM.States where tasks need to be shared without overwriting local changes.

## Solution
- **Local Checklists**: Store in project/checklist/ - never overwritten by sync.
- **Shared Checklists**: Store in groups/{group}/checklist/ - synced read-only; copy tasks locally.
- **Task Requests**: Use dedicated files like task-requests.md for cross-project asks, with requester notes.

## Implementation
1. Create local checklist: vim checklist/my-tasks.md
2. For shared: Add to group file, sync, then copy to local.
3. Avoid direct edits to synced files.

## Benefits
- No lost tasks on sync
- Clear ownership with requester notes
- Scalable for groups

## Examples
- Mapserver requests from Texas: Add to task-requests.md, sync, Texas copies to local. 