# NIXL Architecture Diagram

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Application Layer                                 │
│  (AI Inference Frameworks, Distributed Training, Data Processing)        │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 │ North Bound API (NB API)
                                 │ - Create Agent
                                 │ - Register Memory
                                 │ - Create Transfer Request
                                 │ - Post Transfer
                                 │
┌────────────────────────────────▼────────────────────────────────────────┐
│                          nixlAgent                                       │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  • Agent Configuration & Lifecycle                               │   │
│  │  • Memory Registration Management                                │   │
│  │  • Metadata Exchange Coordination                                │   │
│  │  • Transfer Request Routing                                      │   │
│  │  • Backend Selection Logic                                       │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Plugin Manager                               │   │
│  │  • Plugin Discovery (static/dynamic)                             │   │
│  │  • Plugin Loading & Versioning                                   │   │
│  │  • Backend Engine Instantiation                                  │   │
│  │  • Capability Querying                                           │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 │ South Bound API (SB API)
                                 │ - registerMem() / deregisterMem()
                                 │ - connect() / disconnect()
                                 │ - prepXfer() / postXfer() / checkXfer()
                                 │ - getPublicData() / loadRemoteMD()
                                 │ - getNotifs() / genNotif()
                                 │
┌────────────────────────────────▼────────────────────────────────────────┐
│                        Backend Plugins (SB API)                          │
│                                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │     UCX      │  │     GDS      │  │    POSIX     │  │  LibFabric  │ │
│  │              │  │              │  │              │  │             │ │
│  │ • Remote ✓   │  │ • Local ✓    │  │ • Local ✓    │  │ • Remote ✓  │ │
│  │ • Local ✓    │  │ • Remote ✗   │  │ • Remote ✗   │  │ • Local ✓   │ │
│  │ • Notif ✓    │  │ • Notif ✗    │  │ • Notif ✗    │  │ • Notif ✓   │ │
│  │              │  │              │  │              │  │             │ │
│  │ DRAM/VRAM    │  │ VRAM/BLK     │  │ DRAM/FILE    │  │ DRAM/VRAM   │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────────┘ │
│                                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │  GPUNETIO    │  │   Mooncake   │  │    HF3FS     │  │     OBJ     │ │
│  │              │  │              │  │              │  │   (S3)      │ │
│  │ • Remote ✓   │  │ • Remote ✓   │  │ • Local ✓    │  │ • Local ✓   │ │
│  │ • Local ✓    │  │ • Local ✓    │  │ • Remote ✗   │  │ • Remote ✗  │ │
│  │ • Notif ✓    │  │ • Notif ✓    │  │ • Notif ✗    │  │ • Notif ✗   │ │
│  │              │  │              │  │              │  │             │ │
│  │ VRAM         │  │ DRAM/VRAM    │  │ VRAM/FILE    │  │ DRAM/OBJ    │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────────┘ │
│                                                                           │
│  ┌──────────────┐  ┌──────────────┐                                     │
│  │   GDS_MT     │  │    GUSLI     │                                     │
│  │              │  │              │                                     │
│  │ • Local ✓    │  │ • Local ✓    │                                     │
│  │ • Remote ✗   │  │ • Remote ✗   │                                     │
│  │ • Notif ✗    │  │ • Notif ✗    │                                     │
│  │              │  │              │                                     │
│  │ VRAM/BLK     │  │ VRAM/BLK     │                                     │
│  └──────────────┘  └──────────────┘                                     │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 │ Hardware Access Layer
                                 │
┌────────────────────────────────▼────────────────────────────────────────┐
│                     Physical Resources                                   │
│                                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │   Network    │  │  GPU Memory  │  │   Storage    │  │  CPU Memory │ │
│  │              │  │              │  │              │  │             │ │
│  │ • InfiniBand │  │ • CUDA VRAM  │  │ • NVMe/SSD   │  │ • DRAM      │ │
│  │ • Ethernet   │  │ • GPU Direct │  │ • HDD        │  │ • NUMA      │ │
│  │ • EFA (AWS)  │  │ • Fabric     │  │ • Object Stre│  │ • Pinned    │ │
│  │ • RoCE       │  │ • P2P        │  │ • Block Dev  │  │             │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

## Data Flow Diagrams

### 1. Agent Creation and Backend Initialization

