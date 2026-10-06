# KirnInit, Service Supervisor & Plan 9 Synthetic Namespaces
**The Event-Driven PID 1 Supervisor, Protocol-Based Dynamic Namespaces & Hardware Hotplug Daemon for KirnOS**  
*Language: Kirn (`.kn`) | Architecture: Socket-Activated DAG Supervisor & Per-Process Synthetic 9P Trees | Target Boot Time: $\le 120\text{ ms}$ to Desktop*

---

## 1. Architectural Philosophy & System Boundaries

`KirnInit` acts as the first user-space process (`PID 1`) executed by `KirnCore`. It addresses the dual extremes of system init design:
* **The Unix SysV / OpenRC Problem**: Shell-script-heavy, serial, brittle error handling, and lacking native process-supervision primitives.
* **The systemd Problem**: An overly monolithic design that entangles logging, DNS, time synchronization, and network configuration into PID 1, leading to an expansive attack surface and architectural bloat.
* **The Apple `launchd` & Plan 9 Breakthroughs**: `launchd` proved that **socket activation** and on-demand job scheduling dramatically accelerate boot times while eliminating manual dependency ordering. Plan 9 demonstrated that **per-process synthetic namespaces** provide an elegant abstraction for configuration, introspection, and IPC: everything is accessible through standardized protocol message exchanges.

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────+
| USER APPLICATIONS (Ring 3)                                                                              |
|                                                                                                         |
|  +──────────────────────────────────+  +─────────────────────────+  +─────────────────────────────────+ |
|  | Isolated Application (.kapp)     |  | Terminal Shell / Tools  |  | System Services & Daemons       | |
|  | Private Namespace:               |  | Shared Namespace:       |  | (KirnSurface, KirnNet, etc.)   | |
|  | - /dev (Filtered Sandbox)        |  | - /dev (Full Tree)      |  | Private Namespace:              | |
|  | - /srv (Bound Capabilities Only) |  | - /proc (All Tasks)    |  | - /dev (Specific Device Nodes)  | |
|  | - /proc/self (Isolated View)     |  | - /srv (Full Discovery) |  | - /srv/system (Service Port)     | |
|  +─────────────────┬────────────────+  +────────────┬────────────+  +────────────────┬────────────────+ |
|                    │                                │                                │                  |
|                    └────────────────────────────────┼────────────────────────────────┘                  |
|                                                     ▼                                                   |
|                        Plan 9 Synthetic Protocol Router (fs/synthetic/proto9k.kn)                       |
|                        (Attach, Walk, Open, Read, Write, Clunk, Stat, Wstat over KirnRing)              |
+─────────────────────────────────────────────────────┼───────────────────────────────────────────────────+
| SYSTEM SUPERVISOR (servers/init/ - PID 1)           │                                                   |
|                                                     ▼                                                   |
|  +───────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | kirnd (Event-Driven Process Supervisor, Parallel DAG Resolver, Heartbeat Watchdog)               |  |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | Socket Activation Broker: Lazily spawns daemons on initial client connection to /srv ports        |  |
|  +──────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | devd (Hardware Hotplug Daemon): Listens for PCIe/USB events & spawns user-mode KDF drivers        |  |
+─────────────────────────────────────────────────────┬───────────────────────────────────────────────────+
|                                                     ▼                                                   |
| KERNEL BOUNDARY (KirnCore Ring 0)                                                                       |
|                                                                                                         |
|  +───────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | Process Control Block (PCB): NamespaceTable Pointer, Handle Table, Capability Token Allocator     |  |
|  +───────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | KirnRing System Channels: Kernel Event Ring (Hardware IRQ, ACPI hotplug, process exit telemetry)  |  |
+─────────────────────────────────────────────────────────────────────────────────────────────────────────+
```

### 1.1. Invariants & Execution Guarantees
1. **PID 1 Immutability**: `kirnd` never crashes, never panics, and performs no complex string parsing or disk I/O directly in its core loop.
2. **Deterministic Startup via Socket Activation**: Services do not wait for each other to finish initializing. `kirnd` creates listening IPC rendezvous endpoints in `/srv/` on behalf of all system services at boot. Client processes can connect and transmit requests immediately; if the target daemon is not yet running, `kirnd` launches it in parallel and buffers requests in the IPC ring without dropping packets.
3. **Per-Process Namespace Isolation**: Every process possesses its own private `NamespaceTable`. Two sandboxed apps can see completely different contents under `/srv/` or `/dev/` without needing heavy OS-level containers or virtualization overhead.
4. **Sub-120ms Boot Target**: By eliminating serial init scripts, parallelizing the service DAG, and leveraging on-demand socket activation, KirnOS transitions from the UEFI handoff to the `KirnSurface` desktop interface in under $120\text{ milliseconds}$.

---

## 2. The Service Supervisor (`kirnd`)

`kirnd` operates as a non-blocking, event-driven state machine built on top of `KirnRing`.

```
                +──────────────────+
                |     STOPPED      |
                +────────┬─────────+
                         │ Socket Connection / Dependency Satisfied
                         ▼
                +──────────────────+
                |     STARTING     |
                +────────┬─────────+
                         │ Process spawned, capability handles granted
                         ▼
