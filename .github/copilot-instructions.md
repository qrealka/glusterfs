# GlusterFS AI Coding Agent Instructions

## Project Overview

GlusterFS is a distributed file system with a modular, plugin-based architecture built around **translators (xlators)**. The system uses a unique stack-based I/O framework where file operations flow through a graph of xlators, each transforming or forwarding operations via `STACK_WIND` and `STACK_UNWIND` macros.

## Core Architecture

### Xlator Plugin System

Translators are dynamically-loaded shared objects organized by category:
- **storage/** - Backend storage (posix)
- **cluster/** - Distribution/replication (dht, afr, ec)
- **performance/** - Caching/optimization (write-behind, read-ahead, io-cache)
- **protocol/** - Client/server communication
- **features/** - Extended functionality (locks, quota, changelog)
- **debug/** - Diagnostics (trace, error-gen, io-stats)

Each xlator exports required symbols via `dlsym`:
- `fops` - File operation dispatch table (82 operations: open, read, write, stat, etc.)
- `cbks` - Callbacks for inode/fd lifecycle events
- `init`/`fini` - Initialization and cleanup functions
- `xlator_api` structure with `.identifier` and `.category` (GF_MAINTAINED, GF_TECH_PREVIEW, etc.)

### STACK_WIND/STACK_UNWIND Pattern

File operations propagate through xlator chains using frame-based execution:

```c
// Forward operation to child xlator
STACK_WIND(frame, my_xlator_callback, FIRST_CHILD(this),
           FIRST_CHILD(this)->fops->operation, args...);

// Return results to parent xlator
STACK_UNWIND_STRICT(operation, frame, op_ret, op_errno, return_values...);
```

**Critical rules:**
- Each `STACK_WIND` requires a callback to handle child's response
- `STACK_UNWIND` must be called exactly once per frame to avoid leaks
- Frames track operation context: `frame->local` for xlator-private data
- Never assume synchronous execution - callbacks may execute on different threads

### Call Frames and Stacks

- `call_frame_t` - Execution context for one xlator in the chain
- `call_stack_t` - Complete operation chain from FUSE/protocol down to storage
- Memory pools (`mem_pool_new`) recycle frames/stacks to avoid allocation overhead
- `frame->local` holds xlator-specific state (must be allocated with `GF_CALLOC`)

## Build System

### Initial Setup
```bash
./autogen.sh              # Generate configure script
./configure               # Detect dependencies and generate Makefiles
make                      # Build all components
make install              # Install binaries and libraries
```

### Adding New Xlators
1. Generate skeleton: `python extras/create_new_xlator/generate_xlator.py <dir> <name> <prefix>`
2. Edit `configure.ac` to add xlator's Makefile paths
3. Update parent `Makefile.am` (e.g., `xlators/features/Makefile.am`) with new subdirectory
4. Implement required fops in `src/<name>.c`
5. Add `.sym` file exporting `fops`, `cbks`, `xlator_api`

## Testing Framework

Tests are shell scripts with `.t` extension using a domain-specific test language:

```bash
. $(dirname $0)/../include.rc    # Load test helpers
cleanup;                          # Clean previous test artifacts

TEST glusterd                     # Start glusterd (assert success)
TEST $CLI volume create $V0 ...   # Create volume
EXPECT 'Started' volinfo_field $V0 'Status'  # Assert expected value

cleanup;                          # Clean up
```

**Key variables** (from `tests/include.rc`):
- `$M0`, `$M1`, `$M2` - FUSE mount points
- `$B0` - Brick directory base
- `$V0`, `$V1` - Volume names
- `$H0` - Hostname
- Timeouts: `PROCESS_UP_TIMEOUT=45`, `HEAL_TIMEOUT=80`, etc.

**Running tests:**
```bash
./run-tests.sh                    # Run all tests (destructive!)
./run-tests.sh tests/basic/*.t    # Run specific tests
prove -vmfe '/bin/bash' path/to/test.t  # Run single test with verbose output
```

**WARNING:** Tests delete `GLUSTERD_WORKDIR` and kill gluster processes - never run on production systems.

## Code Conventions

### Formatting
- **clang-format** is mandatory before submission:
```bash
git show --pretty="format:" --name-only | grep -v "contrib/" | egrep "*\.[ch]$" | xargs clang-format -i
```

### Structure Member Documentation
Every struct member needs descriptive comments:
```c
struct my_struct {
    DBTYPE access_mode;    /* access mode for databases, can be
                            * DB_HASH, DB_BTREE (option access-mode) */
    gf_lock_t lock;        /* protects ->connections and ->state */
    void *ctx;             /* ref-counted; freed in destructor */
};
```

### Memory Management
- Use `GF_CALLOC`/`GF_FREE` (not malloc/free) for tracking
- Use `mem_pool_new` for frequently allocated structures
- Always check allocation failures and unwind with error
- Free `frame->local` before `STACK_UNWIND` if no longer needed

### Error Handling Pattern
```c
int my_fop(call_frame_t *frame, xlator_t *this, ...) {
    my_local_t *local = GF_CALLOC(1, sizeof(*local), gf_my_mt_local);
    if (!local) {
        goto err;
    }
    frame->local = local;
    
    STACK_WIND(frame, my_callback, FIRST_CHILD(this), ...);
    return 0;

err:
    STACK_UNWIND_STRICT(my_fop, frame, -1, ENOMEM, NULL);
    return 0;
}
```

## RPC and Protocol

- RPC programs/versions defined in `.x` files (XDR), converted via `rpcgen`
- **Never modify existing RPC structures** - add new program versions instead
- Client-server handshake (`GF_DUMP_DUMP`) negotiates supported program versions
- See `doc/developer-guide/rpc-for-glusterfs.new-versions.md` for compatibility rules

## Developer Workflows

### Submitting Changes
```bash
git commit -a -s -m "component: description"  # -s adds Signed-off-by
# Commit message must include "Fixes: #NNNN" or "Updates: #NNNN"
./submit-for-review.sh     # Or: git push origin HEAD:issueNNNN
```

### GitHub PR Requirements
- Reference issue with `Fixes: #NNNN` or `Updates: #NNNN` in commit
- Smoke tests auto-trigger; comment `/recheck smoke` to retry
- Comment `/run regression` for full regression (maintainer-only)
- Use `clang-format` before pushing
- Squash-and-merge policy preserves one commit per PR

