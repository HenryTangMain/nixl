# ETCD's Role in NIXL

ETCD serves as a **distributed metadata exchange service** that enables NIXL agents to discover each other and share the information needed to establish point-to-point communication.

## The Core Problem ETCD Solves

In distributed systems, before two nodes can transfer data, they need to exchange:
1. **Connection information** - How to reach each other (IP addresses, ports, network endpoints)
2. **Memory identifiers** - Remote keys/handles to access registered memory regions
3. **Backend capabilities** - What communication backends are available

**Without ETCD**: Applications must manually orchestrate this metadata exchange through custom side-channels or orchestration logic.

**With ETCD**: NIXL agents automatically publish and discover metadata through a centralized key-value store.

## ETCD vs Side-Channel Approaches

NIXL supports **two metadata exchange patterns**:

### Option 1: Side-Channel (Manual)
```
Application Code
    │
    ├─> Agent A: getLocalMD() → serialized_metadata_A
    │                            ↓
    │                     [Custom distribution logic]
    │                            ↓
    └─> Agent B: loadRemoteMD(metadata_A)
```

**Use case**: When you already have orchestration infrastructure (like MPI, custom control plane)

### Option 2: ETCD (Automatic)
```
Agent A                    ETCD Server               Agent B
   │                            │                       │
   │ sendLocalMD()              │                       │
   ├───────────────────────────>│                       │
   │ PUT /nixl/agents/A/        │                       │
   │                            │                       │
   │                            │<──────────────────────┤
   │                            │  fetchRemoteMD("A")   │
   │                            │                       │
   │                            │  GET /nixl/agents/A/  │
   │                            ├──────────────────────>│
   │                            │                       │
   │                            │  loadRemoteMD()       │
   │                            │  [automatic]          │
```

**Use case**: Cloud-native deployments, containerized environments, dynamic scaling

## Metadata Structure in ETCD

```
/nixl/agents/                              # Namespace prefix (configurable)
├── AgentA/
│   ├── metadata                           # Full metadata (default label)
│   │   ├── Connection info for each backend (UCX, GDS, etc.)
│   │   └── Remote identifiers for registered memory
│   ├── conn_info_1                        # Partial metadata (custom label)
│   └── conn_info_2                        # Another partial metadata
├── AgentB/
│   ├── metadata
│   └── ...
└── AgentC/
    └── metadata
```

## Complete ETCD Workflow

### Step 1: Agent Initialization with ETCD
```cpp
// Set ETCD endpoint
export NIXL_ETCD_ENDPOINTS="http://localhost:2379"
export NIXL_ETCD_NAMESPACE="/nixl/agents"  // Optional, default shown

// Create agent with ETCD enabled
nixlAgentConfig cfg(true);  // true = enable ETCD listener
nixlAgent agent("AgentA", cfg);
```

### Step 2: Backend Creation and Memory Registration
```cpp
// Create backend (e.g., UCX)
agent.createBackend("UCX", params, backend_handle);

// Register memory - backend creates local metadata
agent.registerMem(descriptor_list);

// At this point, agent has:
// - Connection info for UCX backend
// - Local metadata for registered memory
// But hasn't published to ETCD yet!
```

### Step 3: Publish Metadata to ETCD
```cpp
// Agent publishes all its metadata to ETCD
agent.sendLocalMD();

// Behind the scenes, NIXL:
// 1. Serializes connection info from all backends
// 2. Calls getPublicData() on each registered memory
// 3. Creates hierarchical structure:
//    {
//      "backends": {
//        "UCX": {
//          "connection_info": <serialized_bytes>,
//          "registered_memories": [
//            { "addr_range": ..., "remote_id": <bytes> },
//            ...
//          ]
//        }
//      }
//    }
// 4. Stores in ETCD at: /nixl/agents/AgentA/metadata
```