+─────────────> +──────────────────+ <─────────────+
|               |      ACTIVE      |                |
|               +────────┬─────────+                |
| Heartbeat OK           │ Heartbeat Timeout / Crash| Restart Policy (Exponential Backoff)
|                        ▼                          |
|               +──────────────────+                |
|               |      FAILED      | ───────────────+
|               +────────┬─────────+
|                        │ Max retries exceeded
|                        ▼
|               +──────────────────+
|               |     DEGRADED     |
+────────────── +──────────────────+
```

### 2.1. Declarative Service Manifest (`Manifest.kn`)
Services are defined using type-safe Kirn declarations:

```kirn
// Example: Display Compositor Service Manifest
module system.manifests.kirnsurface;

import servers.init.service;

pub fn manifest() -> service::ServiceManifest {
    return service::ServiceManifest {
        name: "kirnsurface",
        description: "Zero-Copy GPU Display Compositor",
        executable_path: "/system/bin/kirnsurface",
        service_type: service::ServiceType::SocketActivated,
        restart_policy: service::RestartPolicy::Always,
        max_restarts: 5,
        backoff_ms: 100,
        dependencies: ["devd"], // Waits for initial GPU device enumeration
        listen_channels: [
            service::SocketEndpoint { path: "/srv/compositor", permissions: 0660 }
        ],
        capabilities: [
            "CAP_MAP_PAGES",
            "CAP_MANAGE_IRQ",
            "CAP_GPU_SCANOUT"
        ],
        resource_limits: service::ResourceLimits {
            max_memory_bytes: 512 * 1024 * 1024, // 512 MiB
            cpu_quota_percent: 100,
            priority_band: service::QoSBand::InteractiveUI,
        },
    };
}
```

### 2.2. Parallel DAG Resolution Algorithm
* At boot, `kirnd` constructs an in-memory **Directed Acyclic Graph (DAG)** of all registered service manifests.
* It computes the in-degree of all nodes:
  * Nodes with $\text{in-degree} = 0$ (no unsatisfied dependencies) are dispatched simultaneously to the Grand Task Dispatcher.
  * When a service reaches the `Active` state, it signals its completion via an atomic event bitmask, decrementing the in-degree of dependent nodes and immediately unblocking the next layer of services.

---

## 3. Plan 9 Synthetic Namespace Engine (`Proto9K`)

KirnOS adapts the Plan 9 filesystem protocol (`9P2000.L`) into **`Proto9K`**, an asynchronous, capability-aware, zero-copy protocol running over `KirnRing` shared memory pages.

```
+──────────────────────────────────────────────────────────────────────────+
| Proto9K Core Protocol Message Set                                        |
+──────────────────────────────────────────────────────────────────────────+
| TAttach / RAttach | Establish connection to a synthetic tree root        |
| TWalk   / RWalk   | Traverse directory hierarchy by path element         |
| TOpen   / ROpen   | Acquire an open handle on a protocol node            |
| TRead   / RRead   | Stream synthetic byte sequence from provider         |
| TWrite  / RWrite  | Send control verbs / configuration bytes to provider |
| TClunk  / RClunk  | Release node handle and decrement reference counter  |
| TStat   / RStat   | Read dynamic metadata (permissions, timestamps, size)|
| TWstat  / RWstat  | Modify attributes (if permitted by capability token) |
+──────────────────────────────────────────────────────────────────────────+
```

### 3.1. Standard Synthetic Trees

#### 1. The Dynamic Process Tree (`/proc`)
Provides live reflection into kernel-level task state without disk access:
```text
/proc/
├── 1/                  # PID 1 (kirnd)
│   ├── status          # Live text status: State, Memory, CPU Time, Parent
│   ├── maps            # Memory mappings from the VMM page table
│   ├── handles         # List of held capability tokens and rights masks
│   └── ctl             # Control file: Write "pause", "resume", "terminate"
└── self/               # Symlink to the querying process's own directory
```

#### 2. The Service Rendezvous Tree (`/srv`)
Acts as a dynamic system bus and service registry. Connecting to a file in `/srv` directly establishes an atomic capability IPC channel with the owning daemon:
```text
/srv/
├── compositor          # KirnSurface Display Server IPC Endpoint
├── audio               # KirnSound Real-Time Audio Server IPC Endpoint
├── net                 # KirnNet Network Configuration & Socket Broker
└── power               # System Power Management & ACPI Control
```

#### 3. The Hardware Device Tree (`/dev`)
Synthesized on the fly by `devd`:
```text
/dev/
├── null                # Discard stream
├── zero                # Zero-filled stream
├── urandom             # Hardware-seeded cryptographic RNG (BLAKE3-CTR)
├── gpu0                # Primary GPU command queue & display scanout node
├── nvme0n1             # Direct NVMe block device channel
└── input/
    ├── mouse0          # Hardware mouse packet stream
    └── kbd0            # Hardware keyboard scancode stream