## Key Files to Reference

- `libglusterfs/src/xlator.c` - Xlator loading/initialization
- `libglusterfs/src/glusterfs/stack.h` - Frame/stack macros and structures
- `doc/developer-guide/translator-development.md` - Complete xlator tutorial
- `doc/developer-guide/coding-standard.md` - Style guide with examples
- `tests/include.rc` - Test framework helpers and variables
- `extras/create_new_xlator/` - Xlator scaffolding tool

## Common Pitfalls

1. **Missing STACK_UNWIND** - Every code path must unwind or leak frames
2. **Wrong callback signature** - Use `_STRICT` variants to catch type errors
3. **Synchronous assumptions** - Never assume STACK_WIND returns before callback
4. **Default fops** - Unimplemented fops automatically passthrough to children (see `defaults.c`)
5. **Test cleanup** - Always call `cleanup;` at start/end of `.t` tests

## Debugging Resource Leaks

### Memory Leaks with Statedump

Statedump is the primary tool for diagnosing memory/fd/inode leaks in production:

```bash
# Generate statedump for brick processes
gluster volume statedump <volname>

# For specific process (client apps using libgfapi)
gluster volume statedump <volname> client <hostname>:<pid>

# Or directly via signal
kill -USR1 <pid-of-gluster-process>

# Files created in: gluster --print-statedumpdir
```

**Key statedump sections:**
- **Memory accounting** - `num_allocs` per data type (gf_common_mt_*):
  ```
  grep -w num_allocs glusterdump.<pid>.dump.<timestamp>
  ```
  Rising `num_allocs` indicates leak. Check tag to find allocation site.

- **Memory pools** - `hot-count` (active), `cold-count` (free), `cur-stdalloc`:
  ```
  [mempool]
  pool-name=glusterfs:dict_t
  hot-count=0
  cold-count=0
  cur-stdalloc=214  # <-- Heap allocations not yet freed
  max-stdalloc=220
  ```
  Rising `cur-stdalloc` (compile with `-DDEBUG`) indicates pool exhaustion/leak.

- **Call stacks** - Detect frame leaks or hangs:
  ```
  [global.callpool.stack.3.frame.2]
  translator=r2-replicate-0
  complete=0  # <-- Not unwound! Check why this xlator didn't unwind
  wind_to=children[i]->fops->lookup
  unwind_to=afr_lookup_cbk
  ```
  If `complete=0` but child completed, parent xlator lost the frame.

### Valgrind for Development