```
Application                     nixlAgent                   Plugin Manager              Backend Plugin
    │                               │                             │                           │
    │ createAgent(name, config)     │                             │                           │
    ├──────────────────────────────>│                             │                           │
    │                               │                             │                           │
    │                               │ discoverPlugins()           │                           │
    │                               ├────────────────────────────>│                           │
    │                               │                             │                           │
    │                               │ <plugin list>               │                           │
    │                               │<────────────────────────────┤                           │
    │                               │                             │                           │
    │ createBackend(type, params)   │                             │                           │
    ├──────────────────────────────>│                             │                           │
    │                               │                             │                           │
    │                               │ createEngine(type, params)  │                           │
    │                               ├────────────────────────────>│                           │
    │                               │                             │                           │
    │                               │                             │ new BackendEngine(params) │
    │                               │                             ├──────────────────────────>│
    │                               │                             │                           │
    │                               │                             │ Constructor()             │
    │                               │                             │ getConnInfo()             │
    │                               │                             │ connect(self) [if local]  │
    │                               │                             │<──────────────────────────│
    │                               │                             │                           │
    │                               │ <backend handle>            │                           │
    │                               │<────────────────────────────┤                           │
    │                               │                             │                           │
    │ <agent handle>                │                             │                           │
    │<──────────────────────────────┤                             │                           │
```

### 2. Memory Registration Flow

```
Application                nixlAgent                Backend Plugin           Backend Metadata
    │                          │                          │                        │
    │ registerMem(desc, type)  │                          │                        │
    ├─────────────────────────>│                          │                        │
    │                          │                          │                        │
    │                          │ registerMem(desc)        │                        │
    │                          ├─────────────────────────>│                        │
    │                          │                          │                        │
    │                          │                          │ Pin memory             │
    │                          │                          │ Create local metadata  │
    │                          │                          ├───────────────────────>│
    │                          │                          │                        │
    │                          │ <local MD handle>        │                        │
    │                          │<─────────────────────────┤                        │
    │                          │                          │                        │
    │                          │ loadLocalMD(local MD)    │                        │
    │                          ├─────────────────────────>│                        │
    │                          │                          │                        │
    │                          │                          │ Create target metadata │
    │                          │                          ├───────────────────────>│
    │                          │                          │                        │
    │                          │ <target MD handle>       │                        │
    │                          │<─────────────────────────┤                        │
    │                          │                          │                        │
    │                          │ Store both handles       │                        │
    │                          │                          │                        │
    │ <registration handle>    │                          │                        │
    │<─────────────────────────┤                          │                        │
```

### 3. Metadata Exchange Between Agents

```
Agent A                    Agent B                    ETCD/KV Store         Backend Plugin A    Backend Plugin B
   │                          │                              │                      │                  │
   │ getLocalMetadata()       │                              │                      │                  │
   ├─────────────────────────>│                              │                      │                  │
   │                          │                              │                      │                  │
   │  ┌─ For each backend ────────────────────────────────────────────────┐        │                  │
   │  │                       │                              │             │        │                  │
   │  │ getConnInfo()         │                              │             │        │                  │
   │  ├───────────────────────┼──────────────────────────────┼─────────────┼───────>│                  │
   │  │                       │                              │             │        │                  │
   │  │ <conn info bytes>     │                              │             │        │                  │
   │  │<──────────────────────┼──────────────────────────────┼─────────────┼────────│                  │
   │  │                       │                              │             │        │                  │
   │  │ getPublicData(reg)    │                              │             │        │                  │
   │  ├───────────────────────┼──────────────────────────────┼─────────────┼───────>│                  │
   │  │                       │                              │             │        │                  │
   │  │ <remote ID bytes>     │                              │             │        │                  │
   │  │<──────────────────────┼──────────────────────────────┼─────────────┼────────│                  │
   │  │                       │                              │             │        │                  │
   │  └───────────────────────────────────────────────────────────────────┘        │                  │
   │                          │                              │                      │                  │
   │ <serialized metadata>    │                              │                      │                  │
   │                          │                              │                      │                  │
   │ publish(metadata)        │                              │                      │                  │
   ├──────────────────────────┼─────────────────────────────>│                      │                  │
   │                          │                              │                      │                  │
   │                          │ subscribe("agent_A")         │                      │                  │
   │                          ├─────────────────────────────>│                      │                  │
   │                          │                              │                      │                  │
   │                          │ <metadata of Agent A>        │                      │                  │
   │                          │<─────────────────────────────┤                      │                  │
   │                          │                              │                      │                  │
   │                          │ loadRemoteMetadata(bytes)    │                      │                  │
   │                          │                              │                      │                  │
   │                          │  ┌─ For each backend ────────────────────────────────────────────┐    │
   │                          │  │                           │                      │            │    │
   │                          │  │ loadRemoteConnInfo(conn)  │                      │            │    │
   │                          │  ├───────────────────────────┼──────────────────────┼────────────┼───>│
   │                          │  │                           │                      │            │    │
   │                          │  │ Store remote conn info    │                      │            │    │
   │                          │  │<──────────────────────────┼──────────────────────┼────────────┼────│
   │                          │  │                           │                      │            │    │
   │                          │  │ loadRemoteMD(remote ID)   │                      │            │    │
   │                          │  ├───────────────────────────┼──────────────────────┼────────────┼───>│
   │                          │  │                           │                      │            │    │
   │                          │  │ <remote MD handle>        │                      │            │    │
   │                          │  │<──────────────────────────┼──────────────────────┼────────────┼────│
   │                          │  │                           │                      │            │    │
   │                          │  └───────────────────────────────────────────────────────────────┘    │
   │                          │                              │                      │                  │
   │                          │ Ready for transfers          │                      │                  │
```

