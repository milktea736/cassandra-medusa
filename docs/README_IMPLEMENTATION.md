# Backup & Restore Implementation Analysis

This directory contains detailed technical documentation analyzing Medusa's backup and restore implementation at the source code level.

## Overview

These documents provide a comprehensive analysis of how Medusa implements backup and restore operations for Apache Cassandra, with direct links to the relevant source code in the repository. They are intended for:

- Developers working on Medusa internals
- System architects evaluating Medusa for production use
- Contributors understanding the codebase
- Anyone needing deep technical understanding of Medusa's mechanisms

## Documentation Structure

### [01_backup_restore_targets.md](01_backup_restore_targets.md)
**What gets backed up and restored**

Describes the objects, data, and metadata that define backup and restore operations:
- **SSTable Files**: Cassandra's immutable data files and their components
- **Schema (schema.cql)**: Complete DDL for recreating database structure
- **Token Map (tokenmap.json)**: Cluster topology and token assignments
- **Manifest (manifest.json)**: File inventory with checksums and sizes
- **Differential Marker**: Indicator for backup mode
- **Server Version**: Cassandra version metadata
- **Backup Index**: Fast lookup structure for backups
- **Storage Structure**: Full vs differential backup layouts

**Key topics**: Data structures, file formats, storage organization, deduplication

### [02_backup_flow.md](02_backup_flow.md)
**How backups are performed**

Detailed sequence of operations during backup:
1. CLI invocation and initialization
2. Metadata capture (schema, tokenmap, server version)
3. Snapshot creation via Cassandra
4. File discovery and enumeration
5. Deduplication checking (differential mode)
6. Parallel upload to remote storage
7. Manifest creation and indexing
8. Cleanup and completion

**Covers**: Single node backup, cluster backup orchestration, differential vs full backup logic, storage drivers, async upload architecture

**Key topics**: Workflow steps, source code references, optimization techniques, parallelism

### [03_restore_flow.md](03_restore_flow.md)
**How restores are performed**

Detailed sequence of operations during restore:
1. Backup selection and validation
2. Topology compatibility checking
3. File download from remote storage
4. Cassandra shutdown (local mode)
5. Data directory cleanup
6. File placement and permissions
7. Token configuration
8. Cassandra startup
9. Schema application
10. Post-restore verification

**Covers**: Single node restore, cluster restore orchestration, in-place vs new cluster, local vs SSTableLoader methods, validation steps

**Key topics**: Restore modes, topology mapping, data integrity, compatibility checks

## How to Use This Documentation

### For Understanding the Codebase
1. Start with **01_backup_restore_targets.md** to understand what Medusa manages
2. Read **02_backup_flow.md** to see how backups work
3. Read **03_restore_flow.md** to understand the restore process
4. Follow the hyperlinks to explore specific source code implementations

### For Development Work
- Use the source code links to navigate directly to relevant functions
- Reference the flow diagrams to understand execution sequences
- Check the operation breakdowns for detailed implementation logic
- Review the comparison tables for design decisions

### For Architecture Review
- Review the storage structure diagrams
- Understand the differential backup optimization
- Examine topology validation logic
- Evaluate parallelism and orchestration approaches

## Source Code Links

All documentation includes direct hyperlinks to the Medusa source code on GitHub, including:
- Specific line number ranges for functions and classes
- Complete file paths for comprehensive context
- Version-pinned URLs (using commit SHA) for stability

Example link format:
```
[medusa/backup_node.py:78-137](https://github.com/milktea736/cassandra-medusa/blob/<commit-sha>/medusa/backup_node.py#L78-L137)
```

## Statistics

| Document | Size | Lines | Code Links |
|----------|------|-------|------------|
| 01_backup_restore_targets.md | 18 KB | 401 | 26 |
| 02_backup_flow.md | 32 KB | 791 | 37 |
| 03_restore_flow.md | 36 KB | 1,060 | 26 |
| **Total** | **86 KB** | **2,252** | **89** |

## Related Documentation

For user-focused documentation, see:
- [Performing-backups.md](Performing-backups.md) - User guide for backup operations
- [Restoring-a-single-node.md](Restoring-a-single-node.md) - User guide for single node restore
- [Restoring-a-full-cluster.md](Restoring-a-full-cluster.md) - User guide for cluster restore
- [design.md](design.md) - High-level design overview

For operational documentation, see:
- [Configuration.md](Configuration.md) - Configuration reference
- [Usage.md](Usage.md) - Command-line usage
- [Installation.md](Installation.md) - Installation instructions

## Contributing

When updating this documentation:
1. Keep source code links current with code changes
2. Update line number ranges if functions move
3. Add new sections for new features
4. Maintain consistency in formatting and style
5. Test all hyperlinks to ensure they work

## Feedback

If you find any issues with this documentation:
- Outdated source code references
- Broken links
- Missing explanations
- Suggestions for improvement

Please open an issue or submit a pull request.

---

**Last Updated**: 2024
**Medusa Version**: Latest (master branch)
**Commit**: 9eb7969137faa961eeb71a6f610662612d2e35db