Build with debug options to enable Valgrind:
```bash
./configure --enable-debug --enable-valgrind
make && make install

# Run single xlator test
cd tests/basic/gfapi/
valgrind --leak-check=full --show-leak-kinds=all \
         --fullpath-after= ./gfapi-load-volfile sink.vol
```

**What these flags do:**
- `--enable-debug` - Disables memory pools, uses malloc/free directly
- `--enable-valgrind` - Prevents `dlclose()` so xlator symbols remain for stack traces

**Valgrind shows:**
```
==2450== 80 bytes in 1 blocks are definitely lost
==2450==    by 0x12F10CDA: init (/path/to/xlators/meta/src/meta.c:231)
```
Look for leaks in xlator `init()` without corresponding `fini()` cleanup.

## Testing Deep Dive

### Test Framework Helpers

Beyond basic `TEST`/`EXPECT`, key functions in `tests/include.rc` and `tests/volume.rc`:

```bash
# Wait for condition with timeout
EXPECT_WITHIN $PROCESS_UP_TIMEOUT "1" brick_up_status $V0 $H0 $B0/${V0}1

# Loop until success or timeout
TEST_WITHIN $HEAL_TIMEOUT "0" get_pending_heal_count $V0

# Volume introspection
volinfo_field $V0 'Status'           # Get volume status
brick_count $V0                       # Count bricks
volume_option $V0 'performance.stat-prefetch'  # Get option value
```

**Common patterns:**
```bash
# Create replicate volume
TEST $CLI volume create $V0 replica 3 $H0:$B0/${V0}{1,2,3,4,5,6}
TEST $CLI volume start $V0

# Mount FUSE (never use 'mount -t glusterfs')
TEST $GFS -s $H0 --volfile-id $V0 $M0

# Set volume options
TEST $CLI volume set $V0 performance.stat-prefetch off
```

### Geo-replication Test Setup

Geo-rep tests require special setup in `tests/00-geo-rep/`:
```bash
. $(dirname $0)/../geo-rep.rc  # Load geo-rep helpers

# Variables
primary=$GMV0
secondary=${H0}::${GSV0}
usr="nroot"  # Non-root user for geo-rep
ssh_url=$usr@$H0

# Create primary and secondary volumes
TEST $CLI volume create $GMV0 replica 2 $H0:$B0/${GMV0}{1,2,3,4}
TEST $CLI volume start $GMV0
TEST $CLI volume create $GSV0 replica 2 $H0:$B0/${GSV0}{1,2,3,4}
TEST $CLI volume start $GSV0

# Setup SSH keys (non-root)
${GLUSTER_LIBEXECDIR}/set_geo_rep_pem_keys.sh $usr $primary $secondary_vol

# Start geo-replication
TEST $CLI volume geo-replication $primary $secondary_url create push-puff
TEST $CLI volume geo-replication $primary $secondary_url start
```

### Test Timeouts

Adjust timeouts for slow systems (see `tests/include.rc`):
```bash
PROCESS_UP_TIMEOUT=45    # Process startup
HEAL_TIMEOUT=80          # Self-heal completion
REBALANCE_TIMEOUT=600    # Rebalance operations
PROBE_TIMEOUT=60         # Peer probe
```

## Glusterd and CLI Workflows

### Volume Lifecycle

**Volume creation** (`cli/src/cli-cmd-volume.c`):
```bash
# Basic distributed volume
gluster volume create test-vol server1:/bricks/brick1 server2:/bricks/brick2

# Replicate volume
gluster volume create test-vol replica 2 server{1,2}:/bricks/brick{1,2}

# Disperse (erasure coding) volume
gluster volume create test-vol disperse 6 redundancy 2 server{1..6}:/bricks/brick{1..6}
```

**Volume start/stop:**
```bash
gluster volume start test-vol           # Start all brick processes
gluster volume stop test-vol            # Stop bricks (data remains)
gluster volume delete test-vol          # Remove volume metadata
```

### Volume Options and Tuning

**Set options** (takes effect immediately on running volume):
```bash
gluster volume set test-vol performance.cache-size 256MB
gluster volume set test-vol network.ping-timeout 30
gluster volume set test-vol diagnostics.brick-log-level DEBUG
```

**Get options:**
```bash
gluster volume get test-vol all                    # All options
gluster volume get test-vol performance.cache-size # Specific option
```

### Statedump in Volume Management