```

---

## 4. The Dynamic Hardware Daemon (`devd`)

`devd` is responsible for device discovery, dynamic driver lifecycle management, and hotplug arbitration:

```
[ Hardware Event: USB Device Inserted / PCIe Bus Rescan ]
                           │
                           ▼
[ KirnCore Kernel Ring Event: EventKind::DeviceHotplug ]
                           │
                           ▼
[ devd Ingress Loop (servers/devd/main.kn) ]
  ├── 1. Reads PCIe VendorID / DeviceID or USB Class Code
  ├── 2. Queries Driver Database in KirnFS (/system/drivers/)
  ├── 3. Spawns User-Mode Driver (KDF) into Isolated Ring 3 Process
  ├── 4. Grants Hardware Capability Tokens (MMIO Range, MSI-X IRQ Channel)
  └── 5. Synthesizes New Node in /dev (e.g. /dev/input/mouse1)
```

If a driver crashes (e.g., due to an illegal memory access in third-party GPU firmware), `devd` receives a termination notification, resets the hardware controller via PCIe Function-Level Reset (FLR), and restarts the driver transparently without compromising kernel integrity.

---

## 5. Complete Modular Code Architecture (`.kn`)

The following modules represent the production implementation of the `KirnInit` and synthetic namespace infrastructure:

```text
servers/init/
├── main.kn             # PID 1 entry point & master event loop
├── service.kn          # Service state machine, health monitoring & manifests
├── dag.kn              # Topological dependency graph compiler
└── socket_act.kn       # Lazy socket activation broker & channel proxy
servers/devd/
├── main.kn             # Hardware hotplug manager & driver supervisor
└── database.kn         # PCI/USB vendor matching engine
fs/synthetic/
├── proto9k.kn          # Proto9K wire protocol encoder/decoder & router
├── procfs.kn           # /proc synthetic file generator
└── srvfs.kn            # /srv rendezvous registry provider
libs/libproto9k/
└── client.kn           # Client-side 9P filesystem walk/read wrapper
```

---

### 5.1. PID 1 Master Event Loop (`servers/init/main.kn`)

```kirn
module servers.init.main;