### Step 4: Discover and Load Remote Metadata
```cpp
// On another agent, fetch AgentA's metadata
agent_B.fetchRemoteMD("AgentA");

// Behind the scenes, NIXL:
// 1. Queries ETCD: GET /nixl/agents/AgentA/metadata
// 2. Deserializes the metadata structure
// 3. For each common backend:
//    - Calls loadRemoteConnInfo() to store connection info
//    - Calls loadRemoteMD() for each memory region
// 4. Caches everything locally
// 5. Now ready for P2P transfers!
```

### Step 5: Perform Transfer (No ETCD Involved)
```cpp
// Create and post transfer - uses cached metadata
agent_B.createXferReq(WRITE, local_desc, remote_desc, "AgentA", handle);
agent_B.postXferReq(handle);

// Data path does NOT touch ETCD!
// All transfers go directly through backend plugins (UCX, etc.)
```

### Step 6: Dynamic Scaling - Remove Agent
```cpp
// Agent going away invalidates its metadata
agent_A.invalidateLocalMD();

// Behind the scenes:
// - Removes all keys under /nixl/agents/AgentA/
// - Other agents watching for changes get notified
// - They automatically invalidate cached data for AgentA
```

## ETCD Watcher Pattern for Dynamic Discovery

```
Agent B (Running)                ETCD Server              New Agent C
    │                                 │                         │
    │ watch("/nixl/agents/*")         │                         │
    ├────────────────────────────────>│                         │
    │                                 │                         │
    │         [Watching for changes]  │                         │
    │                                 │                         │
    │                                 │<────────────────────────┤
    │                                 │  PUT /nixl/agents/C/    │
    │                                 │                         │
    │<────────────────────────────────┤                         │
    │  NOTIFICATION: New agent "C"    │                         │
    │                                 │                         │
    │ fetchRemoteMD("C")              │                         │
    ├────────────────────────────────>│                         │
    │                                 │                         │
    │ <metadata_C>                    │                         │
    │<────────────────────────────────┤                         │
    │                                 │                         │
    │ loadRemoteMD(metadata_C)        │                         │
    │ [Now can transfer with C]       │                         │
```

## ETCD in nixlbench (Benchmark Tool)

The benchmark uses ETCD for **two separate purposes**:

### 1. Worker Coordination (Runtime Layer)
```
nixlbench Worker 1          ETCD           nixlbench Worker 2
       │                      │                    │
       │  Register rank 0     │                    │
       ├─────────────────────>│                    │
       │                      │<───────────────────┤
       │                      │   Register rank 1  │
       │                      │                    │
       │  broadcastInt()      │                    │
       │  barrier()           │                    │
       │  reduceSumDouble()   │                    │
       │        ↑             │         ↑          │
       │        └─────────────┴─────────┘          │
       │     All coordinated via ETCD              │
```

**Purpose**: Synchronize benchmark workers, exchange test parameters, coordinate timing

### 2. NIXL Agent Metadata Exchange
```
Worker 1's nixlAgent     ETCD      Worker 2's nixlAgent
       │                    │              │
       │  sendLocalMD()     │              │
       ├───────────────────>│              │
       │                    │<─────────────┤
       │                    │ fetchRemoteMD()
       │                    │              │
       │  ← Ready for actual data transfers → │
       │                                   │
       │  ←─────── UCX/GDS/etc ──────────→ │
       │     (Data path, no ETCD)          │
```

**Purpose**: Same as above - exchange NIXL metadata for P2P transfers

## Key Characteristics

### 1. Control Path Only
- ETCD is used **only for metadata exchange** (control path)
- **Actual data transfers** go directly through backend plugins (UCX, GDS, etc.)
- No performance impact on data path

### 2. Optional Feature
```cpp
// ETCD disabled - manual metadata exchange
nixlAgentConfig cfg(false);
agent.getLocalMD(blob);      // Get serialized metadata
// ... manually send to other agents ...
agent.loadRemoteMD(remote_agent_name, blob);

// ETCD enabled - automatic exchange
nixlAgentConfig cfg(true);
agent.sendLocalMD();         // Publish to ETCD
agent.fetchRemoteMD("AgentB");  // Fetch from ETCD
```