**Generate statedump for troubleshooting:**
```bash
# Set custom statedump directory
gluster volume set test-vol server.statedump-path /var/log/glusterfs/dumps
mkdir -p /var/log/glusterfs/dumps

# Trigger statedump
gluster volume statedump test-vol        # All bricks
gluster volume statedump test-vol nfs    # NFS server
gluster volume statedump test-vol client server1:12345  # Specific client
```

### Volume Info Inspection

**In tests, use helper functions:**
```bash
EXPECT 'Started' volinfo_field $V0 'Status'
EXPECT '6' brick_count $V0
brick_up_status $V0 $H0 $B0/${V0}1  # Returns 1 if brick is up
```

**Check brick status:**
```bash
gluster volume status test-vol                 # All components
gluster volume status test-vol detail          # Include disk usage
gluster volume status test-vol server1:/brick1 # Specific brick
```

### Debugging Hangs via Statedump

When operations hang, statedump shows the call chain:
1. Take statedump: `gluster volume statedump $V0`
2. Look for stacks with `complete=0`
3. Find last xlator that didn't unwind - that's where the hang is
4. Check if child xlator completed (`complete=1`) but parent still waiting

## Syncop Framework

For threading contexts (self-heal, rebalance), use syncops (synchronous operations):
- `syncop_lookup()`, `syncop_readv()`, etc. wrap async fops
- Implemented via coroutines - yielding on `STACK_WIND`, resuming on unwind
- See `doc/developer-guide/syncop.md` for lifecycle details

## AFR (Automatic File Replication) Architecture

### Core Concepts

AFR provides **synchronous replication** across multiple bricks using a transaction-based approach:

**Key data structures:**
- `afr_private_t` - Per-volume state (child xlators, lock domains, healing config)
- `afr_local_t` - Per-transaction state (operation context, child responses, locks held)

**Changelog extended attributes:**
```bash
trusted.afr.<volname>-client-<N>  # Pending operations on brick N
trusted.afr.dirty                  # Marks file modified during transaction

# Values are 24-byte hex arrays tracking:
# [metadata_pending][data_pending][entry_pending]
```

### AFR Transaction Phases

**Write operations** (inode-write, dir-write):
1. **Pre-op**: Acquire locks (`GF_FILE_LK` for files, `GF_DIR_LK` for dirs)
   - If lock fails: retry with blocking locks, mark failed bricks as dead
2. **Mark pending**: Set changelog xattrs on all alive children
3. **Perform operation**: Execute fop on all alive children
4. **Post-op**: Clear pending xattrs on successful children
5. **Unlock**: Release all locks

**Read operations** (inode-read, dir-read):
- Try first available child
- On failure, try next child in round-robin
- No locking required

### Self-Heal Daemon (shd)

**Process architecture:**
- One shd per server node (shared across volumes)
- Graph contains: io-stats → replicate xlators → client xlators (all children)

**Two heal modes:**

**Index heal** (automatic, every 600s or on-demand):
```bash
# Triggered by:
gluster volume heal <volname>
# Or: brick coming back online
# Or: cluster.heal-timeout expires

# Heals files in: .glusterfs/indices/xattrop/
# These are hardlinks to files needing heal (created in FOP pre-op)
```

**Full heal** (manual, for disk replacement):
```bash
gluster volume heal <volname> full

# Crawls entire volume tree from root
# Runs only on node with highest UUID per replica set
```

**Heal process per file:**
1. Take self-heal domain lock (`GF_DIR_LK`/`GF_FILE_LK`)
2. Determine source (good copy) from changelog xattrs
3. Perform metadata → data → entry heal in order
4. Clear changelog xattrs and index entries

### Split-Brain Detection and Resolution

**Occurs when:** Multiple bricks modified while others were down, with conflicting changes.

**Detection** - File shows non-zero pending xattrs in both directions:
```bash
# On brick1: trusted.afr.vol-client-1 = non-zero (brick2 needs heal)
# On brick2: trusted.afr.vol-client-0 = non-zero (brick1 needs heal)
```

**Resolution:**
```bash
# 1. Identify split-brain files
gluster volume heal <volname> info split-brain

# 2. Examine xattrs to find correct copy
getfattr -d -m . -e hex <file-path-on-brick>

# 3. Reset xattr on bad copy
setfattr -n trusted.afr.<volname>-client-<N> -v 0x000000000000000000000000 <bad-file>

# 4. Trigger heal
ls -l <file-on-mount>
```

### Building and Testing AFR