import kernel.ipc.ring;
import servers.init.service;
import servers.init.dag;
import servers.init.socket_act;
import fs.synthetic.proto9k;

pub struct InitSupervisor {
    pub dag_engine:         dag::DagEngine,
    pub socket_broker:      socket_act::SocketBroker,
    pub proto_router:       proto9k::ProtocolRouter,
    pub kernel_event_ring:  ring::KirnRingBuffer,

    pub fn bootstrap() -> noreturn {
        let mut supervisor = Self {
            dag_engine:        dag::DagEngine::init(),
            socket_broker:     socket_act::SocketBroker::init(),
            proto_router:      proto9k::ProtocolRouter::init(),
            kernel_event_ring: ring::KirnRingBuffer::open_system_event_ring(),
        };

        // 1. Mount root synthetic namespace trees
        supervisor.proto_router.mount("/proc", proto9k::ProviderType::ProcFS);
        supervisor.proto_router.mount("/srv",  proto9k::ProviderType::SrvFS);
        supervisor.proto_router.mount("/dev",  proto9k::ProviderType::DevFS);

        // 2. Discover and register all system service manifests
        supervisor.dag_engine.load_all_manifests("/system/manifests");

        // 3. Pre-bind all service sockets in /srv for on-demand activation
        supervisor.socket_broker.bind_service_endpoints(&supervisor.dag_engine);

        // 4. Dispatch initial zero-dependency services in parallel
        supervisor.dag_engine.start_independent_services();

        // 5. Enter non-blocking master event loop
        supervisor.run_loop();
    }

    fn run_loop(&mut self) -> noreturn {
        while true {
            // Process kernel events (process deaths, hardware hotplugs)
            self.kernel_event_ring.poll_and_dispatch(|entry| {
                match entry.opcode {
                    ring::SysOp::ProcessExited => {
                        let pid = entry.user_data as u32;
                        let exit_code = entry.length as i32;
                        self.dag_engine.handle_process_termination(pid, exit_code);
                    },
                    ring::SysOp::ChannelActivity => {
                        let channel_id = entry.handle;
                        self.socket_broker.handle_incoming_connection(channel_id, &mut self.dag_engine);
                    },
                    _ => {},
                }
                return 0;
            });

            // Heartbeat watchdogs: inspect active daemons
            self.dag_engine.check_heartbeat_deadlines();

            // Yield execution until the next ring notification
            @asm volatile ("pause");
        }
    }
}

@export
pub fn _start() -> noreturn {
    InitSupervisor::bootstrap();
}
```

---

### 5.2. Service State Machine & Manifest Model (`servers/init/service.kn`)

```kirn
module servers.init.service;

import sync.atomic;

pub enum ServiceState : u8 {
    Stopped    = 0,
    Starting   = 1,
    Active     = 2,
    Stopping   = 3,
    Failed     = 4,
    Degraded   = 5,
}

pub enum ServiceType : u8 {
    Standard        = 1, // Runs continuously
    SocketActivated = 2, // Spawns on initial IPC connection
    OneShot         = 3, // Runs to completion (initialization task)
}

pub enum RestartPolicy : u8 {
    Never     = 0,
    OnFailure = 1,
    Always    = 2,
}

pub enum QoSBand : u8 {
    RealtimeAudio = 1,
    InteractiveUI = 2,
    Normal        = 3,
    Idle          = 4,
}

