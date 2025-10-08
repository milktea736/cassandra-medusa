# Backup & Restore Targets

This document details the objects, data, and metadata that Medusa captures during backup operations and reconstructs during restore operations.

## Table of Contents
- [Overview](#overview)
- [SSTable Files](#sstable-files)
- [Schema (schema.cql)](#schema-schemacql)
- [Token Map (tokenmap.json)](#token-map-tokenmapjson)
- [Manifest (manifest.json)](#manifest-manifestjson)
- [Differential Marker](#differential-marker)
- [Server Version](#server-version)
- [Backup Index](#backup-index)
- [Storage Structure](#storage-structure)

---

## Overview

Medusa creates backups by capturing:
1. **Data files** (SSTables) from Cassandra snapshots
2. **Schema definition** (CQL commands to recreate the schema)
3. **Cluster topology** (token assignments, racks, datacenters)
4. **Backup manifest** (file inventory with checksums and sizes)
5. **Metadata markers** (differential mode, server version, timestamps)

These components are uploaded to remote storage (S3, GCS, Azure, etc.) and form a complete snapshot of the Cassandra cluster state at a point in time.

---

## SSTable Files

### What Are SSTables?
SSTables (Sorted String Tables) are immutable data files that Cassandra uses to store table data on disk. Each SSTable consists of multiple component files:
- `*-Data.db` - The actual data
- `*-Index.db` - Index for efficient data lookup
- `*-Statistics.db` - Statistical metadata
- `*-Summary.db` - Summary of the index
- `*-Filter.db` - Bloom filter
- `*-CompressionInfo.db` - Compression metadata (if enabled)
- `*-TOC.txt` - Table of contents listing all components

### How Medusa Captures SSTables

During backup, Medusa creates a Cassandra snapshot which hard-links SSTable files in a snapshot directory. The snapshot process is handled in:

- **Snapshot creation**: [`medusa/cassandra_utils.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/cassandra_utils.py) - `Cassandra.create_snapshot()` method
- **Snapshot discovery**: [`medusa/backup_node.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L301-L358) - `backup_snapshots()` function
- **File enumeration**: [`medusa/cassandra_utils.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/cassandra_utils.py#L47-L59) - `SnapshotPath.list_files()` method

The files are discovered by iterating through the snapshot directory structure:
```
<data_dir>/data/<keyspace>/<table>/snapshots/<backup_name>/
```

Each file is then uploaded to remote storage under:
```
<prefix>/<fqdn>/<backup-name>/data/<keyspace>/<table>/<sstable-file>
```

### Differential Backup Optimization

In **differential mode**, Medusa avoids re-uploading SSTables that already exist in storage:

- **Deduplication logic**: [`medusa/backup_node.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L365-L405) - `check_already_uploaded()` function
- **Storage location**: In differential mode, SSTables are stored at `<prefix>/<fqdn>/data/<keyspace>/<table>/` (shared across backups)
- **Immutability advantage**: Since SSTables are immutable, the same SSTable file can be referenced by multiple backup manifests

The deduplication compares:
1. File name
2. File size
3. MD5 checksum (if enabled)

If a match is found, the file is not re-uploaded; instead, its reference is added to the manifest.

---

## Schema (schema.cql)

### What Is the Schema File?

The `schema.cql` file contains the complete CQL Data Definition Language (DDL) statements needed to recreate the database schema, including:
- Keyspace definitions (replication strategy, replication factor)
- Table definitions (columns, types, primary keys, clustering keys)
- Index definitions
- User-defined types (UDTs)
- Materialized views
- User-defined functions and aggregates

### How Medusa Captures Schema

Schema is captured by querying Cassandra's system tables:

- **Schema extraction**: [`medusa/backup_node.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L204-L211) - `get_schema_and_tokenmap()` function
- **CQL session schema dump**: The `CqlSession.dump_schema()` method queries Cassandra metadata
- **Storage**: [`medusa/storage/node_backup.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/node_backup.py#L60-L62) - Schema is stored at `<meta_path>/schema.cql`

The schema file is **the first file uploaded** to storage during backup. Its presence indicates that a backup has started.

**Location in storage**: `<prefix>/<fqdn>/<backup-name>/meta/schema.cql`

### Use During Restore

During restore, the schema is applied to the target cluster to ensure the database structure matches the backed-up data:

- **Schema application**: [`medusa/restore_node.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_node.py) - The schema is downloaded and applied after data files are placed

---

## Token Map (tokenmap.json)

### What Is the Token Map?

The `tokenmap.json` file captures the cluster topology at backup time, recording:
- **Node FQDNs** (fully qualified domain names)
- **Token assignments** for each node (token ranges owned)
- **Datacenter placement** for each node
- **Rack placement** for each node

This information is critical for:
1. **In-place restores**: Verifying that the restore target has the same topology as the backup source
2. **Cluster-wide restores**: Mapping backup nodes to restore targets based on token ownership
3. **Consistency validation**: Ensuring data is restored to the correct nodes

### How Medusa Captures Token Map

The token map is obtained by querying the cluster's current topology:

- **Extraction**: [`medusa/backup_node.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L204-L211) - `get_schema_and_tokenmap()` function
- **CQL session query**: `CqlSession.tokenmap()` queries Cassandra's system tables for topology
- **Storage**: [`medusa/storage/node_backup.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/node_backup.py#L60) - Tokenmap is stored at `<meta_path>/tokenmap.json`

**Location in storage**: `<prefix>/<fqdn>/<backup-name>/meta/tokenmap.json`

### Token Map Structure

Example tokenmap.json content:
```json
{
  "node1.example.com": {
    "tokens": ["-9223372036854775808", "-4611686018427387904", ...],
    "datacenter": "dc1",
    "rack": "rack1"
  },
  "node2.example.com": {
    "tokens": ["0", "4611686018427387904", ...],
    "datacenter": "dc1",
    "rack": "rack2"
  }
}
```

### Use During Restore

During cluster restore, the token map is used to:

1. **Validate topology compatibility**: [`medusa/restore_cluster.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_cluster.py#L133-L150) - `RestoreJob.prepare_restore()` method
2. **Map source nodes to target nodes**: Based on token ranges
3. **Retrieve token map**: [`medusa/fetch_tokenmap.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/fetch_tokenmap.py#L21-L32) - Utility to fetch tokenmap from a backup

---

## Manifest (manifest.json)

### What Is the Manifest?

The `manifest.json` file is an inventory of all data files included in a backup. For each table, it lists:
- **Keyspace name**
- **Table (columnfamily) name**
- **Object list**: For each SSTable file:
  - `path`: Relative path in storage
  - `size`: File size in bytes
  - `MD5`: MD5 checksum for integrity validation

The manifest serves multiple purposes:
1. **Backup completeness validation**: Verifies all expected files are present
2. **Integrity checking**: Validates file checksums during restore
3. **Differential backup reference**: In differential mode, references files from previous backups
4. **Restore planning**: Determines which files need to be downloaded

### How Medusa Creates the Manifest

The manifest is built incrementally as files are uploaded:

- **Manifest construction**: [`medusa/backup_node.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L301-L362) - `backup_snapshots()` function
- **Object creation**: [`medusa/backup_node.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L408-L417) - `make_manifest_object()` function
- **Upload result tracking**: [`medusa/storage/abstract_storage.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/abstract_storage.py#L42) - Returns `ManifestObject` namedtuples with path, size, and MD5

Each storage driver's `upload_blobs()` method returns a list of `ManifestObject` instances that are aggregated into the manifest.

**Location in storage**: `<prefix>/<fqdn>/<backup-name>/meta/manifest.json`

The manifest is **the last file uploaded** to storage. Its presence indicates that the backup completed successfully.

### Manifest Structure

Example manifest.json content:
```json
[
  {
    "keyspace": "my_keyspace",
    "columnfamily": "my_table",
    "objects": [
      {
        "path": "data/my_keyspace/my_table/mc-1-big-Data.db",
        "size": 1048576,
        "MD5": "d41d8cd98f00b204e9800998ecf8427e"
      },
      {
        "path": "data/my_keyspace/my_table/mc-1-big-Index.db",
        "size": 4096,
        "MD5": "098f6bcd4621d373cade4e832627b4f6"
      }
    ]
  }
]
```

### Use During Restore

During restore, the manifest guides the download process:

- **Manifest parsing**: [`medusa/download.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/download.py#L27-L71) - `download_data()` function
- **File filtering**: [`medusa/filtering.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/filtering.py) - Filters manifest based on keyspaces/tables to restore
- **Download execution**: Files listed in the manifest are downloaded from storage

---

## Differential Marker

### What Is the Differential Marker?

The differential marker is a simple file that indicates a backup was created in differential mode. Its presence changes how Medusa interprets the storage layout.

**Location in storage**: `<prefix>/<fqdn>/<backup-name>/meta/differential`

### Storage Layout Difference

**Full backup structure**:
```
<prefix>/<fqdn>/<backup-name>/data/<keyspace>/<table>/sstable-files
<prefix>/<fqdn>/<backup-name>/meta/...
```

**Differential backup structure**:
```
<prefix>/<fqdn>/data/<keyspace>/<table>/sstable-files  # Shared across backups
<prefix>/<fqdn>/<backup-name>/meta/...                 # Per-backup metadata
<prefix>/<fqdn>/<backup-name>/meta/differential        # Marker file
```

### Implementation

- **Marker creation**: [`medusa/index.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/index.py#L99-L101) - `add_backup_start_to_index()` creates differential marker
- **Marker detection**: [`medusa/storage/node_backup.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/node_backup.py#L52-L58) - `NodeBackup.__init__()` checks for differential marker
- **Path selection**: Based on marker presence, `NodeBackup` sets `_data_path` to either backup-specific or shared location

---

## Server Version

### What Is the Server Version File?

The `server_version.json` file captures the Cassandra server type and version at backup time:
```json
{
  "server_type": "Apache Cassandra",
  "release_version": "4.0.0"
}
```

This information is used during restore to:
1. Validate compatibility between backup source and restore target
2. Handle version-specific restore logic
3. Warn about potential compatibility issues

### How Medusa Captures Server Version

- **Version extraction**: [`medusa/backup_node.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L214-L218) - `get_server_type_and_version()` function
- **CQL session query**: Queries Cassandra system tables for version information
- **Storage**: [`medusa/storage/node_backup.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/node_backup.py#L66) - Stored at `<meta_path>/server_version.json`

**Location in storage**: `<prefix>/<fqdn>/<backup-name>/meta/server_version.json`

---

## Backup Index

### What Is the Backup Index?

The backup index is a separate storage area that maintains metadata about all backups in the system. It provides:
1. **Fast backup discovery** without scanning all node directories
2. **Latest backup tracking** per node
3. **Backup completion status** tracking

### Index Structure

```
<prefix>/index/
├── backup_index/
│   └── <backup-name>/
│       ├── tokenmap_<fqdn>.json
│       ├── schema_<fqdn>.cql
│       ├── manifest_<fqdn>.json
│       ├── started_<fqdn>_<timestamp>.timestamp
│       ├── finished_<fqdn>_<timestamp>.timestamp
│       └── differential_<fqdn> (if differential)
└── latest_backup/
    └── <fqdn>/
        ├── tokenmap.json
        └── backup_name.txt
```

### Index Management

- **Start index entry**: [`medusa/index.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/index.py#L86-L102) - `add_backup_start_to_index()` function
- **Finish index entry**: [`medusa/index.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/index.py#L104-L110) - `add_backup_finish_to_index()` function
- **Latest backup marker**: [`medusa/index.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/index.py#L113-L117) - `set_latest_backup_in_index()` function
- **Index building**: [`medusa/index.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/index.py#L36-L83) - `build_indices()` can rebuild index from existing backups

The index enables efficient operations like:
- Listing all backups without scanning node directories
- Finding the latest backup for a node
- Checking backup completion status

---

## Storage Structure

### Full Backup Layout

```
<storage-root>/
├── <fqdn>/
│   └── <backup-name>/
│       ├── data/
│       │   ├── <keyspace1>/
│       │   │   └── <table1>/
│       │   │       ├── mc-1-big-Data.db
│       │   │       ├── mc-1-big-Index.db
│       │   │       └── ...
│       │   └── <keyspace2>/
│       │       └── ...
│       └── meta/
│           ├── manifest.json
│           ├── schema.cql
│           ├── tokenmap.json
│           └── server_version.json
└── index/
    ├── backup_index/<backup-name>/...
    └── latest_backup/<fqdn>/...
```

### Differential Backup Layout

```
<storage-root>/
├── <fqdn>/
│   ├── data/  # Shared across all differential backups
│   │   ├── <keyspace1>/
│   │   │   └── <table1>/
│   │   │       ├── mc-1-big-Data.db
│   │   │       ├── mc-2-big-Data.db
│   │   │       └── ...
│   │   └── ...
│   ├── <backup-name1>/
│   │   └── meta/
│   │       ├── manifest.json  # References files in ../data/
│   │       ├── schema.cql
│   │       ├── tokenmap.json
│   │       ├── server_version.json
│   │       └── differential   # Marker
│   └── <backup-name2>/
│       └── meta/
│           └── ...
└── index/...
```

### Implementation

- **Path construction**: [`medusa/storage/node_backup.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/node_backup.py#L20-L95) - `NodeBackup.__init__()` constructs paths based on mode
- **Storage abstraction**: [`medusa/storage/__init__.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/__init__.py#L64-L105) - `Storage` class provides unified interface
- **Storage drivers**: Different implementations for S3, GCS, Azure, local storage in [`medusa/storage/`](https://github.com/milktea736/cassandra-medusa/tree/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage)

---

## Summary

Medusa's backup targets form a comprehensive snapshot of a Cassandra cluster:

| Target | Purpose | Storage Path | Created When |
|--------|---------|--------------|--------------|
| **SSTables** | Actual data files | `data/<ks>/<table>/` | During snapshot upload |
| **schema.cql** | Schema definition | `meta/schema.cql` | First (backup start) |
| **tokenmap.json** | Cluster topology | `meta/tokenmap.json` | At backup start |
| **manifest.json** | File inventory | `meta/manifest.json` | Last (backup complete) |
| **differential** | Mode marker | `meta/differential` | If differential mode |
| **server_version.json** | Server info | `meta/server_version.json` | At backup start |
| **Index entries** | Fast lookup | `index/...` | Start and finish |

Together, these components enable complete backup and restore operations with integrity validation and topology awareness.