### 4. Transfer Request Flow

```
Application          nixlAgent            Backend Plugin         Physical Hardware
    │                    │                      │                        │
    │ createXferReq()    │                      │                        │
    ├───────────────────>│                      │                        │
    │                    │                      │                        │
    │                    │ Select backend       │                        │
    │                    │ Attach MD handles    │                        │
    │                    │                      │                        │
    │ <xfer handle>      │                      │                        │
    │<───────────────────┤                      │                        │
    │                    │                      │                        │
    │ postXfer(handle)   │                      │                        │
    ├───────────────────>│                      │                        │
    │                    │                      │                        │
    │                    │ prepXfer()           │                        │
    │                    ├─────────────────────>│                        │
    │                    │                      │                        │
    │                    │ <backend req handle> │                        │
    │                    │<─────────────────────┤                        │
    │                    │                      │                        │
    │                    │ postXfer()           │                        │
    │                    ├─────────────────────>│                        │
    │                    │                      │                        │
    │                    │ POSTED               │ Initiate DMA           │
    │                    │<─────────────────────┤───────────────────────>│
    │                    │                      │                        │
    │ checkXfer()        │                      │                        │
    ├───────────────────>│                      │                        │
    │                    │                      │                        │
    │                    │ checkXfer()          │                        │
    │                    ├─────────────────────>│                        │
    │                    │                      │                        │
    │                    │ IN_PROGRESS          │ DMA in progress        │
    │                    │<─────────────────────┤<──────────────────────>│
    │                    │                      │                        │
    │ IN_PROGRESS        │                      │                        │
    │<───────────────────┤                      │                        │
    │                    │                      │                        │
    │ ... polling ...    │                      │                        │
    │                    │                      │                        │
    │ checkXfer()        │                      │                        │
    ├───────────────────>│                      │                        │
    │                    │                      │                        │
    │                    │ checkXfer()          │                        │
    │                    ├─────────────────────>│                        │
    │                    │                      │                        │
    │                    │ DONE                 │ DMA complete           │
    │                    │<─────────────────────┤<──────────────────────>│
    │                    │                      │                        │
    │ DONE               │                      │                        │
    │<───────────────────┤                      │                        │
    │                    │                      │                        │
    │ releaseXfer()      │                      │                        │
    ├───────────────────>│                      │                        │
    │                    │                      │                        │
    │                    │ releaseReqH()        │                        │
    │                    ├─────────────────────>│                        │
    │                    │                      │                        │
    │                    │ SUCCESS              │                        │
    │                    │<─────────────────────┤                        │
```

## Backend Plugin Capability Matrix

| Backend    | Remote | Local | Notif | Memory Types      | Use Case                           |
|------------|--------|-------|-------|-------------------|------------------------------------|
| UCX        | ✓      | ✓     | ✓     | DRAM, VRAM        | Network communication, GPU-GPU     |
| LibFabric  | ✓      | ✓     | ✓     | DRAM, VRAM        | Fabric networks (EFA, Verbs)       |
| GPUNETIO   | ✓      | ✓     | ✓     | VRAM              | DOCA-based GPU networking          |
| Mooncake   | ✓      | ✓     | ✓     | DRAM, VRAM        | Mooncake transport                 |
| GDS        | ✗      | ✓     | ✗     | VRAM, BLK         | GPU Direct Storage                 |
| GDS_MT     | ✗      | ✓     | ✗     | VRAM, BLK         | Multi-threaded GDS                 |
| POSIX      | ✗      | ✓     | ✗     | DRAM, FILE        | File I/O (AIO, io_uring)           |
| HF3FS      | ✗      | ✓     | ✗     | VRAM, FILE        | HyperFile filesystem               |
| OBJ        | ✗      | ✓     | ✗     | DRAM, OBJ         | S3 object storage                  |
| GUSLI      | ✗      | ✓     | ✗     | VRAM, BLK         | Direct block device access         |