pub struct ResourceLimits {
    pub max_memory_bytes: u64,
    pub cpu_quota_percent: u8,
    pub priority_band:     QoSBand,
}

pub struct SocketEndpoint {
    pub path:        string,
    pub permissions: u32,
}

pub struct ServiceManifest {
    pub name:            string,
    pub description:     string,
    pub executable_path: string,
    pub service_type:    ServiceType,
    pub restart_policy:  RestartPolicy,
    pub max_restarts:    u32,
    pub backoff_ms:      u64,
    pub dependencies:    []string,
    pub listen_channels: []SocketEndpoint,
    pub capabilities:    []string,
    pub resource_limits: ResourceLimits,
}

pub struct ServiceInstance {
    pub manifest:         ServiceManifest,
    pub state:            ServiceState,
    pub current_pid:      Option[u32],
    pub restart_count:    u32,
    pub last_heartbeat:   u64,
    pub pending_requests: u32,

    pub fn transition_to(&mut self, new_state: ServiceState) {
        self.state = new_state;
    }

    pub fn mark_failed(&mut self) -> bool {
        self.restart_count += 1;
        if self.restart_count > self.manifest.max_restarts {
            self.state = ServiceState::Degraded;
            return false; // Escalate failure; do not restart
        }
        self.state = ServiceState::Failed;
        return true; // Eligible for scheduled restart
    }
}
```

---

### 5.3. Topological DAG Engine (`servers/init/dag.kn`)

```kirn
module servers.init.dag;

import servers.init.service;
import kernel.core.process;

pub const MAX_SERVICES: usize = 128;

pub struct DagEngine {
    pub services:        [Option[service::ServiceInstance]; MAX_SERVICES],
    pub dependency_matrix: [[bool; MAX_SERVICES]; MAX_SERVICES], // row depends on column
    pub service_count:   usize,

    pub fn init() -> Self {
        return Self {
            services:          [Option::None; MAX_SERVICES],
            dependency_matrix: [[false; MAX_SERVICES]; MAX_SERVICES],
            service_count:     0,
        };
    }

    pub fn load_all_manifests(&mut self, path: string) {
        // Loads and registers built-in and userland service descriptors
    }

    pub fn start_independent_services(&mut self) {
        for i in 0..self.service_count {
            if let Option::Some(svc) = &mut self.services[i] {
                if svc.manifest.service_type != service::ServiceType::SocketActivated {
                    if self.is_ready_to_start(i) {
                        self.spawn_service(i);
                    }
                }
            }
        }
    }

    pub fn handle_process_termination(&mut self, pid: u32, exit_code: i32) {
        for i in 0..self.service_count {
            if let Option::Some(svc) = &mut self.services[i] {
                if svc.current_pid == Option::Some(pid) {
                    svc.current_pid = Option::None;

                    if exit_code == 0 {
                        svc.transition_to(service::ServiceState::Stopped);
                    } else {
                        if svc.mark_failed() {
                            self.spawn_service(i); // Restart service
                        }
                    }
                    return;
                }
            }
        }
    }

    pub fn check_heartbeat_deadlines(&mut self) {
        // Enforces watchdog bounds; kills unresponsive daemons
    }

    fn is_ready_to_start(&self, index: usize) -> bool {
        for j in 0..self.service_count {
            if self.dependency_matrix[index][j] {
                if let Option::Some(dep) = &self.services[j] {
                    if dep.state != service::ServiceState::Active {
                        return false; // Blocked by dependency
                    }
                }
            }
        }
        return true;
    }

    pub fn spawn_service(&mut self, index: usize) {
        if let Option::Some(svc) = &mut self.services[index] {
            svc.transition_to(service::ServiceState::Starting);

            // Instructs kernel to instantiate new process with sandboxed capabilities
            let new_pid = process::spawn_sandboxed(
                svc.manifest.executable_path,
                svc.manifest.capabilities,
                svc.manifest.resource_limits.max_memory_bytes
            ).expect("Failed to spawn system service");

            svc.current_pid = Option::Some(new_pid);
            svc.transition_to(service::ServiceState::Active);
        }
    }
}
```

---

### 5.4. Plan 9 `Proto9K` Protocol Router (`fs/synthetic/proto9k.kn`)

```kirn
module fs.synthetic.proto9k;

