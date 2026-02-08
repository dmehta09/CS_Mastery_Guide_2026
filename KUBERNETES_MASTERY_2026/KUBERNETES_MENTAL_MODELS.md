# Kubernetes Mental Models
## Intuitive Frameworks for Deep Understanding

---

## The Big Picture: What is Kubernetes?

### Mental Model 1: The City Analogy

```
KUBERNETES = A SMART CITY MANAGEMENT SYSTEM

┌─────────────────────────────────────────────────────────────────┐
│                        KUBERNETES CLUSTER                        │
│                     (The Entire City)                           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              CONTROL PLANE (City Hall)                   │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐  │   │
│  │  │API Server│ │Scheduler │ │Controller│ │   etcd     │  │   │
│  │  │(Reception│ │(Urban    │ │Manager   │ │(City       │  │   │
│  │  │  Desk)   │ │Planner)  │ │(Mayor)   │ │ Records)   │  │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              WORKER NODES (City Districts)               │   │
│  │                                                          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │   │
│  │  │   Node 1    │  │   Node 2    │  │   Node 3    │      │   │
│  │  │ (District A)│  │ (District B)│  │ (District C)│      │   │
│  │  │             │  │             │  │             │      │   │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │      │   │
│  │  │ │  Pods   │ │  │ │  Pods   │ │  │ │  Pods   │ │      │   │
│  │  │ │(Bldngs) │ │  │ │(Bldngs) │ │  │ │(Bldngs) │ │      │   │
│  │  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │      │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

City Hall (Control Plane):
- Reception Desk (API Server) → All requests come here first
- Urban Planner (Scheduler) → Decides where to build new buildings
- Mayor (Controller Manager) → Ensures city runs as planned
- City Records (etcd) → Stores all city information

Districts (Nodes):
- Buildings (Pods) → Where workers (containers) do their job
- District Manager (Kubelet) → Reports to City Hall, manages buildings
- Traffic Controller (Kube-Proxy) → Routes visitors to right buildings
```

---

## Control Plane Mental Models

### Mental Model 2: The API Server as a Gatekeeper

```
                    ALL COMMUNICATION FLOWS THROUGH API SERVER

Users/Kubectl ──┐
                │
CI/CD Systems ──┼──→ [API SERVER] ──→ etcd (Storage)
                │         │
Other Services ─┘         ↓
                    ┌─────────────┐
                    │ Scheduler   │
                    │ Controllers │
                    │ Kubelet     │
                    └─────────────┘

THINK OF IT AS:
┌────────────────────────────────────────────────────────────┐
│  API Server = Airport Security + Air Traffic Control       │
│                                                            │
│  1. Authentication → "Who are you?" (Show ID)              │
│  2. Authorization  → "Can you do this?" (Check permissions)│
│  3. Admission      → "Is this allowed?" (Policy check)     │
│  4. Validation     → "Is this correct?" (Format check)     │
│  5. Persistence    → "Store it" (Write to etcd)            │
└────────────────────────────────────────────────────────────┘
```

### Mental Model 3: etcd as the Single Source of Truth

```
etcd = THE BRAIN'S MEMORY

┌──────────────────────────────────────────────────────────────┐
│                         etcd                                  │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  DESIRED STATE (What you WANT)                       │    │
│  │  "I want 3 replicas of nginx running"                │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  ACTUAL STATE (What you HAVE)                        │    │
│  │  "Currently 2 replicas are running"                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Controllers constantly compare and reconcile these states   │
└──────────────────────────────────────────────────────────────┘

KEY INSIGHT:
- etcd only stores DECLARATIVE state
- It doesn't run containers
- It's the "memory" that controllers read from
- Lose etcd = Lose your cluster's memory
```

### Mental Model 4: Scheduler as a Matchmaker

```
SCHEDULER = DATING APP FOR PODS AND NODES

Pod Looking for a Home:
┌─────────────────────────┐
│ Pod Requirements:       │
│ - 2 CPU                 │
│ - 4GB RAM               │
│ - SSD storage           │
│ - Not on Node with      │
│   same app (anti-affin) │
└─────────────────────────┘
           │
           ↓
    ┌──────────────┐
    │  SCHEDULER   │
    │  (Matchmaker)│
    └──────────────┘
           │
    ┌──────┴──────┐
    ↓             ↓
┌────────┐  ┌────────┐  ┌────────┐
│ Node 1 │  │ Node 2 │  │ Node 3 │
│ 4 CPU  │  │ 8 CPU  │  │ 2 CPU  │
│ 8GB RAM│  │ 16GB   │  │ 4GB RAM│
│ HDD    │  │ SSD    │  │ SSD    │
│        │  │        │  │        │
│ Score: │  │ Score: │  │ Score: │
│   45   │  │   92   │  │   30   │
└────────┘  └────────┘  └────────┘
     ✗      ↑    ✓           ✗
            │
      WINNER! Best match

SCHEDULING PROCESS:
1. FILTERING  → Eliminate nodes that don't meet requirements
2. SCORING    → Rank remaining nodes by preference
3. BINDING    → Assign pod to highest-scoring node
```

