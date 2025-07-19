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

## Sync Conflict Resolution

### **Standard Workflow for Merge Resolution**

When sync operations encounter conflicts or show 'Manual merge recommended' messages:

#### **Step 1: Push Local Improvements**
```bash
python tools/project-sync.py sync-group {group_name} --pull-local-improvements
```

This command preserves local changes and merges them with the group version.

#### **Step 2: Verify Clean Sync**
```bash
python tools/project-sync.py sync-group {group_name}
```

You should see no more 'Manual merge recommended' messages. If conflicts persist, repeat the process.

### **Why This Matters**
- Ensures local changes are preserved and merged into the group version
- Prevents sync conflicts and keeps all projects up to date
- Maintains consistency across the entire group
- Provides a standardized approach for all team members

### **Best Practices**
1. **Run merge cleanup regularly** - Don't let conflicts accumulate
2. **Verify after each sync** - Ensure clean synchronization
3. **Communicate major changes** - Let team know about significant updates
4. **Use dry-run first** - Check what will be affected before syncing

### **Troubleshooting**
- If merge cleanup fails, check file permissions and git status
- If conflicts persist, review specific files mentioned in error messages
- Coordinate with team members on conflicting changes 