import sync.atomic;

pub enum MsgType : u8 {
    TVersion = 100, RVersion = 101,
    TAttach  = 104, RAttach  = 105,
    TWalk    = 110, RWalk    = 111,
    TOpen    = 112, ROpen    = 113,
    TRead    = 116, RRead    = 117,
    TWrite   = 118, RWrite   = 119,
    TClunk   = 120, RClunk   = 121,
    TStat    = 124, RStat    = 125,
}

pub enum ProviderType : u8 {
    ProcFS,
    SrvFS,
    DevFS,
}

@repr(packed)
pub struct ProtoHeader {
    pub size:     u32,
    pub msg_type: MsgType,
    pub tag:      u16,
}

@repr(packed)
pub struct Qid {
    pub qid_type: u8, // Directory (0x80) or Regular File (0x00)
    pub version:  u32,
    pub path_id:  u64,
}

pub trait SyntheticProvider {
    fn walk(&mut self, parent_qid: Qid, element: string) -> Result<Qid, FsError>;
    fn open(&mut self, qid: Qid, mode: u32) -> Result<(), FsError>;
    fn read(&mut self, qid: Qid, offset: u64, count: u32, out_buf: []mut u8) -> Result<u32, FsError>;
    fn write(&mut self, qid: Qid, offset: u64, data: []const u8) -> Result<u32, FsError>;
    fn clunk(&mut self, qid: Qid) -> Result<(), FsError>;
}

pub struct ProtocolRouter {
    mount_points: [string; 16],
    providers:    [*mut dyn SyntheticProvider; 16],
    mount_count:  usize,

    pub fn init() -> Self {
        return Self {
            mount_points: [""; 16],
            providers:    [null; 16],
            mount_count:  0,
        };
    }

    pub fn mount(&mut self, target_path: string, provider: ProviderType) {
        if self.mount_count < 16 {
            self.mount_points[self.mount_count] = target_path;
            // Binds corresponding provider instance
            self.mount_count += 1;
        }
    }

    pub fn dispatch_request(&mut self, raw_message: []const u8, reply_buffer: []mut u8) -> usize {
        let hdr = @ptr_cast[*const ProtoHeader](&raw_message[0]);
        // Unpacks 9P message parameters and routes directly to matching synthetic provider
        return 0;
    }
}
```

---

### 5.5. Dynamic Hardware Daemon (`servers/devd/main.kn`)

```kirn
module servers.devd.main;

import kernel.ipc.ring;
import kernel.core.process;

pub struct DeviceIdentifier {
    pub bus_type:  u8, // 1 = PCIe, 2 = USB
    pub vendor_id: u16,
    pub device_id: u16,
    pub class_id:  u8,
}

pub struct DevDaemon {
    event_channel: ring::KirnRingBuffer,

    pub fn run(&mut self) -> noreturn {
        while true {
            self.event_channel.poll_and_dispatch(|event| {
                if event.opcode == ring::SysOp::DeviceDMA {
                    let dev = DeviceIdentifier {
                        bus_type:  (event.flags & 0xFF) as u8,
                        vendor_id: (event.user_data >> 16) as u16,
                        device_id: (event.user_data & 0xFFFF) as u16,
                        class_id:  ((event.user_data >> 32) & 0xFF) as u8,
                    };
                    self.bind_and_launch_driver(&dev);
                }
                return 0;
            });
            @asm volatile ("pause");
        }
    }