### 3. Hierarchical Namespaces
- Default: `/nixl/agents/<agent_name>/metadata`
- Configurable via `NIXL_ETCD_NAMESPACE`
- Supports multiple NIXL deployments on same ETCD cluster

### 4. Partial Metadata Updates
```cpp
// Send only connection info, no memory registrations
nixl_opt_args_t opts;
opts.includeConnInfo = true;
opts.metadataLabel = "conn_only";
agent.sendLocalPartialMD(empty_dlist, &opts);

// Fetch specific label
opts.metadataLabel = "conn_only";
agent.fetchRemoteMD("AgentB", &opts);
```

**Use case**: Update connection info without re-sharing all memory registrations

## Environment Configuration

```bash
# Required: ETCD server endpoint(s)
export NIXL_ETCD_ENDPOINTS="http://etcd1:2379,http://etcd2:2379,http://etcd3:2379"

# Optional: Custom namespace (default: /nixl/agents/)
export NIXL_ETCD_NAMESPACE="/my_app/nixl/agents"

# Example: Multiple isolated deployments
export NIXL_ETCD_NAMESPACE="/prod/nixl/agents"     # Production
export NIXL_ETCD_NAMESPACE="/staging/nixl/agents"  # Staging
export NIXL_ETCD_NAMESPACE="/dev/nixl/agents"      # Development
```

## Comparison with Alternative Solutions

| Feature | ETCD | Redis | MPI | Custom Side-Channel |
|---------|------|-------|-----|---------------------|
| **Discovery** | Automatic | Automatic | Manual ranks | Manual |
| **Watch/Notify** | Built-in | Pub/Sub | No | Custom |
| **Consistency** | Strong (Raft) | Eventual | N/A | Depends |
| **Cloud-Native** | Yes | Yes | No | Depends |
| **Setup Complexity** | Low | Low | High | High |
| **NIXL Support** | Built-in | Not implemented | Not needed | Manual APIs |

## When to Use ETCD vs Side-Channel

### Use ETCD when:
- ✅ Deploying in Kubernetes/containerized environments
- ✅ Need dynamic agent discovery
- ✅ Agents join/leave frequently (elastic scaling)
- ✅ Want automatic metadata synchronization
- ✅ Multiple inference instances need to discover each other

### Use Side-Channel when:
- ✅ Already have orchestration framework (MPI, custom control plane)
- ✅ Static topology known at startup
- ✅ Want complete control over metadata exchange
- ✅ Operating in restricted network environments
- ✅ Prefer zero external dependencies

## Security Considerations

ETCD in NIXL currently uses:
- Basic endpoint configuration
- No authentication by default
- Namespace isolation via prefixes

For production:
```bash
# Use TLS-enabled endpoints
export NIXL_ETCD_ENDPOINTS="https://etcd.secure:2379"

# Use separate ETCD cluster for NIXL
# Apply ETCD RBAC and access controls
# Network isolation between nodes
```

## Performance Characteristics

- **Latency**: ETCD operations are ~1-10ms (depends on cluster size)
- **Impact**: Only affects agent initialization and scaling events
- **Data path**: Zero impact - transfers don't touch ETCD
- **Scalability**: ETCD handles 10,000s of keys easily
- **Throughput**: Metadata exchange is infrequent (initialization only)

## Summary

ETCD in NIXL is:
- **A metadata exchange service** - not for data transfers
- **Optional** - you can use manual side-channels instead
- **Control-path only** - zero impact on transfer performance
- **Cloud-native friendly** - perfect for Kubernetes and dynamic environments
- **Not a collective communication library** - that's what NCCL/MPI are for

Think of ETCD as the "phone book" that helps NIXL agents find each other before they start making actual data transfer "calls" through UCX/GDS/etc.