Legend:
- **Remote**: Supports inter-node transfers
- **Local**: Supports intra-node transfers
- **Notif**: Supports notifications
- **Memory Types**: Supported memory spaces

## Key Design Patterns

### 1. Plugin Selection Logic

```
┌─────────────────────────────────────────────────────────────┐
│  Transfer Request Creation                                   │
│                                                              │
│  1. Analyze source/target memory types                      │
│  2. Check if local or remote transfer                       │
│  3. Query available backends on both sides                  │
│  4. Filter by capability requirements:                      │
│     • supportsLocal/supportsRemote                          │
│     • getSupportedMems                                      │
│     • Memory regions registered with backend                │
│  5. Select first matching backend (or use preference list)  │
│  6. Attach appropriate metadata handles to descriptors      │
└─────────────────────────────────────────────────────────────┘
```

### 2. Memory Descriptor Flow

```
Registration:
  Memory Buffer → registerMem() → Local MD Handle
                              └→ loadLocalMD() → Target MD Handle

Remote Identifier Export:
  Local MD Handle → getPublicData() → Serialized Bytes → [Network] →
  → loadRemoteMD() → Remote MD Handle

Transfer:
  Initiator Side: Local MD Handle
  Target Side: Target MD Handle (local) or Remote MD Handle (remote)
```

### 3. Backend Instance Lifecycle

```
1. Discovery:     Plugin Manager scans plugin directories
2. Registration:  Plugins register with manager
3. Creation:      User requests backend → manager creates instance
4. Initialization: Backend constructor, getConnInfo(), self-connect
5. Operation:     Memory registration, transfers, notifications
6. Teardown:      Deregister memories, disconnect, destructor
```

## ETCD-Based Coordination

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Agent A    │         │  ETCD Server │         │   Agent B    │
│  (Node 1)    │         │              │         │  (Node 2)    │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       │ publish("/nixl/agents/A", metadata)             │
       ├───────────────────────>│                        │
       │                        │                        │
       │                        │<──────────────────────┤
       │                        │ watch("/nixl/agents/*")│
       │                        │                        │
       │                        │                        │
       │                        ├───────────────────────>│
       │                        │ notify: new agent "A"  │
       │                        │                        │
       │                        │<──────────────────────┤
       │                        │ get("/nixl/agents/A")  │
       │                        │                        │
       │                        ├───────────────────────>│
       │                        │ metadata of Agent A    │
       │                        │                        │
       │ watch("/nixl/agents/*")│                        │
       ├───────────────────────>│                        │
       │                        │                        │
       │                        │ publish("/nixl/agents/B", metadata)
       │                        │<──────────────────────┤
       │                        │                        │
       │<──────────────────────┤                        │
       │ notify: new agent "B"  │                        │
       │                        │                        │
       │ loadRemoteMetadata(B)  │                        │
       │                        │ loadRemoteMetadata(A)  │
       │                        │                        │
       │◄───────────────────────┼────────────────────────►
       │   Ready for P2P Transfers via Backend Plugins   │
```

## Performance Optimization Layers

```
┌─────────────────────────────────────────────────────────┐
│  Application Layer                                       │
│  • Batch multiple transfers                             │
│  • Reuse transfer handles (prep once, post multiple)    │
│  • Async operation with polling                         │
└────────────────────────────┬────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────┐
│  NIXL Agent Layer                                        │
│  • Progress threads for async progress                  │
│  • Backend selection optimization                       │
│  • Memory registration caching                          │
│  • Metadata serialization efficiency                    │
└────────────────────────────┬────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────┐
│  Backend Plugin Layer                                    │
│  • Transport-specific optimizations                     │
│  • Load balancing across descriptors                    │
│  • Connection pooling                                   │
│  • Zero-copy operations                                 │
│  • Pipeline parallel transfers                          │
└────────────────────────────┬────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────┐
│  Hardware Layer                                          │
│  • RDMA, GPUDirect, GDS                                 │
│  • DMA engines                                          │
│  • NVLink, PCIe P2P                                     │
└─────────────────────────────────────────────────────────┘
```
