# Backup Flow

This document details the complete sequence of operations during a Medusa backup, from invocation to completion, with links to the relevant source code.

## Table of Contents
- [Overview](#overview)
- [Single Node Backup Flow](#single-node-backup-flow)
- [Cluster Backup Flow](#cluster-backup-flow)
- [Detailed Operation Breakdown](#detailed-operation-breakdown)
- [Storage Upload Process](#storage-upload-process)
- [Differential vs Full Backup Logic](#differential-vs-full-backup-logic)

---

## Overview

Medusa supports two backup modes:

1. **Node Backup** (`medusa backup`): Backs up a single Cassandra node
2. **Cluster Backup** (`medusa backup-cluster`): Orchestrates backup across all nodes in a cluster

Both modes follow similar core operations but differ in orchestration:
- **Node backup**: Runs locally on one node
- **Cluster backup**: Coordinates snapshot creation across all nodes, then uploads in parallel

---

## Single Node Backup Flow

### Flow Diagram

```
┌─────────────────────────────────────┐
│ CLI: medusa backup                  │
│ [medusacli.py]                      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ handle_backup()                     │
│ [backup_node.py]                    │
│ - Parse arguments                   │
│ - Generate backup name              │
│ - Create backup-in-progress marker  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ start_backup()                      │
│ [backup_node.py]                    │
│ - Initialize Storage & Cassandra    │
│ - Get schema & tokenmap             │
│ - Get server version                │
│ - Add backup start to index         │
│ - Stagger if configured             │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ do_backup()                         │
│ [backup_node.py]                    │
│ - Create/get snapshot               │
│ - Call backup_snapshots()           │
│ - Handle DSE snapshots if needed    │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ backup_snapshots()                  │
│ [backup_node.py]                    │
│ - For each keyspace/table:          │
│   - List snapshot files             │
│   - Check if already uploaded       │
│   - Upload new/changed files        │
│   - Build manifest                  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ Finalize Backup                     │
│ - Upload manifest.json              │
│ - Update index (finished)           │
│ - Set latest backup marker          │
│ - Print statistics                  │
│ - Cleanup snapshot (unless kept)    │
│ - Remove backup-in-progress marker  │
└─────────────────────────────────────┘
```

### Step-by-Step Execution

#### 1. CLI Invocation

**Entry point**: [`medusa/medusacli.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/medusacli.py) - `backup()` command handler

The CLI parses arguments including:
- `--backup-name`: Custom backup name (defaults to timestamp)
- `--stagger`: Time to wait for previous node completion
- `--mode`: `full` or `differential`
- `--enable-md5-checks`: Enable MD5 validation
- `--keep-snapshot`: Keep snapshot after backup
- `--use-existing-snapshot`: Use pre-existing snapshot

#### 2. Backup Initialization

**Function**: [`medusa/backup_node.py:78-137`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L78-L137) - `handle_backup()`

**Operations**:

1. **Generate backup name** (if not provided):
   ```python
   backup_name = backup_name_arg or start.strftime('%Y%m%d%H%M')
   ```

2. **Create backup-in-progress marker**:
   - Prevents concurrent backups on the same node
   - Location: Temporary file managed by `medusa.utils.MedusaTempFile()`
   - Deleted in `finally` block after backup completes

3. **Initialize Storage**:
   ```python
   with Storage(config=config.storage) as storage:
   ```
   - Loads appropriate storage driver (S3, GCS, Azure, local)
   - Implementation: [`medusa/storage/__init__.py:64-105`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/__init__.py#L64-L105)

4. **Create NodeBackup object**:
   ```python
   node_backup = storage.get_node_backup(
       fqdn=config.storage.fqdn,
       name=backup_name,
       differential_mode=differential_mode
   )
   ```
   - Constructs storage paths for data and metadata
   - Implementation: [`medusa/storage/node_backup.py:20-95`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/node_backup.py#L20-L95)

5. **Check for existing backup**:
   - Raises error if backup already exists (unless using existing snapshot)

#### 3. Pre-Backup Operations

**Function**: [`medusa/backup_node.py:140-201`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L140-L201) - `start_backup()`

**Operations**:

1. **Throttle backup process** (if supported):
   - Sets process to IDLE I/O priority: [`medusa/backup_node.py:37-44`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L37-L44) - `throttle_backup()`
   - Uses `ionice` and `nice` to minimize impact on Cassandra

2. **Capture schema and tokenmap**:
   - Function: [`medusa/backup_node.py:204-211`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L204-L211) - `get_schema_and_tokenmap()`
   - Opens CQL session and queries system tables
   - Retries up to 7 times with exponential backoff
   
   ```python
   with cassandra.new_session() as cql_session:
       schema = cql_session.dump_schema()
       tokenmap = cql_session.tokenmap()
   ```

3. **Capture server version**:
   - Function: [`medusa/backup_node.py:214-218`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L214-L218) - `get_server_type_and_version()`
   - Queries Cassandra for server type (Apache Cassandra, DSE, etc.) and version

4. **Store metadata in NodeBackup**:
   ```python
   node_backup.schema = schema
   node_backup.tokenmap = json.dumps(tokenmap)
   node_backup.server_version = json.dumps({'server_type': ..., 'release_version': ...})
   ```

5. **Add backup start to index**:
   - Function: [`medusa/index.py:86-102`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/index.py#L86-L102) - `add_backup_start_to_index()`
   - Uploads to `index/backup_index/<backup-name>/`:
     - `tokenmap_<fqdn>.json`
     - `schema_<fqdn>.cql`
     - `started_<fqdn>_<timestamp>.timestamp`
     - `differential_<fqdn>` (if differential mode)

6. **Stagger coordination** (if enabled):
   - Function: [`medusa/backup_node.py:47-72`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L47-L72) - `stagger()`
   - Waits for the previous node in the token ring to complete a backup
   - Prevents all nodes from backing up simultaneously
   - Polls every 60 seconds until stagger time expires

#### 4. Snapshot Creation

**Function**: [`medusa/backup_node.py:221-255`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L221-L255) - `do_backup()`

**Operations**:

1. **Create or get snapshot**:
   ```python
   if use_existing_snapshot:
       snapshot = cassandra.get_snapshot(backup_name, keep_snapshot)
   else:
       snapshot = cassandra.create_snapshot(backup_name, keep_snapshot)
   ```
   
   - **Snapshot creation**: Uses Cassandra nodetool or JMX to create snapshot
   - **Implementation**: `medusa.cassandra_utils.Cassandra.create_snapshot()`
   - Creates hard links to SSTable files in `<data_dir>/<keyspace>/<table>/snapshots/<backup_name>/`

2. **Enter snapshot context**:
   - Snapshot object is a context manager (`with snapshot:`)
   - Automatically cleans up snapshot on exit (unless `keep_snapshot=True`)
   - Ensures cleanup even if backup fails

3. **DSE-specific snapshots** (if applicable):
   - Additional snapshot for DSE-specific data (Solr cores, etc.)
   - Uses `cassandra.create_dse_snapshot(backup_name)`

#### 5. File Discovery and Upload

**Function**: [`medusa/backup_node.py:301-362`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L301-L362) - `backup_snapshots()`

**Operations**:

1. **Initialize differential mode** (if enabled):
   ```python
   if node_backup.is_differential:
       files_in_storage = storage.list_files_per_table()
   ```
   - Lists all files currently in storage for deduplication
   - Organized by keyspace → table → filename

2. **Iterate through snapshot directories**:
   ```python
   for snapshot_path in snapshot.find_dirs():
   ```
   - `snapshot.find_dirs()` yields `SnapshotPath` objects for each table
   - Implementation: [`medusa/cassandra_utils.py:47-59`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/cassandra_utils.py#L47-L59)

3. **List files in snapshot**:
   ```python
   srcs = list(snapshot_path.list_files())
   ```
   - Recursively finds all files in snapshot directory
   - Includes subdirectories (e.g., secondary indexes in `.index_name/`)

4. **Check which files need upload**:
   - Function: [`medusa/backup_node.py:365-405`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L365-L405) - `check_already_uploaded()`
   
   **Full backup mode**:
   - All files (except manifest.json and schema.cql) are uploaded
   
   **Differential mode**:
   - For each file:
     - Check if it exists in storage (by name)
     - Compare size and MD5 (if enabled)
     - If match found: add to `already_backed_up` list (no upload)
     - If not found or mismatch: add to `needs_backup` or `needs_reupload` list

5. **Upload files to storage**:
   ```python
   if len(needs_upload) > 0:
       manifest_objects += storage.storage_driver.upload_blobs(needs_upload, dst_path)
   ```
   
   - **Destination path**: `<prefix>/<fqdn>/<backup-name>/data/<keyspace>/<table>/` (full mode)
     or `<prefix>/<fqdn>/data/<keyspace>/<table>/` (differential mode)
   - **Upload implementation**: [`medusa/storage/abstract_storage.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/abstract_storage.py) - `upload_blobs()`
   - Returns `ManifestObject` list with path, size, and MD5 for each uploaded file

6. **Build manifest section**:
   ```python
   manifest.append(make_manifest_object(node_backup.fqdn, snapshot_path, manifest_objects, storage))
   ```
   
   - Function: [`medusa/backup_node.py:408-417`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L408-L417) - `make_manifest_object()`
   - Creates JSON structure:
     ```json
     {
       "keyspace": "ks_name",
       "columnfamily": "table_name",
       "objects": [
         {"path": "...", "size": 123, "MD5": "..."},
         ...
       ]
     }
     ```
   - Includes both newly uploaded files and previously backed-up files (in differential mode)

#### 6. Backup Finalization

**Function**: [`medusa/backup_node.py:251-255`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L251-L255) (continuation of `do_backup()`)

**Operations**:

1. **Upload manifest**:
   ```python
   node_backup.manifest = json.dumps(manifest)
   ```
   - The manifest property setter uploads to storage
   - Location: `<prefix>/<fqdn>/<backup-name>/meta/manifest.json`
   - This is the **last file uploaded**, indicating backup completion

2. **Update backup index**:
   - Function: [`medusa/index.py:104-110`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/index.py#L104-L110) - `add_backup_finish_to_index()`
   - Uploads to `index/backup_index/<backup-name>/`:
     - `manifest_<fqdn>.json`
     - `finished_<fqdn>_<timestamp>.timestamp`

3. **Set latest backup marker**:
   - Function: [`medusa/index.py:113-117`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/index.py#L113-L117) - `set_latest_backup_in_index()`
   - Updates `index/latest_backup/<fqdn>/` with this backup's info

4. **Print statistics**:
   - Function: [`medusa/backup_node.py:258-284`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L258-L284) - `print_backup_stats()`
   - Logs:
     - Start/end times
     - Duration (excluding stagger wait)
     - File count and total size
     - New files vs. reused files

5. **Send monitoring metrics**:
   - Function: [`medusa/backup_node.py:286-298`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L286-L298) - `update_monitoring()`
   - Sends metrics to configured monitoring system (if any)

6. **Snapshot cleanup** (on context exit):
   - If `keep_snapshot=False`: snapshot is deleted
   - Removes hard links from `snapshots/` directory

7. **Remove backup-in-progress marker**:
   - Executed in `finally` block of `handle_backup()`
   - Allows subsequent backups to proceed

---

## Cluster Backup Flow

### Overview

**Entry point**: [`medusa/backup_cluster.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_cluster.py)

A cluster backup orchestrates the backup process across all nodes in the cluster in two phases:

1. **Snapshot Phase**: Create snapshots on all nodes (in parallel)
2. **Upload Phase**: Upload snapshot data from each node (in parallel)

This two-phase approach ensures:
- All snapshots are taken at approximately the same time (cluster-wide consistency)
- Network bandwidth is utilized efficiently during upload
- Node failure during upload doesn't affect already-created snapshots

### Flow Diagram

```
┌─────────────────────────────────────┐
│ CLI: medusa backup-cluster          │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ orchestrate()                       │
│ [backup_cluster.py]                 │
│ - Create BackupJob                  │
│ - Call job.execute()                │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ BackupJob.execute()                 │
│ - Get cluster tokenmap              │
│ - Identify all nodes                │
└──────────────┬──────────────────────┘
               │
               ├─────────────────┐
               │                 │
      PHASE 1: SNAPSHOT   PHASE 2: UPLOAD
               │                 │
               ▼                 ▼
    ┌─────────────────┐  ┌─────────────────┐
    │ _create_        │  │ _upload_        │
    │ snapshots()     │  │ backup()        │
    │                 │  │                 │
    │ Parallel SSH    │  │ Parallel SSH    │
    │ to all nodes:   │  │ to all nodes:   │
    │                 │  │                 │
    │ medusa backup   │  │ medusa backup   │
    │ (snapshot only) │  │ (upload only)   │
    └─────────────────┘  └─────────────────┘
```

### Step-by-Step Execution

#### 1. Cluster Backup Initialization

**Function**: [`medusa/backup_cluster.py:31-62`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_cluster.py#L31-L62) - `orchestrate()`

**Operations**:

1. **Generate backup name** (if not provided):
   ```python
   backup_name = backup_name_arg if backup_name_arg else datetime.datetime.now().strftime('%Y%m%d%H')
   ```

2. **Create BackupJob**:
   - Class: [`medusa/backup_cluster.py:102-196`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_cluster.py#L102-L196)
   - Stores configuration for orchestration:
     - `parallel_snapshots`: Max concurrent snapshot operations
     - `parallel_uploads`: Max concurrent upload operations
     - `stagger`: Time to stagger between nodes
     - `mode`: `full` or `differential`
     - `enable_md5_checks`: Whether to validate MD5 checksums

#### 2. Get Cluster Topology

**Function**: [`medusa/backup_cluster.py:64-76`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_cluster.py#L64-L76) - `BackupJob.execute()`

**Operations**:

1. **Connect to seed node**:
   ```python
   session_provider = CqlSessionProvider([seed_target], self.config)
   ```

2. **Get tokenmap** (cluster topology):
   ```python
   with session_provider.new_session() as session:
       tokenmap = session.tokenmap()
       self.hosts = [host for host in tokenmap.keys()]
   ```
   - Identifies all nodes in the cluster
   - Gets FQDN for each node

#### 3. Phase 1: Create Snapshots (Parallel)

**Function**: [`medusa/backup_cluster.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_cluster.py) - `BackupJob._create_snapshots()`

**Operations**:

1. **Build snapshot commands** for each node:
   - Runs `medusa backup` with snapshot-only flags
   - Uses SSH (via orchestration module) to execute on each node
   - Example command:
     ```bash
     medusa backup \
       --backup-name <name> \
       --mode <full|differential> \
       [--use-sudo] \
       [other flags...]
     ```

2. **Execute in parallel**:
   - Uses `Orchestration` class to manage parallelism
   - Implementation: [`medusa/orchestration.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/orchestration.py)
   - Respects `parallel_snapshots` limit

3. **Wait for all snapshots to complete**:
   - Monitors SSH command completion
   - Logs progress and any failures

#### 4. Phase 2: Upload Backups (Parallel)

**Function**: [`medusa/backup_cluster.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_cluster.py) - `BackupJob._upload_backup()`

**Operations**:

1. **Build upload commands** for each node:
   - Runs `medusa backup` with upload flags, referencing existing snapshot
   - Example command:
     ```bash
     medusa backup \
       --backup-name <name> \
       --use-existing-snapshot \
       --mode <full|differential> \
       [other flags...]
     ```

2. **Execute in parallel**:
   - Uses `Orchestration` class
   - Respects `parallel_uploads` limit

3. **Each node performs**:
   - Snapshot discovery (from Phase 1)
   - File deduplication check (differential mode)
   - Upload to remote storage
   - Manifest creation and upload
   - Index updates

---

## Detailed Operation Breakdown

### Schema and Tokenmap Capture

**Purpose**: Capture cluster metadata required for restore validation and coordination.

**Implementation**: [`medusa/backup_node.py:204-211`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L204-L211)

**Process**:

1. **Establish CQL connection**:
   ```python
   with cassandra.new_session() as cql_session:
   ```

2. **Dump schema**:
   - Queries `system_schema` tables (Cassandra 3.0+) or `system` tables (earlier versions)
   - Generates CQL DDL statements for:
     - Keyspaces
     - Tables
     - Indexes
     - User-defined types
     - Materialized views
     - User-defined functions/aggregates

3. **Retrieve tokenmap**:
   - Queries `system.peers` and `system.local`
   - Builds topology map with:
     - Token ranges per node
     - Datacenter assignments
     - Rack assignments

4. **Retry logic**:
   - Up to 7 retry attempts
   - Exponential backoff: 10s, 20s, 40s, ..., max 120s
   - Handles transient connection failures

### Snapshot Management

**Purpose**: Create a point-in-time snapshot of Cassandra data files.

**Implementation**: `medusa.cassandra_utils.Cassandra`

**Process**:

1. **Trigger snapshot creation**:
   - **Via JMX** (default): Calls `StorageServiceMBean.takeSnapshot()`
   - **Via nodetool** (fallback): Executes `nodetool snapshot`

2. **Cassandra creates hard links**:
   - For each SSTable component in each table:
     - Creates hard link in `<data_dir>/<ks>/<table>/snapshots/<backup_name>/`
   - Hard links are instant and don't consume additional disk space

3. **Snapshot verification**:
   - Medusa verifies snapshot exists by checking snapshot directory
   - Lists all snapshot directories for discovery

4. **Cleanup on completion**:
   - If `keep_snapshot=False`: Cassandra removes hard links
   - Via `StorageServiceMBean.clearSnapshot()` or `nodetool clearsnapshot`

### File Upload Process

**Purpose**: Transfer snapshot files to remote storage with integrity validation.

**Implementation**: Storage driver-specific (S3, GCS, Azure)

**Base class**: [`medusa/storage/abstract_storage.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/abstract_storage.py)

**Process**:

1. **Async upload orchestration**:
   ```python
   async def _upload_blobs(self, srcs, dest):
       coros = [self._upload_blob(src, dest) for src in srcs]
       n = int(self.config.concurrent_transfers)
       for chunk in [coros[i:i + n] for i in range(0, len(coros), n)]:
           manifest_objects += await asyncio.gather(*chunk)
   ```
   - Creates coroutines for each file upload
   - Executes in chunks based on `concurrent_transfers` config
   - Uses asyncio for efficient I/O

2. **Individual file upload** (S3 example):
   - **Small files** (< `multi_part_upload_threshold`):
     - Single PUT operation
     - MD5 calculated locally
   - **Large files** (>= threshold):
     - Multipart upload
     - Split into chunks
     - Upload chunks in parallel
     - Combine on completion
   
3. **Calculate checksums**:
   - MD5 hash computed during upload
   - Used for integrity validation
   - Returned in `ManifestObject`

4. **Retry logic**:
   - Up to `MAX_UP_DOWN_LOAD_RETRIES` (5) attempts per file
   - Handles transient network failures

5. **Return manifest data**:
   - Each upload returns `ManifestObject(path, size, MD5)`
   - Aggregated into backup manifest

---

## Storage Upload Process

### Upload Architecture

```
┌──────────────────┐
│ backup_snapshots │
└────────┬─────────┘
         │
         ▼
┌──────────────────────┐
│ storage.storage_     │
│ driver.upload_blobs()│
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ AbstractStorage.     │
│ upload_blobs()       │
│ [abstract_storage.py]│
└────────┬─────────────┘
         │
         ▼
┌──────────────────────────┐
│ _upload_blobs() [async]  │
│ - Create coroutines      │
│ - Chunk by concurrency   │
│ - asyncio.gather()       │
└────────┬─────────────────┘
         │
         ├────────┬────────┬─────────┐
         ▼        ▼        ▼         ▼
    ┌────────┐ ┌────────┐ ...   ┌────────┐
    │_upload_│ │_upload_│       │_upload_│
    │ blob() │ │ blob() │       │ blob() │
    └────────┘ └────────┘       └────────┘
         │        │                  │
         └────────┴──────────────────┘
                  │
                  ▼
         ┌────────────────┐
         │ Storage Driver │
         │ (S3/GCS/Azure) │
         └────────────────┘
```

### Storage Driver Implementations

Different storage backends implement the abstract storage interface:

- **S3**: [`medusa/storage/s3_storage.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/s3_storage.py) & [`s3_base_storage.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/s3_base_storage.py)
- **GCS**: [`medusa/storage/google_storage.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/google_storage.py)
- **Azure**: [`medusa/storage/azure_storage.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/azure_storage.py)
- **Local**: [`medusa/storage/local_storage.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/local_storage.py)

Each implements:
- `connect()` / `disconnect()`
- `_upload_blob()`: Upload single file
- `_download_blob()`: Download single file
- `_list_blobs()`: List objects in storage
- `get_object()`: Get object metadata

---

## Differential vs Full Backup Logic

### Full Backup Mode

**Behavior**: Upload all snapshot files to backup-specific storage location.

**Storage path**: `<prefix>/<fqdn>/<backup-name>/data/<keyspace>/<table>/`

**Logic**: [`medusa/backup_node.py:380-382`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L380-L382)
```python
if node_backup.is_differential is False:
    return [src for src in srcs if src.name not in NEVER_BACKED_UP], [], []
```

**Characteristics**:
- Every backup is self-contained
- No dependencies on previous backups
- Higher storage usage
- Simpler restore (single backup contains everything)

### Differential Backup Mode

**Behavior**: Only upload new or changed SSTables; reuse existing ones.

**Storage path**: `<prefix>/<fqdn>/data/<keyspace>/<table>/` (shared)

**Logic**: [`medusa/backup_node.py:365-405`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/backup_node.py#L365-L405)

**Process**:

1. **List existing files in storage**:
   ```python
   files_in_storage = storage.list_files_per_table()
   ```
   - Organized as: `{keyspace: {table: {filename: ManifestObject}}}`

2. **For each snapshot file**:
   
   a. **Check if file exists in storage**:
   ```python
   item_in_storage = keyspace_files_in_storage.get(table, {}).get(filename, None)
   ```
   
   b. **If not found**: Mark for upload
   ```python
   if item_in_storage is None:
       needs_backup.append(src)
   ```
   
   c. **If found, compare integrity**:
   ```python
   if not storage_driver.file_matches_storage(src, item_in_storage, 
                                               multipart_threshold, 
                                               enable_md5_checks):
       needs_reupload.append(src)
   else:
       already_backed_up.append(item_in_storage)
   ```
   
   **Comparison criteria**:
   - File size must match
   - MD5 checksum must match (if `enable_md5_checks=True`)
   - Small files (<= threshold): Compare full MD5
   - Large files (> threshold): Compare multipart ETag

3. **Upload only new/changed files**:
   ```python
   needs_upload = needs_backup + needs_reupload
   if len(needs_upload) > 0:
       manifest_objects += storage.storage_driver.upload_blobs(needs_upload, dst_path)
   ```

4. **Include existing files in manifest**:
   ```python
   if len(already_backed_up) > 0 and node_backup.is_differential:
       for obj in already_backed_up:
           manifest_objects.append(obj)
   ```
   - Manifest references files without re-uploading
   - Enables restore from this backup alone

**Characteristics**:
- Lower storage usage (deduplication)
- Faster backups (fewer uploads)
- Each backup manifest is still complete (includes references)
- Requires file listing operation before backup

### Deduplication Example

**Scenario**: 
- First backup: 100 SSTables (1 GB each) → 100 GB uploaded
- Compaction occurs: 90 SSTables → 30 new SSTables, 10 new SSTables (1 GB each)
- Second differential backup: Only 10 new SSTables uploaded → 10 GB uploaded

**Storage layout**:
```
<fqdn>/data/my_ks/my_table/
  ├── sstable-1-Data.db  # From first backup
  ├── sstable-2-Data.db  # From first backup
  ├── ...
  ├── sstable-90-Data.db # From first backup
  ├── sstable-91-Data.db # From second backup (new)
  ├── ...
  └── sstable-100-Data.db # From second backup (new)

<fqdn>/backup-1/meta/
  └── manifest.json  # References sstable-1 through sstable-90

<fqdn>/backup-2/meta/
  └── manifest.json  # References sstable-1 through sstable-100
```

---

## Summary

The backup flow involves:

1. **Initialization**: CLI invocation, argument parsing, storage setup
2. **Metadata capture**: Schema, tokenmap, server version
3. **Snapshot creation**: Cassandra creates hard-linked snapshot
4. **File discovery**: Enumerate all files in snapshot
5. **Deduplication** (differential mode): Check which files already exist
6. **Upload**: Transfer new/changed files to remote storage
7. **Manifest creation**: Build inventory with checksums
8. **Indexing**: Update backup index for fast lookup
9. **Cleanup**: Remove snapshot, update monitoring, mark complete

**Key design principles**:
- **Immutability awareness**: Exploits immutable SSTables for deduplication
- **Parallel execution**: Async uploads, cluster-wide coordination
- **Resilience**: Retries, exponential backoff, cleanup on failure
- **Observability**: Comprehensive logging, metrics, progress tracking
- **Flexibility**: Full vs. differential, stagger, existing snapshots