### Mental Model 5: Controller Manager as Autopilot

```
CONTROLLER MANAGER = THERMOSTAT SYSTEM

┌────────────────────────────────────────────────────────────┐
│                   RECONCILIATION LOOP                       │
│                                                            │
│      ┌──────────┐         ┌──────────┐                    │
│      │ DESIRED  │         │  ACTUAL  │                    │
│      │  STATE   │         │  STATE   │                    │
│      │ (3 pods) │         │ (2 pods) │                    │
│      └────┬─────┘         └────┬─────┘                    │
│           │                     │                          │
│           └──────────┬──────────┘                          │
│                      ↓                                     │
│              ┌──────────────┐                              │
│              │   COMPARE    │                              │
│              │   (Diff)     │                              │
│              └──────┬───────┘                              │
│                     ↓                                      │
│              ┌──────────────┐                              │
│              │    ACTION    │                              │
│              │ (Create 1    │                              │
│              │  more pod)   │                              │
│              └──────────────┘                              │
│                                                            │
│   This loop runs CONTINUOUSLY (every few seconds)          │
└────────────────────────────────────────────────────────────┘

LIKE A THERMOSTAT:
- Set temperature to 72°F (Desired State)
- Room is 68°F (Actual State)
- Thermostat turns on heat (Action)
- Loop repeats until 72°F reached
```

---

## Workload Mental Models

### Mental Model 6: Pod as an Apartment

```
POD = APARTMENT IN A BUILDING

┌─────────────────────────────────────────────────────────┐
│                         POD                              │
│                    (The Apartment)                       │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Container 1 │  │ Container 2 │  │ Container 3 │     │
│  │   (App)     │  │  (Sidecar)  │  │  (Logger)   │     │
│  │             │  │             │  │             │     │
│  │  Bedroom    │  │  Bathroom   │  │   Kitchen   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                         │
│  SHARED RESOURCES (Shared by all roommates):            │
│  ┌────────────────────────────────────────────────┐    │
│  │  - Network (Same IP address - like same WiFi)   │    │
│  │  - Storage (Shared volumes - like shared fridge)│    │
│  │  - IPC (Can talk to each other via localhost)   │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  Pod IP: 10.1.2.3 (Apartment Address)                  │
└─────────────────────────────────────────────────────────┘

KEY INSIGHTS:
- Containers in same pod = Roommates (tightly coupled)
- Different pods = Different apartments (loosely coupled)
- Pod is the SMALLEST deployable unit
- Pod is EPHEMERAL (can be evicted/replaced anytime)
```

### Mental Model 7: Workload Hierarchy

```
WORKLOAD TYPES = MANAGEMENT HIERARCHY

                    ┌─────────────────┐
                    │   DEPLOYMENT    │  ← Manager (controls desired state)
                    │  "I want 3      │
                    │   workers"      │
                    └────────┬────────┘
                             │ manages
                             ↓
                    ┌─────────────────┐
                    │   REPLICASET    │  ← Supervisor (maintains count)
                    │  "I ensure 3    │
                    │   exist"        │
                    └────────┬────────┘
                             │ creates/deletes
                             ↓
              ┌──────────────┼──────────────┐
              ↓              ↓              ↓
         ┌────────┐    ┌────────┐    ┌────────┐
         │  POD   │    │  POD   │    │  POD   │  ← Workers
         │  #1    │    │  #2    │    │  #3    │
         └────────┘    └────────┘    └────────┘


WORKLOAD TYPE DECISION TREE:

Is your app stateless?
├── YES → Use DEPLOYMENT
│         (Web servers, APIs, microservices)
│
└── NO → Does each instance need unique identity?
         ├── YES → Use STATEFULSET
         │         (Databases, Kafka, Zookeeper)
         │
         └── NO → Should it run on EVERY node?
                  ├── YES → Use DAEMONSET
                  │         (Log collectors, monitoring agents)
                  │
                  └── NO → Is it a one-time task?
                           ├── YES → Use JOB
                           │         (Batch processing, migrations)
                           │
                           └── NO → Is it scheduled?
                                    └── YES → Use CRONJOB
                                              (Backups, reports)
```

### Mental Model 8: StatefulSet vs Deployment

```
DEPLOYMENT = CATTLE (Interchangeable)
┌─────────────────────────────────────────────────────────┐
│  Pod-abc123    Pod-def456    Pod-ghi789                 │
│      │              │              │                    │
│      └──────────────┼──────────────┘                    │
│                     ↓                                   │
│            All are IDENTICAL                            │
│            Any can be replaced                          │
│            No guaranteed order                          │
│            Random names                                 │
│            Shared storage (if any)                      │
└─────────────────────────────────────────────────────────┘

STATEFULSET = PETS (Unique Identity)
┌─────────────────────────────────────────────────────────┐
│  mysql-0         mysql-1         mysql-2                │
│     │                │               │                  │
│     ↓                ↓               ↓                  │
│  ┌──────┐        ┌──────┐       ┌──────┐               │
│  │ PVC-0│        │ PVC-1│       │ PVC-2│               │
│  │(10GB)│        │(10GB)│       │(10GB)│               │
│  └──────┘        └──────┘       └──────┘               │
│                                                         │
│  Each has:                                              │
│  - Stable hostname (mysql-0, mysql-1, mysql-2)         │
│  - Ordered deployment (0 → 1 → 2)                      │
│  - Ordered deletion (2 → 1 → 0)                        │
│  - Persistent storage per pod                          │
│  - Stable network identity                             │
└─────────────────────────────────────────────────────────┘
```