**Build focus:**
```bash
# AFR lives in cluster translator category
./configure
make  # Builds xlators/cluster/afr/

# Key files:
# - afr.c: Init/fini, lock management
# - afr-inode-write.c: Write transaction logic
# - afr-self-heal-*.c: Healing algorithms
# - afr-read-txn.c: Read path logic
```

**Testing AFR:**
```bash
# Source AFR test helpers
. $(dirname $0)/../afr.rc

# Create replicate volume
TEST $CLI volume create $V0 replica 3 $H0:$B0/${V0}{1,2,3}
TEST $CLI volume start $V0

# Check brick status
EXPECT_WITHIN $CHILD_UP_TIMEOUT "1" afr_child_up_status $V0 0
EXPECT_WITHIN $CHILD_UP_TIMEOUT "1" afr_child_up_status_in_shd $V0 0

# Verify self-heal completion
EXPECT_WITHIN $HEAL_TIMEOUT "Y" is_file_heal_done $B0/${V0}1 $B0/${V0}2 file.txt

# Check pending heal count
get_pending_heal_count $V0  # Should be 0 when healed
```

**Common AFR debugging patterns:**
- Check changelog xattrs: `getfattr -d -m trusted.afr <file>`
- Monitor index heal queue: `ls .glusterfs/indices/xattrop/`
- Check shd logs: `/var/log/glusterfs/glustershd.log`
- Verify heal status: `gluster volume heal <vol> info`

## NFS Server Architecture

### Overview

GlusterFS includes a **legacy NFSv3 server** (`xlators/nfs/server/`) that exports volumes via NFS protocol:
- Standalone process (not kernel NFS)
- Translates NFSv3 to GlusterFS fops
- Supports MOUNT, NFS, NLM, ACL protocols

**Modern alternative:** NFS-Ganesha (separate project, recommended for new deployments)

### NFS Server Components

**Protocol layers:**
```
Client NFSv3 → RPC layer → NFS3 xlator → Volume graph
                         ↓
                    MOUNT3 xlator (exports)
                         ↓
                     NLM4 xlator (locking)
```

**Key files:**
- `nfs.c` - Main NFS xlator, protocol version initialization
- `nfs3.c` - NFSv3 protocol implementation
- `nfs3-fh.c` - File handle (nfs3_fh) generation/validation
- `mount3.c` - MOUNT protocol (export discovery)
- `nfs-fops.c` - NFS → GlusterFS fop translation

**File handle structure (`nfs3_fh`):**
- Maps NFS file handles to GlusterFS gfids
- Contains export ID to identify volume
- Used for stateless operation (NFS requirement)

### Building and Testing NFS

**Build configuration:**
```bash
./configure  # NFS built by default
make         # Builds xlators/nfs/server/

# Disable NFS in volume:
gluster volume set <vol> nfs.disable on
```

**NFS-specific build files:**
- `configure.ac`: Check for `xlators/nfs/` entries
- `xlators/nfs/server/src/Makefile.am`: NFS server build rules

**Testing NFS:**
```bash
# Source NFS test helpers
. $(dirname $0)/../nfs.rc

# Create volume with NFS enabled
TEST $CLI volume create $V0 $H0:$B0/${V0}1
TEST $CLI volume set $V0 nfs.disable false
TEST $CLI volume start $V0

# Wait for NFS export
EXPECT_WITHIN $NFS_EXPORT_TIMEOUT "1" is_nfs_export_available $V0

# Mount via NFS
mount_nfs localhost:/$V0 $N0
# Or manually:
# mount -t nfs -o vers=3,nolock localhost:/$V0 $N0

# Unmount
umount_nfs $N0
```

**NFS debugging:**
- Check exports: `showmount -e localhost`
- NFS logs: `/var/log/glusterfs/nfs.log`
- RPC stats: `rpcinfo -p localhost`
- Trace NFS ops: Enable `nfs.log-level DEBUG` option

### NFS vs FUSE Access

**NFS characteristics:**
- Stateless protocol (every request self-contained)
- File handle based (not path based)
- Better for legacy app compatibility
- Network protocol overhead

**FUSE characteristics:**
- Kernel module integration
- Direct file system semantics
- Better performance for local access
- Supports extended attributes natively

## Integration Points

- **FUSE** - `xlators/mount/fuse/` translates kernel FUSE to GlusterFS fops
- **glusterd** - Management daemon in `cli/` and `libglusterd/`
- **geo-replication** - `geo-replication/syncdaemon/` for disaster recovery
- **NFS** - Legacy NFS server in `xlators/nfs/` (consider NFS-Ganesha for production)
