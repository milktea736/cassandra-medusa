# Restore Flow

This document details the complete sequence of operations during a Medusa restore, from backup selection to post-restore verification, with links to the relevant source code.

## Table of Contents
- [Overview](#overview)
- [Single Node Restore Flow](#single-node-restore-flow)
- [Cluster Restore Flow](#cluster-restore-flow)
- [Detailed Operation Breakdown](#detailed-operation-breakdown)
- [Restore Modes](#restore-modes)
- [Data Placement and Validation](#data-placement-and-validation)

---

## Overview

Medusa supports two restore approaches:

1. **Node Restore** (`medusa restore-node`): Restores data to a single Cassandra node
2. **Cluster Restore** (`medusa restore-cluster`): Orchestrates restore across all nodes in a cluster

Both can operate in two modes:
- **In-place restore**: Restore to the original cluster (same topology)
- **New cluster restore**: Restore to different hardware (topology mapping required)

Additionally, two restore methods are available:
- **Local restore**: Direct file placement (requires Cassandra shutdown)
- **SSTableLoader**: Online restore using Cassandra's bulk loader (no downtime)

---

## Single Node Restore Flow

### Flow Diagram

```
┌─────────────────────────────────────┐
│ CLI: medusa restore-node            │
│ [medusacli.py]                      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ restore_node()                      │
│ [restore_node.py]                   │
│ - Validate arguments                │
│ - Initialize Storage                │
│ - Capture release version           │
└──────────────┬──────────────────────┘
               │
               ├────────────────┬──────────────────┐
               │                │                  │
        LOCAL RESTORE    SSTABLELOADER      VERIFY
               │                │                  │
               ▼                ▼                  ▼
    ┌──────────────────┐ ┌──────────────┐  ┌──────────┐
    │restore_node_     │ │restore_node_ │  │verify_   │
    │locally()         │ │sstableloader │  │restore() │
    │                  │ │()            │  │          │
    │- Get backup      │ │- Get backup  │  │- Run     │
    │- Download data   │ │- Download    │  │  queries │
    │- Stop Cassandra  │ │- Load with   │  │- Validate│
    │- Clean paths     │ │  sstableloader│ │  results │
    │- Move data       │ │              │  └──────────┘
    │- Start Cassandra │ │              │
    └──────────────────┘ └──────────────┘
```

### Step-by-Step Execution

#### 1. CLI Invocation and Validation

**Entry point**: [`medusa/restore_node.py:40-61`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_node.py#L40-L61) - `restore_node()`

**Arguments**:
- `backup-name`: Name of backup to restore
- `--in-place`: Restore to same cluster (default)
- `--keep-auth`: Keep existing system_auth data (not compatible with in-place)
- `--seed <host>`: Seed node for coordination
- `--verify`: Run post-restore verification
- `--keyspaces`: Filter restore to specific keyspaces
- `--tables`: Filter restore to specific tables
- `--use-sstableloader`: Use SSTableLoader instead of direct file placement

**Validation**:
```python
if in_place and keep_auth:
    logging.error('Cannot keep system_auth when restoring in-place. It would be overwritten')
    sys.exit(1)
```
- In-place restore overwrites all data, including system_auth
- To preserve authentication, use new cluster restore

#### 2. Storage Initialization and Backup Discovery

**Function**: [`medusa/restore_node.py:63-75`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_node.py#L63-L75) - `restore_node_locally()` (local mode)

**Operations**:

1. **Initialize Storage**:
   ```python
   with Storage(config=config.storage) as storage:
   ```

2. **Check for differential mode**:
   ```python
   differential_blob = storage.storage_driver.get_blob(
       os.path.join(config.storage.fqdn, backup_name, 'meta', 'differential'))
   ```
   - Looks for differential marker file
   - Determines storage path structure

3. **Get NodeBackup object**:
   ```python
   node_backup = storage.get_node_backup(
       fqdn=config.storage.fqdn,
       name=backup_name,
       differential_mode=True if differential_blob is not None else False
   )
   ```
   - Constructs paths based on mode
   - Implementation: [`medusa/storage/node_backup.py:20-95`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/node_backup.py#L20-L95)

4. **Verify backup exists**:
   ```python
   if not node_backup.exists():
       logging.error('No such backup')
       sys.exit(1)
   ```

#### 3. Table Filtering

**Function**: Part of [`medusa/restore_node.py:63-83`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_node.py#L63-L83)

**Operations**:

1. **Parse manifest**:
   ```python
   fqtns_to_restore, ignored_fqtns = filter_fqtns(keyspaces, tables, node_backup.manifest)
   ```
   - Implementation: [`medusa/filtering.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/filtering.py)
   - Parses manifest JSON
   - Applies keyspace and table filters
   - Returns list of fully-qualified table names (FQTNs) to restore

2. **Log filtering results**:
   ```python
   for fqtns in ignored_fqtns:
       logging.info('Skipping restore of {}'.format(fqtns))
   ```

3. **Validate restore set**:
   ```python
   if len(fqtns_to_restore) == 0:
       logging.error('There is nothing to restore')
       sys.exit(0)
   ```

#### 4. Data Download

**Function**: [`medusa/download.py:27-71`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/download.py#L27-L71) - `download_data()`

**Operations**:

1. **Parse manifest**:
   ```python
   manifest = json.loads(backup.manifest)
   ```
   - Manifest contains list of tables with object details

2. **Check available disk space**:
   ```python
   _check_available_space(manifest, destination)
   ```
   - Calculates total download size from manifest
   - Compares with available disk space
   - Fails early if insufficient space

3. **Create temporary download directory**:
   ```python
   download_dir = temp_dir / 'medusa-restore-{}'.format(uuid.uuid4())
   ```
   - Unique directory for this restore operation
   - Structure: `<download_dir>/<keyspace>/<table>/`

4. **Download data files**:
   ```python
   for section in manifest:
       fqtn = "{}.{}".format(section['keyspace'], section['columnfamily'])
       dst = destination / section['keyspace'] / section['columnfamily']
       srcs = ['{}{}'.format(storage.storage_driver.get_path_prefix(backup.data_path), 
                             obj['path'])
               for obj in section['objects']]
       
       if len(srcs) > 0 and (fqtn in fqtns_to_restore):
           dst.mkdir(parents=True)
           storage.storage_driver.download_blobs(srcs, dst)
   ```
   
   **Download process**:
   - For each table in manifest:
     - Extract object paths from manifest
     - Create destination directory (including subdirs for indexes)
     - Call `storage_driver.download_blobs()`
   
   **Storage driver download**:
   - Implementation: Storage-specific (S3, GCS, Azure)
   - Base class: [`medusa/storage/abstract_storage.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/abstract_storage.py)
   - Parallel downloads based on `concurrent_transfers` config
   - Retry logic with exponential backoff

5. **Download metadata files**:
   ```python
   storage.storage_driver.download_blobs(
       srcs=[backup.manifest_path, backup.schema_path, backup.tokenmap_path],
       dest=destination
   )
   ```
   - Downloads manifest.json, schema.cql, tokenmap.json
   - Used for validation and schema application

#### 5. Cassandra Shutdown (Local Restore Only)

**Function**: [`medusa/restore_node.py:92-99`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_node.py#L92-L99)

**Operations**:

1. **Stop Cassandra**:
   ```python
   cassandra.shutdown()
   ```
   - Implementation: `medusa.cassandra_utils.Cassandra.shutdown()`
   - Methods (in order of preference):
     - Stop via systemd (`systemctl stop cassandra`)
     - Stop via init.d (`service cassandra stop`)
     - Stop via custom command (config: `stop_cmd`)

2. **Wait for shutdown**:
   ```python
   wait_for_node_to_go_down(config, cassandra.hostname)
   ```
   - Polls Cassandra port until connection refused
   - Timeout after configured wait time
   - Ensures Cassandra is fully stopped before file manipulation

**Note**: SSTableLoader mode skips this step as Cassandra remains running.

#### 6. Data Directory Cleanup

**Function**: [`medusa/restore_node.py:97-134`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_node.py#L97-L134) (in `restore_node_locally()`)

**Operations**:

1. **Clean commit logs**:
   ```python
   clean_path(cassandra.commit_logs_path, use_sudo, keep_folder=True)
   ```
   - Removes all files in commit log directory
   - Prevents replay of old commits
   - Keeps directory structure

2. **Clean saved caches**:
   ```python
   clean_path(cassandra.saved_caches_path, use_sudo, keep_folder=True)
   ```
   - Removes cached data (key cache, row cache, counter cache)
   - Prevents stale cache conflicts

3. **Clean data directory (table by table)**:
   ```python
   for section in manifest:
       fqtn = "{}.{}".format(section['keyspace'], section['columnfamily'])
       if fqtn in fqtns_to_restore:
           clean_path(data_dir, use_sudo, keep_folder=False)
   ```
   - Removes existing SSTable files for tables being restored
   - Selective: only cleans tables being restored

**Use sudo option**:
- If `config.storage.use_sudo_for_restore=True`, uses sudo for file operations
- Required if Cassandra runs as different user (e.g., `cassandra` user)

#### 7. Data Placement

**Function**: [`medusa/restore_node.py:102-134`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_node.py#L102-L134) (in `restore_node_locally()`)

**Operations**:

1. **Move data files to Cassandra data directory**:
   ```python
   for section in manifest:
       fqtn = "{}.{}".format(section['keyspace'], section['columnfamily'])
       if fqtn not in fqtns_to_restore:
           continue
       maybe_restore_section(section, download_dir, cassandra.root, 
                           in_place, keep_auth, use_sudo)
   ```

2. **Section restore logic** (per table):
   - Implementation: `maybe_restore_section()` in `restore_node.py`
   - Determines correct data directory:
     - Looks up table UUID from Cassandra schema
     - Constructs path: `<data_dir>/<keyspace>/<table>-<uuid>/`
   
3. **Handle system_auth** (if `keep_auth=False`):
   ```python
   if section['keyspace'] == 'system_auth' and keep_auth:
       logging.info('Skipping system_auth restore (keep_auth=True)')
       continue
   ```
   - By default, system_auth is restored (overwrites authentication)
   - If `keep_auth=True`, skip (preserve existing auth)

4. **Move files**:
   - Uses `shutil.move()` or `sudo mv` depending on config
   - Moves all SSTable components
   - Preserves secondary index subdirectories

5. **Set ownership**:
   - If `use_sudo=True`, sets ownership to Cassandra user
   - Ensures Cassandra can read/write files

#### 8. Token Configuration

**Function**: [`medusa/restore_node.py:135-145`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_node.py#L135-L145)

**Operations**:

1. **Read tokenmap**:
   ```python
   token_map_file = download_dir / 'tokenmap.json'
   with open(str(token_map_file), 'r') as f:
       tokens = get_node_tokens(node_fqdn, f)
   ```
   - Extracts tokens for this node from tokenmap

2. **Update Cassandra configuration**:
   - Modifies `cassandra.yaml`
   - Sets `initial_token` to match backed-up tokens
   - Implementation: Edits YAML file directly

**Importance**:
- Ensures restored node has same token ranges as backup
- Critical for data consistency in multi-node restores
- Enables correct data routing

#### 9. Cassandra Startup

**Function**: [`medusa/restore_node.py:147-174`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_node.py#L147-L174)

**Operations**:

1. **Wait for seed nodes** (if configured):
   ```python
   if seeds is not None and len(seeds) > 0:
       wait_for_seeds(seeds)
   ```
   - Ensures seed nodes are online before starting
   - Prevents joining issues

2. **Start Cassandra**:
   ```python
   cassandra.start()
   ```
   - Methods (in order of preference):
     - Start via systemd (`systemctl start cassandra`)
     - Start via init.d (`service cassandra start`)
     - Start via custom command (config: `start_cmd`)

3. **Wait for node to be ready**:
   - Polls CQL port until connection succeeds
   - Waits for node to join cluster (if multi-node)
   - Timeout after configured wait time

4. **Apply schema** (if needed):
   - Loads schema.cql using CQL session
   - Creates keyspaces, tables, indexes, etc.
   - May skip if schema already exists

#### 10. Post-Restore Verification (Optional)

**Function**: [`medusa/verify_restore.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/verify_restore.py) - `verify_restore()`

**Operations** (if `--verify` flag):

1. **Connect to restored node**:
   ```python
   with storage.get_session() as session:
   ```

2. **Run verification queries**:
   - Executes sample queries against restored data
   - Compares results with expected values (from backup metadata)
   - Validates data integrity

3. **Report results**:
   - Logs success or failures
   - Returns non-zero exit code on failure

---

## Cluster Restore Flow

### Overview

**Entry point**: [`medusa/restore_cluster.py:37-95`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_cluster.py#L37-L95) - `orchestrate()`

A cluster restore coordinates the restoration process across all nodes in the cluster. It supports two scenarios:

1. **In-place cluster restore** (using seed target):
   - Restore to the same cluster
   - Uses current cluster topology
   - Maps backup nodes to current nodes based on tokens

2. **New cluster restore** (using host list):
   - Restore to different hardware
   - Requires manual node mapping
   - Creates new cluster with backed-up topology

### Flow Diagram

```
┌─────────────────────────────────────┐
│ CLI: medusa restore-cluster         │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ orchestrate()                       │
│ [restore_cluster.py]                │
│ - Validate arguments                │
│ - Initialize Storage                │
│ - Get ClusterBackup                 │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ RestoreJob.__init__()               │
│ - Store configuration               │
│ - Initialize orchestration          │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│ RestoreJob.execute()                │
│ - prepare_restore()                 │
│ - _restore_data()                   │
└──────────────┬──────────────────────┘
               │
               ├────────────────┬──────────────────┐
               │                │                  │
        PREPARE          RESTORE DATA         VERIFY
               │                │                  │
               ▼                ▼                  ▼
    ┌──────────────────┐ ┌──────────────┐  ┌──────────┐
    │- Validate backup │ │- Create work │  │- Run     │
    │- Build topology  │ │  directories │  │  verify  │
    │  mapping         │ │- SSH to each │  │  on all  │
    │- Capture version │ │  node        │  │  nodes   │
    │                  │ │- Execute     │  │          │
    │                  │ │  restore-node│  │          │
    └──────────────────┘ └──────────────┘  └──────────┘
```

### Step-by-Step Execution

#### 1. Cluster Restore Initialization

**Function**: [`medusa/restore_cluster.py:37-95`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_cluster.py#L37-L95) - `orchestrate()`

**Arguments**:
- `backup-name`: Name of backup to restore
- `--seed-target <host>`: Seed node of existing cluster (in-place restore)
- `--host-list <file>`: File with list of target hosts (new cluster)
- `--keep-auth`: Keep existing authentication
- `--bypass-checks`: Skip validation checks
- `--verify`: Run post-restore verification
- `--keyspaces` / `--tables`: Filter restore scope
- `--use-sstableloader`: Use SSTableLoader method
- `--parallel-restores`: Number of parallel restore operations

**Validation**:

1. **Determine seed target** (if not provided):
   ```python
   if seed_target is None and host_list is None:
       seed_target = hostname_resolver.resolve_fqdn(socket.gethostbyname(socket.getfqdn()))
   ```
   - Defaults to local node
   - Issues warning

2. **Check for conflicting arguments**:
   ```python
   if seed_target is not None and host_list is not None:
       raise RuntimeError('You must either provide a seed target or a list of host, not both')
   ```

3. **Validate temp directory**:
   ```python
   if not temp_dir.is_dir():
       raise RuntimeError('{} is not a directory'.format(temp_dir))
   ```

#### 2. Backup Discovery and Validation

**Function**: [`medusa/restore_cluster.py:60-67`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_cluster.py#L60-L67)

**Operations**:

1. **Get ClusterBackup object**:
   ```python
   cluster_backup = storage.get_cluster_backup(backup_name)
   ```
   - Implementation: [`medusa/storage/cluster_backup.py:19-97`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/cluster_backup.py#L19-L97)
   - Aggregates all NodeBackup objects for this backup
   - Provides cluster-wide view

2. **Check backup exists**:
   ```python
   if not cluster_backup:
       raise RuntimeError('No such backup --> {}'.format(backup_name))
   ```

#### 3. Create RestoreJob

**Class**: [`medusa/restore_cluster.py:102-132`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_cluster.py#L102-L132) - `RestoreJob`

**Initialization**:
```python
restore = RestoreJob(cluster_backup, config, temp_dir, host_list, seed_target, 
                     keep_auth, verify, parallel_restores, keyspaces, tables, 
                     bypass_checks, use_sstableloader, version_target, ignore_racks)
```

**Stored state**:
- `cluster_backup`: ClusterBackup object with all node backups
- `host_map`: Mapping of target hosts to source backups (populated later)
- `ringmap`: Cluster topology mapping (populated later)
- `orchestration`: Parallel execution controller
- `work_dir`: Temporary directory for this restore job

#### 4. Restore Preparation

**Function**: [`medusa/restore_cluster.py:133-152`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_cluster.py#L133-L152) - `RestoreJob.prepare_restore()`

**Operations**:

1. **Validate backup completeness**:
   ```python
   if not self.cluster_backup.is_complete():
       raise RuntimeError('Backup is not complete')
   ```
   - Checks that all nodes in tokenmap have finished backups
   - Implementation: [`medusa/storage/cluster_backup.py:68-82`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/storage/cluster_backup.py#L68-L82)

2. **CASE 1: In-place restore (using seed target)**:
   ```python
   if self.seed_target is not None:
       self.session_provider = CqlSessionProvider([self.seed_target], self.config)
       with self.session_provider.new_session() as session:
           self._populate_ringmap(self.cluster_backup.tokenmap, session.tokenmap())
           self._capture_release_version(session)
   ```
   
   **Ringmap population**:
   - Compares backup tokenmap with current cluster tokenmap
   - Maps backup nodes to current nodes based on token ranges
   - Validates topology compatibility
   - Implementation: `RestoreJob._populate_ringmap()` in `restore_cluster.py`
   
   **Compatibility check**:
   - Ensures token ranges match
   - Validates datacenter and rack placement (unless `ignore_racks=True`)
   - Fails if topology mismatch detected

3. **CASE 2: New cluster restore (using host list)**:
   ```python
   if self.host_list is not None:
       self.in_place = False
       self._populate_hostmap()
       self._capture_release_version(session=None)
   ```
   
   **Hostmap population**:
   - Reads host list file
   - Maps each target host to a backup node
   - Order determines mapping (first target → first backup node, etc.)
   - Implementation: `RestoreJob._populate_hostmap()` in `restore_cluster.py`

#### 5. User Confirmation

**Function**: [`medusa/restore_cluster.py:154-178`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_cluster.py#L154-L178) (in `RestoreJob._restore_data()`)

**Operations**:

1. **Display restore plan**:
   ```python
   for target, sources in self.host_map.items():
       logging.info('About to restore on {} using {} as backup source'.format(target, sources))
   ```

2. **Warn about data deletion**:
   ```python
   logging.info("This will delete all data on the target nodes and replace it with backup '{}'."
                .format(self.cluster_backup.name))
   ```

3. **Prompt for confirmation** (unless `bypass_checks=True`):
   ```python
   proceed = input('Are you sure you want to proceed? (Y/n)')
   if proceed == 'n':
       raise RuntimeError('Restore manually cancelled')
   ```

#### 6. Parallel Node Restore

**Function**: [`medusa/restore_cluster.py:180-220`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_cluster.py#L180-L220) (in `RestoreJob._restore_data()`)

**Operations**:

1. **Create work directories on each target**:
   - SSH to each target node
   - Create unique work directory: `/tmp/medusa-restore-<job-id>`

2. **Build restore command** for each node:
   ```python
   restore_command = self._build_restore_cmd(target, backup_source, work_dir)
   ```
   
   **Command structure**:
   ```bash
   medusa restore-node \
     --backup-name <backup_name> \
     [--in-place | --seed <target>] \
     [--keep-auth] \
     [--use-sstableloader] \
     [--keyspaces k1 k2 ...] \
     [--tables t1 t2 ...] \
     --temp-dir <work_dir>
   ```

3. **Execute restore commands in parallel**:
   ```python
   self.orchestration.pssh_run(
       host_list=list(self.host_map.keys()),
       command_list=restore_commands,
       hosts_variables=host_variables
   )
   ```
   
   - Uses parallel SSH (pssh) to run commands
   - Respects `parallel_restores` limit
   - Monitors progress and logs output
   - Implementation: [`medusa/orchestration.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/orchestration.py)

4. **Wait for completion**:
   - Monitors SSH command exit codes
   - Collects stdout/stderr from each node
   - Fails if any node restore fails

#### 7. Post-Restore Verification (Optional)

**Function**: Part of `RestoreJob.execute()`

**Operations** (if `verify=True`):

1. **Run verification on all nodes**:
   ```python
   if self.verify:
       verify_restore(list(self.host_map.keys()), self.config)
   ```

2. **Aggregate results**:
   - Collects verification results from each node
   - Logs any discrepancies
   - Returns non-zero exit code if any verification fails

---

## Detailed Operation Breakdown

### Manifest Selection and Parsing

**Purpose**: Determine which files need to be downloaded and where to place them.

**Implementation**: Part of download and restore process

**Process**:

1. **Load manifest**:
   ```python
   manifest = json.loads(node_backup.manifest)
   ```
   - Retrieves manifest.json from backup
   - Parses JSON structure

2. **Iterate sections** (one per table):
   ```json
   {
     "keyspace": "ks_name",
     "columnfamily": "table_name",
     "objects": [
       {"path": "data/ks/table/file.db", "size": 123, "MD5": "..."}
     ]
   }
   ```

3. **Apply filters**:
   - Check if `keyspace` matches `--keyspaces` filter
   - Check if `table` matches `--tables` filter
   - Skip if not in restore set

4. **Build download list**:
   - Extract object paths
   - Prepend storage prefix
   - Queue for download

### Metadata and Topology Validation

**Purpose**: Ensure restore target is compatible with backup source.

**Implementation**: [`medusa/restore_cluster.py:133-152`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_cluster.py#L133-L152) - `RestoreJob.prepare_restore()`

**Validation steps**:

1. **Backup completeness**:
   - Check all nodes have `finished` timestamp
   - Verify no missing nodes in backup

2. **Topology compatibility** (in-place restore):
   
   **Token range matching**:
   ```python
   def _populate_ringmap(self, backup_tokenmap, target_tokenmap):
       for node in backup_tokenmap:
           # Find target node with matching token ranges
           matching_target = find_matching_tokens(node, target_tokenmap)
           if not matching_target:
               raise RuntimeError(f'No matching target for backup node {node}')
           self.ringmap[node] = matching_target
   ```
   
   **Rack/DC validation**:
   - Compare datacenter names
   - Compare rack names
   - Fail if mismatch (unless `ignore_racks=True`)

3. **Version compatibility**:
   ```python
   def _capture_release_version(self, session):
       if session:
           target_version = session.get_release_version()
       else:
           target_version = self._version_target
       
       backup_version = self.cluster_backup.schema_version
       
       if not compatible(target_version, backup_version):
           logging.warning('Version mismatch: backup={}, target={}'
                         .format(backup_version, target_version))
   ```
   
   **Compatibility rules**:
   - Major version must match (e.g., 3.x → 3.x)
   - Minor version can differ (with warnings)
   - DSE vs. Apache Cassandra detected and validated

### File Download and Placement

**Download process**: Covered in [Data Download](#4-data-download) section above.

**Placement process**:

1. **Determine target directories**:
   
   **Get table UUID**:
   - Query `system_schema.tables` for table UUID
   - Table UUID is stable identifier (survives renames)
   
   **Construct path**:
   ```
   <cassandra_data_dir>/<keyspace>/<table>-<uuid>/
   ```

2. **Handle secondary indexes**:
   - Secondary indexes stored in subdirectories: `.index_name/`
   - Must preserve subdirectory structure
   - Move entire tree, including subdirs

3. **System tables handling**:
   
   **system_auth**:
   - Contains users, roles, permissions
   - If `keep_auth=True`: Skip restore (preserve existing)
   - If `keep_auth=False`: Restore (overwrite existing auth)
   
   **Other system keyspaces**:
   - Generally restored as part of backup
   - Some tables may be skipped (e.g., `system.peers`)

4. **Set file permissions**:
   ```bash
   chown cassandra:cassandra <files>
   chmod 644 <files>
   ```
   - Ensures Cassandra can read/write
   - Uses `sudo` if configured

### Post-Restore Verification

**Purpose**: Validate that restored data is correct and accessible.

**Implementation**: [`medusa/verify_restore.py`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/verify_restore.py)

**Process**:

1. **Connect to Cassandra**:
   ```python
   with CqlSession(config) as session:
   ```

2. **Run verification queries**:
   - Stored in backup metadata (if available)
   - Or default queries:
     - `SELECT COUNT(*) FROM <table>`
     - Sample data queries
   
3. **Compare results**:
   - Compare with expected values from backup time
   - Log any discrepancies

4. **Check schema**:
   - Verify all expected tables exist
   - Validate table definitions match backup

5. **Report results**:
   - Success: Log confirmation
   - Failure: Log specific errors, return non-zero exit code

---

## Restore Modes

### In-Place Restore

**Characteristics**:
- Restore to the same cluster (same topology)
- Overwrites all existing data
- Preserves cluster configuration (tokens, racks, DCs)

**Use cases**:
- Disaster recovery
- Rollback to previous state
- Testing on production cluster clone

**Requirements**:
- Current cluster topology must match backup topology
- Token ranges must be identical
- Node count must match

**Process**:
1. Validate topology match
2. Stop Cassandra on all nodes
3. Clear data directories
4. Download and place backup data
5. Start Cassandra on all nodes

### New Cluster Restore

**Characteristics**:
- Restore to different hardware
- Creates new cluster with backed-up topology
- Requires node mapping configuration

**Use cases**:
- Migration to new infrastructure
- Multi-region restore
- Capacity changes (different node specs)

**Requirements**:
- Target nodes must be provisioned
- Host list must map all backup nodes
- Network connectivity configured

**Process**:
1. Provision new nodes
2. Create host list mapping
3. Configure Cassandra with backup tokens
4. Download and place backup data
5. Start Cassandra cluster
6. Run repairs (if needed)

### Local Restore (Direct File Placement)

**Method**: Direct file placement in Cassandra data directory

**Advantages**:
- Fastest restore method
- No streaming overhead
- Complete control over data

**Disadvantages**:
- Requires Cassandra downtime
- Must have file system access
- Overwrites existing data

**When to use**:
- Full cluster restore
- Downtime acceptable
- Maximum speed required

### SSTableLoader Restore

**Method**: Uses Cassandra's `sstableloader` tool to stream data

**Advantages**:
- No Cassandra downtime required
- Works with running cluster
- Can restore to different topology
- Cassandra handles data placement

**Disadvantages**:
- Slower (network streaming overhead)
- Requires available disk space for temp files
- May cause compaction load

**When to use**:
- Cannot stop Cassandra
- Partial restore (specific tables)
- Restore to different topology

**Process**:

1. **Download data to temp location**:
   - Same as local restore download phase

2. **For each table**:
   ```bash
   sstableloader -d <seed_node> <table_directory>
   ```
   - Loads SSTables into running cluster
   - Cassandra distributes data based on tokens
   - Triggers compaction

3. **Cleanup temp files**:
   - Remove downloaded SSTables after loading

**Implementation**: [`medusa/restore_node.py:183-240`](https://github.com/milktea736/cassandra-medusa/blob/9eb7969137faa961eeb71a6f610662612d2e35db/medusa/restore_node.py#L183-L240) - `restore_node_sstableloader()`

---

## Data Placement and Validation

### SSTable Discovery and Identification

**Cassandra's data organization**:
```
<data_dir>/
├── <keyspace>/
│   ├── <table>-<uuid>/
│   │   ├── mc-1-big-Data.db
│   │   ├── mc-1-big-Index.db
│   │   ├── mc-1-big-Summary.db
│   │   ├── mc-1-big-Statistics.db
│   │   ├── mc-1-big-Filter.db
│   │   ├── mc-1-big-TOC.txt
│   │   └── .index_name/  # Secondary index
│   │       ├── mc-1-big-Data.db
│   │       └── ...
│   └── <another_table>-<uuid>/
└── ...
```

**Table UUID resolution**:
1. Query `system_schema.tables` (or `system.schema_columnfamilies` for older versions)
2. Match `keyspace` and `table` name
3. Extract UUID
4. Construct path with UUID

### Data Integrity Validation

**During download**:
1. **Size validation**:
   - Compare downloaded file size with manifest
   - Fail if mismatch

2. **Checksum validation** (if enabled):
   - Calculate MD5 of downloaded file
   - Compare with manifest MD5
   - Fail if mismatch

**After placement**:
1. **SSTable validation**:
   - Cassandra validates SSTables on startup
   - Checks TOC files, component consistency
   - Logs warnings for corrupt SSTables

2. **Schema validation**:
   - Apply schema from schema.cql
   - Verify all tables exist
   - Check data types match

### Post-Restore Operations

**Repair** (recommended but not automatic):
```bash
nodetool repair -pr
```
- Ensures data consistency across replicas
- Fills in any missing data
- Should be run after multi-node restore

**Cleanup** (if topology changed):
```bash
nodetool cleanup
```
- Removes data not owned by this node
- Required if token ranges changed

**Compaction** (optional):
```bash
nodetool compact
```
- Merges SSTables
- Improves read performance
- May free disk space

---

## Summary

The restore flow involves:

1. **Initialization**: CLI invocation, argument validation, storage setup
2. **Backup discovery**: Find and validate backup exists and is complete
3. **Topology validation**: Ensure compatibility between backup and target
4. **Download**: Retrieve data files and metadata from storage
5. **Preparation**: Stop Cassandra, clean directories (local mode)
6. **Placement**: Move files to Cassandra data directories or use sstableloader
7. **Configuration**: Update tokens, start Cassandra
8. **Verification**: Validate restored data integrity and accessibility

**Key design principles**:
- **Topology awareness**: Validates and respects cluster topology
- **Flexibility**: Supports in-place, new cluster, and online restores
- **Safety**: Confirmation prompts, validation checks
- **Parallelism**: Cluster-wide orchestration for efficiency
- **Integrity**: Checksum validation, post-restore verification
- **Resilience**: Retries, error handling, detailed logging

**Restore modes comparison**:

| Aspect | Local Restore | SSTableLoader | In-Place | New Cluster |
|--------|---------------|---------------|----------|-------------|
| **Downtime** | Required | None | Required | None* |
| **Speed** | Fastest | Slower | Fast | Moderate |
| **Topology** | Must match | Flexible | Must match | Flexible |
| **Use case** | DR, rollback | Partial, online | DR | Migration |

*New cluster starts clean, no downtime on source