---

## Networking Mental Models

### Mental Model 9: Service as a Load Balancer/Receptionist

```
SERVICE = COMPANY RECEPTIONIST

                     External User
                          │
                          ↓
               ┌──────────────────┐
               │     SERVICE      │
               │  (Receptionist)  │
               │                  │
               │  "How may I      │
               │   direct your    │
               │   call?"         │
               │                  │
               │  ClusterIP:      │
               │  10.96.0.1:80    │
               └────────┬─────────┘
                        │
           ┌────────────┼────────────┐
           ↓            ↓            ↓
      ┌────────┐   ┌────────┐   ┌────────┐
      │ Pod 1  │   │ Pod 2  │   │ Pod 3  │
      │10.1.1.1│   │10.1.1.2│   │10.1.1.3│
      └────────┘   └────────┘   └────────┘

KEY INSIGHT:
- Pods come and go (ephemeral)
- Service provides STABLE endpoint
- Service discovers pods via LABELS (selector)
- Think: "Don't call employees directly, call reception"


SERVICE TYPES MENTAL MODEL:

┌─────────────────────────────────────────────────────────────┐
│                    SERVICE TYPES                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ClusterIP (Default) - Internal phone extension             │
│  ┌─────────────────────────────────────────┐               │
│  │  Only accessible INSIDE the cluster     │               │
│  │  Like: Extension 5001 (internal only)   │               │
│  └─────────────────────────────────────────┘               │
│                                                             │
│  NodePort - Direct line through any office                  │
│  ┌─────────────────────────────────────────┐               │
│  │  Accessible via NodeIP:NodePort         │               │
│  │  Like: Call any branch, ask for ext     │               │
│  │  Port range: 30000-32767                │               │
│  └─────────────────────────────────────────┘               │
│                                                             │
│  LoadBalancer - Main company phone number                   │
│  ┌─────────────────────────────────────────┐               │
│  │  External IP from cloud provider        │               │
│  │  Like: 1-800-COMPANY (public number)    │               │
│  └─────────────────────────────────────────┘               │
│                                                             │
│  ExternalName - Call forwarding                             │
│  ┌─────────────────────────────────────────┐               │
│  │  CNAME to external DNS                  │               │
│  │  Like: "Call Bob" → Redirects to cell   │               │
│  └─────────────────────────────────────────┘               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Mental Model 10: Ingress as a Front Door

```
INGRESS = HOTEL FRONT DESK / RECEPTIONIST

                        Internet
                           │
                           ↓
              ┌────────────────────────┐
              │        INGRESS         │
              │    (Front Desk)        │
              │                        │
              │  "Welcome! Where       │
              │   would you like       │
              │   to go?"              │
              └───────────┬────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ↓                 ↓                 ↓
   /api/*            /web/*            /admin/*
        │                 │                 │
        ↓                 ↓                 ↓
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ API      │     │ Frontend │     │ Admin    │
  │ Service  │     │ Service  │     │ Service  │
  └──────────┘     └──────────┘     └──────────┘


INGRESS ROUTING RULES:

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  shop.example.com     → frontend-service                    │
│  api.example.com      → api-service                         │
│  shop.example.com/api → api-service                         │
│  shop.example.com/img → cdn-service                         │
│                                                             │
│  PLUS:                                                      │
│  - SSL/TLS termination (HTTPS → HTTP)                      │
│  - Rate limiting                                            │
│  - Authentication                                           │
│  - Rewrite rules                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Mental Model 11: Network Policies as Firewalls

```
NETWORK POLICY = APARTMENT BUILDING SECURITY

DEFAULT: No Network Policies = Open Door Policy
┌─────────────────────────────────────────────────────────────┐
│  All pods can talk to all pods (no restrictions)            │
│  Like: Anyone can visit any apartment                       │
└─────────────────────────────────────────────────────────────┘

WITH NETWORK POLICIES: Gated Community
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   ┌─────────────┐                    ┌─────────────┐       │
│   │  Frontend   │ ───── ALLOW ─────→ │   Backend   │       │
│   │    Pods     │                    │    Pods     │       │
│   └─────────────┘                    └──────┬──────┘       │
│          │                                  │               │
│          │                                  │ ALLOW         │
│       DENY                                  ↓               │
│          │                           ┌─────────────┐       │
│          └─────────── DENY ─────────→│  Database   │       │
│                                      │    Pods     │       │
│                                      └─────────────┘       │
│                                                             │
│   Rule: Only backend can talk to database                   │
│   Like: Only staff can access server room                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

NETWORK POLICY MENTAL MODEL:

┌────────────────────────────────────────────────────────────┐
│  NetworkPolicy = "Who can come IN" + "Where can I go OUT"  │
│                                                            │
│  podSelector: Which pods this policy applies to            │
│  ingress:     Who can send traffic TO these pods          │
│  egress:      Where these pods can send traffic TO        │
│                                                            │
│  No policy = Allow all                                     │
│  Empty policy = Deny all                                   │
│  Specific rules = Allow only matching traffic              │
└────────────────────────────────────────────────────────────┘
```

---

## Storage Mental Models

### Mental Model 12: Storage Hierarchy

```
STORAGE = FILING SYSTEM

┌─────────────────────────────────────────────────────────────┐
│                    STORAGE HIERARCHY                         │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              STORAGE CLASS                           │   │
│  │          (Type of Filing Cabinet)                    │   │
│  │                                                      │   │
│  │  "fast-ssd"  "standard"  "archive"                  │   │
│  │  (Premium)   (Standard)  (Cheap/Slow)               │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ↓                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           PERSISTENT VOLUME (PV)                     │   │
│  │         (Actual Filing Cabinet)                      │   │
│  │                                                      │   │
│  │  - 100GB SSD on AWS EBS                             │   │
│  │  - Pre-provisioned or dynamically created           │   │
│  │  - Cluster-level resource                           │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ↓                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │        PERSISTENT VOLUME CLAIM (PVC)                 │   │
│  │            (Request for Cabinet)                     │   │
│  │                                                      │   │
│  │  "I need 50GB of fast-ssd storage"                  │   │
│  │  - Namespace-level resource                         │   │
│  │  - Binds to matching PV                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ↓                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    POD                               │   │
│  │           (Person using cabinet)                     │   │
│  │                                                      │   │
│  │  volumes:                                            │   │
│  │    - name: data                                      │   │
│  │      persistentVolumeClaim:                          │   │
│  │        claimName: my-pvc                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘


ANALOGY:
StorageClass = "I want IKEA furniture" (brand/type)
PV = Actual bookshelf from IKEA (the thing)
PVC = "I need a bookshelf, 6 feet tall" (the request)
Volume Mount = Put bookshelf in my room (use it)
```

### Mental Model 13: Volume Lifecycle

```
VOLUME ACCESS MODES:

┌────────────────────────────────────────────────────────────┐
│                                                            │
│  ReadWriteOnce (RWO) - Personal Diary                      │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Only ONE pod can read/write at a time               │ │
│  │  Like: Only I can write in my diary                  │ │
│  │  Use for: Databases, single-instance apps            │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
│  ReadOnlyMany (ROX) - Public Library Book                  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  MANY pods can read, but none can write              │ │
│  │  Like: Everyone can read library book, none edit     │ │
│  │  Use for: Shared config, static content              │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
│  ReadWriteMany (RWX) - Shared Google Doc                   │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  MANY pods can read AND write simultaneously         │ │
│  │  Like: Team editing same Google Doc                  │ │
│  │  Use for: Shared file storage, CMS uploads           │ │
│  │  Note: Not all storage backends support this!        │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Security Mental Models

### Mental Model 14: RBAC as a Permission System

```
RBAC = CORPORATE ACCESS CARD SYSTEM

┌─────────────────────────────────────────────────────────────┐
│                         RBAC                                 │
│                                                             │
│  WHO (Subjects)           WHAT (Verbs)      ON (Resources)  │
│  ─────────────            ──────────        ─────────────   │
│  - Users                  - get             - pods          │
│  - Groups                 - list            - services      │
│  - ServiceAccounts        - create          - secrets       │
│                           - update          - configmaps    │
│                           - delete          - deployments   │
│                           - watch                           │
│                                                             │
│                                                             │
│      ┌──────────┐         ┌──────────┐                     │
│      │   USER   │         │   ROLE   │                     │
│      │  "Alice" │◄────────│ "viewer" │                     │
│      └──────────┘  Bound  └──────────┘                     │
│                     via         │                           │
│              RoleBinding        │                           │
│                                 ↓                           │
│                           ┌──────────────┐                 │
│                           │ Permissions: │                 │
│                           │ - get pods   │                 │
│                           │ - list pods  │                 │
│                           │ - watch pods │                 │
│                           └──────────────┘                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘


ROLE vs CLUSTERROLE:

┌────────────────────────────────────────────────────────────┐
│                                                            │
│  ROLE + ROLEBINDING = Namespace-scoped                     │
│  ┌───────────────────────────────────────────────────────┐│
│  │  Like: Access card for ONE floor of building          ││
│  │  "Alice can manage pods in 'production' namespace"    ││
│  └───────────────────────────────────────────────────────┘│
│                                                            │
│  CLUSTERROLE + CLUSTERROLEBINDING = Cluster-wide          │
│  ┌───────────────────────────────────────────────────────┐│
│  │  Like: Master key for entire building                 ││
│  │  "Bob can manage pods in ALL namespaces"              ││
│  └───────────────────────────────────────────────────────┘│
│                                                            │
│  CLUSTERROLE + ROLEBINDING = Reusable template            │
│  ┌───────────────────────────────────────────────────────┐│
│  │  Like: Standard role applied to specific floor        ││
│  │  "Use 'viewer' ClusterRole in 'dev' namespace"        ││
│  └───────────────────────────────────────────────────────┘│
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Mental Model 15: Security Layers

```
DEFENSE IN DEPTH = CASTLE SECURITY

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  LAYER 1: CLUSTER ACCESS (Outer Wall)                       │
│  ┌────────────────────────────────────────────────────────┐│
│  │  - API Server Authentication (who are you?)            ││
│  │  - API Server Authorization (can you enter?)           ││
│  │  - Admission Controllers (bag check)                   ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  LAYER 2: NAMESPACE ISOLATION (Inner Walls)                 │
│  ┌────────────────────────────────────────────────────────┐│
│  │  - Namespace RBAC (floor-specific access)              ││
│  │  - ResourceQuotas (resource limits per floor)          ││
│  │  - NetworkPolicies (who can visit whom)                ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  LAYER 3: POD SECURITY (Room Security)                      │
│  ┌────────────────────────────────────────────────────────┐│
│  │  - Pod Security Standards (room rules)                 ││
│  │  - SecurityContext (what guest can do in room)         ││
│  │  - Service Accounts (guest identity)                   ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  LAYER 4: CONTAINER SECURITY (Safe Box)                     │
│  ┌────────────────────────────────────────────────────────┐│
│  │  - Read-only root filesystem                           ││
│  │  - Non-root user                                       ││
│  │  - Dropped capabilities                                ││
│  │  - Seccomp/AppArmor profiles                           ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  LAYER 5: RUNTIME SECURITY (Surveillance)                   │
│  ┌────────────────────────────────────────────────────────┐│
│  │  - Falco (real-time threat detection)                  ││
│  │  - Audit logging (record everything)                   ││
│  │  - Image scanning (check packages for exploits)        ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Scheduling Mental Models

### Mental Model 16: Pod Placement Controls

```
POD SCHEDULING = SEATING ARRANGEMENT

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  NODE SELECTOR (Simple preference)                          │
│  ┌────────────────────────────────────────────────────────┐│
│  │  "Seat me at a table with GPU"                         ││
│  │                                                        ││
│  │  nodeSelector:                                         ││
│  │    accelerator: nvidia-tesla                           ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  NODE AFFINITY (Advanced preference)                        │
│  ┌────────────────────────────────────────────────────────┐│
│  │  "I MUST sit in zone-a" (requiredDuringScheduling)     ││
│  │  "I PREFER SSD nodes" (preferredDuringScheduling)      ││
│  │                                                        ││
│  │  Like: "Must be window seat, prefer aisle if possible" ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  POD AFFINITY (Sit together)                                │
│  ┌────────────────────────────────────────────────────────┐│
│  │  "Seat me NEAR pods with label app=cache"              ││
│  │                                                        ││
│  │  Like: "I want to sit near my teammates"               ││
│  │  Use case: App pods near cache pods (low latency)      ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  POD ANTI-AFFINITY (Sit apart)                              │
│  ┌────────────────────────────────────────────────────────┐│
│  │  "Don't seat me with other app=web pods"               ││
│  │                                                        ││
│  │  Like: "Don't put all eggs in one basket"              ││
│  │  Use case: Spread replicas across nodes/zones          ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  TAINTS & TOLERATIONS (VIP Section)                         │
│  ┌────────────────────────────────────────────────────────┐│
│  │  Node Taint: "This is GPU-only section"                ││
│  │  Pod Toleration: "I'm allowed in GPU section"          ││
│  │                                                        ││
│  │  Like: VIP area needs special pass to enter            ││
│  │  Use case: Dedicated nodes, special hardware           ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘


TOPOLOGY SPREAD CONSTRAINTS:

┌────────────────────────────────────────────────────────────┐
│                                                            │
│  "Spread my pods EVENLY across zones"                      │
│                                                            │
│      Zone A          Zone B          Zone C                │
│    ┌────────┐      ┌────────┐      ┌────────┐             │
│    │ Pod 1  │      │ Pod 2  │      │ Pod 3  │             │
│    │ Pod 4  │      │ Pod 5  │      │ Pod 6  │             │
│    └────────┘      └────────┘      └────────┘             │
│                                                            │
│  maxSkew: 1 (max difference between zones)                 │
│  If Zone A has 3 pods, Zone B can have 2-4 pods           │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Resource Management Mental Models

### Mental Model 17: Requests vs Limits

```
RESOURCES = OFFICE SPACE ALLOCATION

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  REQUESTS = Guaranteed Reservation (Minimum)                │
│  ┌────────────────────────────────────────────────────────┐│
│  │  "I need at LEAST this much to function"               ││
│  │                                                        ││
│  │  resources:                                            ││
│  │    requests:                                           ││
│  │      cpu: 500m      ← Reserved 0.5 CPU cores          ││
│  │      memory: 256Mi  ← Reserved 256 MB RAM             ││
│  │                                                        ││
│  │  Scheduler uses this to place pods                     ││
│  │  Like: "I reserved a desk in the office"               ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  LIMITS = Maximum Allowed (Ceiling)                         │
│  ┌────────────────────────────────────────────────────────┐│
│  │  "Don't let me use more than this"                     ││
│  │                                                        ││
│  │  resources:                                            ││
│  │    limits:                                             ││
│  │      cpu: 1000m     ← Max 1 CPU core (throttled)      ││
│  │      memory: 512Mi  ← Max 512 MB (OOMKilled if exceed)││
│  │                                                        ││
│  │  Like: "I can use conference room but only for 1 hour" ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘


QoS CLASSES (Quality of Service):

┌────────────────────────────────────────────────────────────┐
│                                                            │
│  GUARANTEED (First Class)                                  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  requests == limits (for both CPU and memory)        │ │
│  │  LAST to be evicted when node is under pressure      │ │
│  │  Like: Reserved first-class seat                     │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
│  BURSTABLE (Business Class)                                │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  requests < limits OR only one specified             │ │
│  │  Evicted SECOND when node is under pressure          │ │
│  │  Like: Can upgrade if space available                │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
│  BESTEFFORT (Economy Class)                                │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  No requests or limits specified                     │ │
│  │  FIRST to be evicted when node is under pressure     │ │
│  │  Like: Standby passenger, bumped first               │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Observability Mental Models

### Mental Model 18: Health Probes

```
PROBES = HEALTH CHECKUPS

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  STARTUP PROBE (Birth Certificate)                          │
│  ┌────────────────────────────────────────────────────────┐│
│  │  "Has the application started successfully?"           ││
│  │                                                        ││
│  │  - Runs FIRST, before other probes                     ││
│  │  - Allows slow-starting apps time to initialize        ││
│  │  - Fails? Container is restarted                       ││
│  │  - Like: "Is the restaurant open yet?"                 ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  LIVENESS PROBE (Pulse Check)                               │
│  ┌────────────────────────────────────────────────────────┐│
│  │  "Is the application still alive?"                     ││
│  │                                                        ││
│  │  - Runs continuously after startup                     ││
│  │  - Detects deadlocks, infinite loops                   ││
│  │  - Fails? Container is RESTARTED                       ││
│  │  - Like: "Is the patient breathing?"                   ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  READINESS PROBE (Ready for Work)                           │
│  ┌────────────────────────────────────────────────────────┐│
│  │  "Is the application ready to receive traffic?"        ││
│  │                                                        ││
│  │  - Runs continuously                                   ││
│  │  - Checks dependencies (DB connected, cache warm)      ││
│  │  - Fails? Pod removed from Service endpoints           ││
│  │  - Like: "Is the cashier ready to serve customers?"    ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘


PROBE TYPES:

┌────────────────────────────────────────────────────────────┐
│                                                            │
│  HTTP GET                                                  │
│  - Hits an HTTP endpoint                                   │
│  - 200-399 = Success                                       │
│  - Like: Checking if website loads                         │
│                                                            │
│  TCP Socket                                                │
│  - Opens TCP connection to port                            │
│  - Connection success = Healthy                            │
│  - Like: Can I dial this phone number?                     │
│                                                            │
│  Exec Command                                              │
│  - Runs command in container                               │
│  - Exit code 0 = Healthy                                   │
│  - Like: Running diagnostic script                         │
│                                                            │
│  gRPC                                                      │
│  - Calls gRPC health check endpoint                        │
│  - For gRPC-based services                                 │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## GitOps Mental Models

### Mental Model 19: GitOps Flow

```
GITOPS = GIT AS SINGLE SOURCE OF TRUTH

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   TRADITIONAL (Push-based)                                  │
│   ┌───────────────────────────────────────────────────────┐│
│   │                                                       ││
│   │  Developer → CI Pipeline → PUSH → Kubernetes          ││
│   │                                                       ││
│   │  Problem: Pipeline has cluster credentials            ││
│   │  Problem: Drift between Git and actual state          ││
│   │                                                       ││
│   └───────────────────────────────────────────────────────┘│
│                                                             │
│   GITOPS (Pull-based)                                       │
│   ┌───────────────────────────────────────────────────────┐│
│   │                                                       ││
│   │  Developer → Git Repo ← PULL ← ArgoCD → Kubernetes    ││
│   │                                                       ││
│   │  ┌─────────┐     ┌─────────┐     ┌─────────────┐     ││
│   │  │   Git   │────→│ ArgoCD  │────→│ Kubernetes  │     ││
│   │  │  Repo   │     │(in-clstr│     │   Cluster   │     ││
│   │  │(desired)│←────│ agent)  │←────│  (actual)   │     ││
│   │  └─────────┘     └─────────┘     └─────────────┘     ││
│   │       │               │                │              ││
│   │       └───────────────┴────────────────┘              ││
│   │              CONTINUOUS RECONCILIATION                ││
│   │                                                       ││
│   │  Benefit: Only ArgoCD needs cluster access            ││
│   │  Benefit: Git history = deployment history            ││
│   │  Benefit: Automatic drift correction                  ││
│   │                                                       ││
│   └───────────────────────────────────────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Operator Mental Models

### Mental Model 20: Operators as Domain Experts

```
OPERATOR = AUTOMATED HUMAN EXPERT

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  WITHOUT OPERATOR (Manual DBA)                              │
│  ┌────────────────────────────────────────────────────────┐│
│  │                                                        ││
│  │  Human DBA manually:                                   ││
│  │  - Creates database pods                               ││
│  │  - Configures replication                              ││
│  │  - Takes backups                                       ││
│  │  - Handles failover                                    ││
│  │  - Scales cluster                                      ││
│  │  - Upgrades versions                                   ││
│  │                                                        ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  WITH OPERATOR (Automated DBA)                              │
│  ┌────────────────────────────────────────────────────────┐│
│  │                                                        ││
│  │  ┌───────────────┐                                     ││
│  │  │ PostgresCluster│  ← Custom Resource (your request) ││
│  │  │ replicas: 3    │                                    ││
│  │  │ version: 15    │                                    ││
│  │  │ backup: daily  │                                    ││
│  │  └───────┬───────┘                                     ││
│  │          │                                             ││
│  │          ↓                                             ││
│  │  ┌───────────────┐                                     ││
│  │  │   OPERATOR    │  ← Watches and acts                ││
│  │  │   (Auto-DBA)  │                                     ││
│  │  └───────┬───────┘                                     ││
│  │          │                                             ││
│  │          ↓ Creates/Manages                             ││
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌────────┐ ┌─────────┐      ││
│  │  │Pod 1│ │Pod 2│ │Pod 3│ │Services│ │Secrets  │      ││
│  │  └─────┘ └─────┘ └─────┘ └────────┘ └─────────┘      ││
│  │  ┌─────┐ ┌──────────┐ ┌───────────────────┐          ││
│  │  │PVCs │ │ConfigMaps│ │Backup CronJobs    │          ││
│  │  └─────┘ └──────────┘ └───────────────────┘          ││
│  │                                                        ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  OPERATOR = CRD + Controller                                │
│  - CRD: Define what you want (PostgresCluster spec)        │
│  - Controller: Makes it happen (reconciliation loop)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting Mental Models

### Mental Model 21: Debugging Flow

```
DEBUGGING = DETECTIVE INVESTIGATION

┌─────────────────────────────────────────────────────────────┐
│                    TROUBLESHOOTING FLOW                      │
│                                                             │
│  STEP 1: What's the symptom?                                │
│  ┌────────────────────────────────────────────────────────┐│
│  │  □ Pod not starting?                                   ││
│  │  □ Pod crashing (CrashLoopBackOff)?                    ││
│  │  □ Pod running but not accessible?                     ││
│  │  □ Service not routing traffic?                        ││
│  │  □ Ingress not working?                                ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  STEP 2: Check the evidence                                 │
│  ┌────────────────────────────────────────────────────────┐│
│  │                                                        ││
│  │  kubectl get pods -o wide                              ││
│  │  → Pod status, node placement, restarts                ││
│  │                                                        ││
│  │  kubectl describe pod <name>                           ││
│  │  → Events, conditions, container statuses              ││
│  │                                                        ││
│  │  kubectl logs <pod> [-c container] [--previous]        ││
│  │  → Application logs, crash logs                        ││
│  │                                                        ││
│  │  kubectl get events --sort-by='.lastTimestamp'         ││
│  │  → Cluster-wide events                                 ││
│  │                                                        ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘


POD STATUS DECODER:

┌────────────────────────────────────────────────────────────┐
│                                                            │
│  Pending        → Waiting for scheduling                   │
│                   Check: Resources, node selectors, taints │
│                                                            │
│  ContainerCreating → Pulling image, mounting volumes       │
│                      Check: Image name, secrets, PVCs      │
│                                                            │
│  Running        → Container started                        │
│                   But may not be ready (check probes)      │
│                                                            │
│  CrashLoopBackOff → Container crashes repeatedly          │
│                      Check: Logs, command, dependencies    │
│                                                            │
│  ImagePullBackOff → Can't pull container image            │
│                      Check: Image name, registry auth      │
│                                                            │
│  Evicted        → Node resource pressure                   │
│                   Check: Resource limits, node capacity    │
│                                                            │
│  OOMKilled      → Out of memory                           │
│                   Check: Memory limits, application leak   │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Key Principles Summary

### Mental Model 22: Core Kubernetes Principles

```
┌─────────────────────────────────────────────────────────────┐
│               KUBERNETES CORE PRINCIPLES                     │
│                                                             │
│  1. DECLARATIVE OVER IMPERATIVE                             │
│  ┌────────────────────────────────────────────────────────┐│
│  │  Say WHAT you want, not HOW to do it                   ││
│  │  "I want 3 nginx pods" vs "Create pod 1, create pod 2" ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  2. RECONCILIATION LOOPS                                    │
│  ┌────────────────────────────────────────────────────────┐│
│  │  Continuously compare desired vs actual state           ││
│  │  Take action to close the gap                          ││
│  │  Self-healing by design                                ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  3. LABELS & SELECTORS                                      │
│  ┌────────────────────────────────────────────────────────┐│
│  │  Everything is loosely coupled via labels              ││
│  │  Services find pods by labels                          ││
│  │  Deployments manage pods by labels                     ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  4. PODS ARE EPHEMERAL                                      │
│  ┌────────────────────────────────────────────────────────┐│
│  │  Pods can be killed anytime                            ││
│  │  Don't store state in pods                             ││
│  │  Use volumes for persistence                           ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  5. IMMUTABLE INFRASTRUCTURE                                │
│  ┌────────────────────────────────────────────────────────┐│
│  │  Don't patch running containers                        ││
│  │  Replace with new version (rolling update)             ││
│  │  Enables easy rollback                                 ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  6. API-FIRST DESIGN                                        │
│  ┌────────────────────────────────────────────────────────┐│
│  │  Everything is an API object                           ││
│  │  All operations via API server                         ││
│  │  Extensible via CRDs                                   ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
│  7. NAMESPACE ISOLATION                                     │
│  ┌────────────────────────────────────────────────────────┐│
│  │  Logical separation of resources                       ││
│  │  RBAC scoped to namespaces                             ││
│  │  Resource quotas per namespace                         ││
│  └────────────────────────────────────────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Reference: Decision Trees

### When to Use What Workload?

```
                        ┌─────────────────────┐
                        │  What type of app?  │
                        └──────────┬──────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ↓                    ↓                    ↓
        ┌──────────┐        ┌──────────┐        ┌──────────┐
        │ Stateless│        │ Stateful │        │  Batch   │
        │  Apps    │        │  Apps    │        │  Jobs    │
        └────┬─────┘        └────┬─────┘        └────┬─────┘
             │                   │                    │
             ↓                   ↓                    │
      ┌────────────┐     ┌─────────────┐             │
      │ DEPLOYMENT │     │ STATEFULSET │             │
      └────────────┘     └─────────────┘             │
                                               ┌─────┴─────┐
                                               │           │
                                               ↓           ↓
                                        ┌──────────┐ ┌──────────┐
                                        │   JOB    │ │ CRONJOB  │
                                        │(one-time)│ │(scheduled│
                                        └──────────┘ └──────────┘

        Need on EVERY node? → DAEMONSET
```

### Service Type Selection

```
        ┌─────────────────────────────────────┐
        │  Who needs to access this service?  │
        └─────────────────────┬───────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ↓                     ↓                     ↓
  ┌──────────┐         ┌──────────┐         ┌──────────┐
  │  Inside  │         │  Inside  │         │ Outside  │
  │ Cluster  │         │ Cluster  │         │ Cluster  │
  │  Only    │         │+ Testing │         │(Internet)│
  └────┬─────┘         └────┬─────┘         └────┬─────┘
       │                    │                     │
       ↓                    ↓                     │
  ┌──────────┐         ┌──────────┐              │
  │ClusterIP │         │ NodePort │              │
  │ (default)│         │          │              │
  └──────────┘         └──────────┘              │
                                           ┌─────┴─────┐
                                           │           │
                                           ↓           ↓
                                    ┌──────────┐ ┌──────────┐
                                    │LoadBalncr│ │ Ingress  │
                                    │(L4, $$$) │ │(L7,smart)│
                                    └──────────┘ └──────────┘
```

---

## Learning Path Recommendation

```
BEGINNER → INTERMEDIATE → ADVANCED → EXPERT

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  BEGINNER (Week 1-2)                                        │
│  □ Pods, Deployments, Services                              │
│  □ kubectl basics                                           │
│  □ Namespaces, Labels, Selectors                            │
│  □ ConfigMaps, Secrets                                      │
│                                                             │
│  INTERMEDIATE (Week 3-4)                                    │
│  □ Ingress, Network Policies                                │
│  □ Storage (PV, PVC, StorageClass)                          │
│  □ RBAC basics                                              │
│  □ Resource management (requests/limits)                    │
│  □ Probes (liveness, readiness)                             │
│                                                             │
│  ADVANCED (Week 5-6)                                        │
│  □ StatefulSets, DaemonSets                                 │
│  □ Scheduling (affinity, taints, tolerations)               │
│  □ Helm charts                                              │
│  □ HPA, VPA                                                 │
│  □ GitOps (ArgoCD/Flux)                                     │
│                                                             │
│  EXPERT (Week 7+)                                           │
│  □ Custom Resource Definitions (CRDs)                       │
│  □ Operators                                                │
│  □ Service Mesh (Istio/Linkerd)                             │
│  □ Multi-cluster management                                 │
│  □ Security hardening                                       │
│  □ Troubleshooting & performance tuning                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Final Mental Model: The Big Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  KUBERNETES = DECLARATIVE DESIRED STATE + RECONCILIATION LOOPS      │
│                                                                     │
│  You tell K8s: "I want X"                                           │
│  K8s ensures: "You have X"                                          │
│  If X breaks: "I'll fix X"                                          │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                             │   │
│  │   USER: "I declare I want 3 nginx pods behind a service"   │   │
│  │                           │                                 │   │
│  │                           ↓                                 │   │
│  │   KUBERNETES: "I will create and maintain 3 nginx pods     │   │
│  │               and a service, forever, until you tell me    │   │
│  │               otherwise."                                  │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  That's it. That's Kubernetes.                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