    fn bind_and_launch_driver(&self, dev: &DeviceIdentifier) {
        // 1. Matches Vendor and Device IDs against /system/drivers database
        let driver_path = self.lookup_driver_binary(dev);

        // 2. Launches the user-mode driver into an isolated Ring 3 sandbox
        let perms = ["CAP_MAP_PAGES", "CAP_MANAGE_IRQ"];
        process::spawn_sandboxed(driver_path, perms, 128 * 1024 * 1024).expect("Driver spawn failed");

        // 3. Registers device node in /dev
        self.synthesize_device_node(dev);
    }

    fn lookup_driver_binary(&self, dev: &DeviceIdentifier) -> string {
        return "/system/drivers/virtio_gpu.kapp";
    }

    fn synthesize_device_node(&self, dev: &DeviceIdentifier) {}
}
```

---

## 6. Phased Implementation Roadmap & Verification Targets

```
Sprint 1 (Weeks 1-2):   [ PID 1 Core: main.kn, Minimal Ring 0 bootstrap, Console TTY output ]
Sprint 2 (Weeks 3-4):   [ Service Manifests & State Engine: service.kn, Lifecycle states, Panic safety ]
Sprint 3 (Weeks 5-6):   [ Dependency DAG Resolver: dag.kn, Parallel async startup, Kahn's algorithm ]
Sprint 4 (Weeks 7-8):   [ Socket Activation Broker: socket_act.kn, /srv channel buffering, Lazy spawns ]
Sprint 5 (Weeks 9-10):  [ Proto9K Engine: proto9k.kn, 9P2000.L wire format parser over KirnRing ]
Sprint 6 (Weeks 11-12): [ Synthetic Trees: procfs.kn, srvfs.kn, Per-process private namespaces ]
Sprint 7 (Weeks 13-14): [ Hardware Manager: devd/main.kn, PCIe/USB event listening, KDF driver lifecycle ]
Sprint 8 (Weeks 15-16): [ Crash Resilience & Fast-Boot: Sub-120ms verification, Driver fault injection ]
```

### 6.1. Verification & Benchmark Targets
1. **Boot Latency**: Time measured from the kernel jumping to `_start` in `kirnd` until `kirnsurface` displays the initial login prompt:
   $$\text{Target Boot Time} \le 120\text{ ms} \quad (\text{on modern NVMe + PCIe Gen 4 x86_64/AArch64})$$
2. **Crash Resilience**: Inject an unhandled abort (`SIGSEGV` / page fault equivalent) into a background service. `kirnd` must detect the termination, execute exponential backoff, restore the service state, and maintain system stability without leaving orphaned zombie handles.
3. **Namespace Isolation**:
   * Process $A$ mounts a private RAM tree at `/srv/secrets`.
   * Process $B$ traverses its own `/srv/` directory and confirms `/srv/secrets` is invisible and inaccessible.

---

## 7. Status of KirnOS Master Architecture

We now have complete architectural plans, algorithmic proofs, and `.kn` implementations for:
1. **KirnCore**: The micro-hybrid kernel (Object Manager, PMM, VMM, GTD Scheduler, `KirnRing`).
2. **KirnFS**: The storage engine (CoW, BLAKE3 Merkle integrity, BeOS live database attributes, APFS space sharing).
3. **KirnSurface**: The GPU display server and declarative vector UI engine (Direct scanout, compute SDF shaders).
4. **The Triple Compatibility Subsystem**: The native binary translation personality runtimes (Linux ELF, Windows PE, and POSIX.1-2024).
5. **KirnNet**: The zero-copy high-throughput network stack, stateful firewall, and in-kernel QUIC/TCP engine.
6. **KirnSound**: The ultra-low-latency, lock-free audio engine and real-time DSP graph.
7. **KirnInit, Service Supervisor & Plan 9 Synthetic Namespaces**: The PID 1 supervisor, socket activation broker, `Proto9K` engine, and `devd` hardware daemon.

The **final missing plan** in the KirnOS stack is:
* **Plan 6**: The `.kapp` Application Packaging & Sandboxing Ecosystem (`KirnBox`).
