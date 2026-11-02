## Definition of DRA (Dynamic Resource Allocation)

**Dynamic Resource Allocation (DRA)** is a Kubernetes feature that allows applications to request GPUs by specifying their exact hardware requirements—such as GPU model name and VRAM memory size—instead of just asking for a simple number like "1 GPU".

In traditional Kubernetes setups, GPUs were treated as plain numbers (`nvidia.com/gpu: 1`), with no visibility into a card's actual model or memory size. In contrast, DRA treats every physical GPU as an individual device with its own detailed profile (model name, VRAM capacity, PCIe address, and health status).

To manage devices cleanly, DRA divides the workflow into three steps:

1. **Hardware Inventory (`ResourceSlice`):** Worker nodes publish an exact list of the physical GPUs they have in stock.
2. **Workload Requests (`ResourceClaim`):** Pods specify the exact hardware they need using search filters written in Common Expression Language (CEL).
3. **Container Attachment (CDI):** Once Kubernetes matches a pod to a GPU, it uses the Container Device Interface (CDI) to safely attach that specific card into the container.

---

## How AIRuntime Works with DRA: Step-by-Step Technical Execution

The complete lifecycle of an AIRuntime from API call to active GPU execution follows these 7 distinct steps:

### Step 1: Request Validation, Capacity Hold, and Blueprint Submission (`airuntime-service`)

1. **User Action & API Call:** The user submits a request from the UI to `POST /api/v1beta1/orgs/{org_id}/projects/{project_id}/airuntimes` specifying the runtime name, template ID, GPU hardware config ID, GPU count, region, plan ID, persistent storage size, and optional SSH keys.
2. **Template & Container Image Resolution:** `airuntime-service` queries the database for the selected template (`s.queries.GetAIRuntimeTemplate`) to resolve the container image (`template.Image`), application ports (JupyterLab on port 8888), default environment variables, and any private registry credentials.
3. **Authentication, Quotas, and Plan Verification:** Validates user JWT authentication, verifies project compute quotas, and ensures the billing `planId` matches the requested GPU hardware configuration.
4. **Admission Capacity Hold (`inventory-service`):** Calls `inventoryClient.ReserveResources` to place a temporary 5-minute capacity hold in the database (`resource_reservations` table), preventing concurrent GPU VM or model deployment requests from taking the last GPU before this runtime is provisioned.
5. **Database Record Creation:** Inserts a new row into the PostgreSQL `airuntimes` table with initial status `provisioning` and stores the reservation ID.
6. **Build Blueprint (`AIRuntime` Custom Resource):** Calls `BuildAIRuntimeCR()` to assemble the `AIRuntime` CR (`ai.neevcloud.com/v1alpha1`) containing the target namespace, GPU specifications (model, VRAM, count), resolved container image, Ceph persistent storage configuration, and a randomly generated Jupyter password annotation.
7. **Submit Blueprint to Workload Cluster (`k8sClient.Create`):** Posts the `AIRuntime` CR to the Workload Cluster API server via `k8sClient.Create(ctx, cr)`, handing off execution to the `airuntime-operator`.

---

### Step 2: Operator Creates Storage, GPU Claim, and Pod (`airuntime-operator`)

When the `airuntime-operator` sees the new `AIRuntime` CR in the Kubernetes API server, it does three things:

1. **Creates the Storage Mount (PVC):** If persistent storage is requested, it ensures a `PersistentVolumeClaim` is ready to mount at `/data/workspace` so user files are never lost.
2. **Creates the GPU Request (`ResourceClaim`):** It submits a `ResourceClaim` to Kubernetes asking for the exact GPU needed:

   ```cel
   device.attributes['gpu.nvidia.com'].productName == 'NVIDIA GeForce RTX 5090' &&
   device.capacity['gpu.nvidia.com'].memory.compareTo(quantity('31Gi')) >= 0
   ```

3. **Creates the Pod Object:** It creates the Pod in Kubernetes with container image (`pytorch-jupyter:v2.4`), storage mount (`/data/workspace`), and attaches the GPU `ResourceClaim`. *(The Pod is created in `Pending` state and does not run yet).*

---

### Step 3: Scheduler Matches the Claim to a GPU Node (`kube-scheduler`)

Once the Pod and its `ResourceClaim` are in Kubernetes, `kube-scheduler` handles the matching:

1. **Reads the Claim:** The scheduler sees the pending Pod asking for a GPU via its `ResourceClaim`.
2. **Checks Node Hardware (`ResourceSlice`):** It checks the hardware catalog (`ResourceSlice`) published by all worker nodes to find a node that has a free card matching the GPU model and VRAM.
3. **Locks the GPU:** It finds an available RTX 5090 (`gpu-0`) on `worker-3`, writes `worker-3, gpu-0` into `ResourceClaim.status.allocation`, and locks the card so no other workload can take it.
4. **Places the Pod on the Node:** It binds the Pod to `worker-3` so the node can start preparing the hardware.

---

### Step 4: Node Hardware Preparation (`kubelet` & DRA Driver DaemonSet)

Every GPU worker node in the cluster runs an **NVIDIA DRA driver DaemonSet pod** (the kubelet plugin) that directly interfaces with the node's PCIe hardware.

1. **Kubelet Receives the Pod:** The `kubelet` agent on `worker-3` sees that the Pod has been assigned to its node and recognizes that it requires a DRA resource (`gpu-claim`).
2. **Local gRPC Call (`NodePrepareResources`):** `kubelet` connects to the local **NVIDIA DRA driver DaemonSet pod** running on that same node via a local UNIX socket and executes a `NodePrepareResources` gRPC call.
3. **Hardware Reset & CDI File Generation:**
   - The NVIDIA DRA driver daemon on `worker-3` validates physical `gpu-0`, inspects its hardware address, and clears any leftover memory from prior workloads.
   - The driver then writes the **CDI (Container Device Interface) JSON file** directly to the host filesystem at `/var/run/cdi/nvidia.com-gpu.json`:

     ```json
     {
       "cdiVersion": "0.5.0",
       "kind": "nvidia.com/gpu",
       "devices": [
         {
           "name": "gpu-0",
           "containerEdits": {
             "deviceNodes": [
               {"path": "/dev/nvidia0", "hostPath": "/dev/nvidia0"},
               {"path": "/dev/nvidiactl", "hostPath": "/dev/nvidiactl"},
               {"path": "/dev/nvidia-uvm", "hostPath": "/dev/nvidia-uvm"},
               {"path": "/dev/nvidia-uvm-tools", "hostPath": "/dev/nvidia-uvm-tools"}
             ],
             "mounts": [
               {
                 "hostPath": "/usr/lib/x86_64-linux-gnu/libcuda.so.1",
                 "containerPath": "/usr/lib/x86_64-linux-gnu/libcuda.so.1",
                 "options": ["ro", "nosuid", "nodev", "bind"]
               }
             ]
           }
         }
       ]
     }
     ```

   - This JSON file is the concrete instruction sheet that tells `containerd` exactly which `/dev/` files to create and which driver libraries to attach to the container.

---

### Step 5: Container Device Attachment and Process Launch (`containerd` & CDI)

The Pod is now on `worker-3`, and the NVIDIA driver has prepared the CDI instructions file. Next, `containerd` starts the container to run the user's requested workload (e.g., PyTorch + JupyterLab).

By default, a container is isolated and cannot see or access the physical GPU on the host server. To connect the GPU to this container, `containerd` reads the CDI file (`/var/run/cdi/nvidia.com-gpu.json`) and unlocks the **operating system gates**:

1. **Gate 1: Creating the Device Files inside the Pod's `/dev/` Directory:**
   - In Linux, **every piece of hardware is represented as a file** inside the `/dev/` directory (for example, `/dev/sda` for a disk).
   - By default, a Pod's container has its own private `/dev/` folder with **zero GPU files**. If PyTorch looks for a GPU, it finds nothing and throws an error: `No CUDA GPUs are available`.
   - **Only the Requested GPU is Added (Strict Isolation):**
     - If the host node has 8 physical GPUs (`gpu-0` through `gpu-7`) and you requested **1 GPU**, the CDI file contains **ONLY** the data for that single assigned GPU (`gpu-0`).
     - It does **NOT** include any other GPUs on the node. The other 7 cards (`/dev/nvidia1` through `/dev/nvidia7`) do not exist inside your container and are completely invisible.
   - `containerd` reads the CDI file and injects only the requested card's files directly into the Pod's private `/dev/` directory:
     - **`/dev/nvidia0`**: The direct data pipeline to physical GPU 0. PyTorch writes to this file to send AI models, weights, and CUDA calculation instructions.
     - **`/dev/nvidiactl`**: The master control switch used to initialize the card, power it up, and check temperature and clock speeds.
     - **`/dev/nvidia-uvm`**: Unified Virtual Memory file that allows CPU RAM and GPU VRAM to share data directly.
     - **`/dev/nvidia-uvm-tools`**: Profiling interface used by performance monitoring tools (like PyTorch Profiler).

2. **Gate 2: Device Security Filter (Cgroups v2):**
   - Even with the files in `/dev/`, the Linux kernel blocks unauthorized hardware access by default.
   - `containerd` sets a security rule that **allows access ONLY to GPU 0** (`gpu-0`). Access to all other GPUs on the machine is strictly blocked.
   - This ensures your container can only use its assigned card and cannot touch any other user's GPU on the server.

3. **Storage Mount:**
   - If persistent storage was requested, `containerd` mounts the Ceph PVC to `/data/workspace`.

4. **Container Entrypoint Execution:**
   - With device nodes injected, security filters set, and storage mounted, `containerd` executes the container entrypoint process:

     ```bash
     jupyter lab --ip 0.0.0.0 --port 8888
     ```

   - The Pod officially transitions from `Pending` to **`Running`**.

---

### Step 6: Interactive Access Routing

1. The container entrypoint executes JupyterLab on internal port `8888`.
2. `airuntime-operator` creates an HTTP route via the Gateway API exposing `https://.../jupyter` secured by `JUPYTER_PASSWORD`.
3. If SSH access was requested, the operator configures SSHPiper to proxy SSH connections to the pod's IP using tenant SSH public keys.
4. The user connects via their browser and executes CUDA code on the bare-metal GPU.

---

### Step 7: Release on Stop (`reconcileStop`)

1. When the user stops the AIRuntime, `airuntime-controller` triggers `reconcileStop`:
   - Deletes the Deployment / Pod.
   - Deletes the `ResourceClaim`.
   - Deleting the claim releases the device lock on `gpu-0`, making it immediately free for other users in the cluster.
   - The PVC and `AIRuntime` custom resource are preserved, ensuring all files, notebooks, and checkpoints are safely kept.

---

# Architectural Placement & End-to-End Execution Flows

To ensure anyone reviewing this architecture can clearly trace **where each component lives**, **how the request enters**, and **how the DRA driver prepares and attaches the GPU into the Pod**, the system is structured into 3 clean, progressive flows following the standard RFC layout:

- **Flow 1: Request Ingestion, Admission & Blueprint Assembly** (Client → Control Plane → Operator)
- **Flow 2: DRA Scheduling, Node Preparation & Container CDI Attachment** (Scheduler → Kubelet → DRA Driver DaemonSet → CDI → containerd)
- **Flow 3: Interactive Ingress Routing & Release on Stop** (Gateway API → Running Pod → Teardown Cleanup)

---

## Flow 1: Request Ingestion, Admission & Blueprint Assembly

When a user initiates runtime creation, the request enters the **Platform Control Plane (CP)** via the unified Public API Gateway. The Control Plane authenticates the user, verifies compute quotas and price plans, locks a 5-minute capacity hold in the inventory database, and submits the custom resource blueprint to the Workload Cluster.

### Flow Details:

1. **Client Request:** A client (Browser Console or Neev CLI) sends `POST /api/v1beta1/orgs/{org_id}/projects/{project_id}/airuntimes`. This hits the **public-api-gateway** (`api.ai.neevcloud.com`) in the Control Plane.
2. **Gateway Route:** The gateway matches the `/airuntimes` path prefix and routes the call to **airuntime-service** in namespace `neevai-system`.
3. **Template & Plan Verification:** `airuntime-service` queries **PostgreSQL** (`s.queries.GetAIRuntimeTemplate`) to resolve the container image, Jupyter ports, verify user quotas, and ensure the billing `planId` matches the hardware spec.
4. **Capacity Hold:** `airuntime-service` calls **inventory-service** (`inventoryClient.ReserveResources`) to place a 5-minute temporary hold in the `resource_reservations` table to prevent race conditions on the last available GPU.
5. **DB Insertion:** An entry is inserted into the PostgreSQL `airuntimes` table with initial state `provisioning`.
6. **Submit Blueprint:** `airuntime-service` invokes `BuildAIRuntimeCR()` and submits the `AIRuntime` Custom Resource (`ai.neevcloud.com/v1alpha1`) to the **K8s API Server** of the Workload Cluster via `k8sClient.Create`.
7. **Watch & Reconcile:** The **airuntime-operator** watching the Workload Cluster receives the new CR event.
8. **Create PVC:** The operator creates the `PersistentVolumeClaim` for Ceph storage mounted at `/data/workspace`.
9. **Create ResourceClaim:** The operator creates the DRA `ResourceClaim` containing the CEL expression:

   `device.attributes['gpu.nvidia.com'].productName == 'NVIDIA GeForce RTX 5090' && device.capacity['gpu.nvidia.com'].memory >= 31Gi`

10. **Create Pod Object:** The operator creates the Pod in **`Pending`** state referencing the PVC and `ResourceClaim`.

```mermaid
flowchart LR
    cl["client<br/>Console UI / CLI"]

    subgraph CP["Control Plane - neevai-system"]
        gw["public-api-gateway<br/>api.ai.neevcloud.com"]
        asvc["airuntime-service<br/>JWT Auth - Quota - Plan"]
        db[("PostgreSQL DB<br/>airuntimes - templates")]
        inv["inventory-service<br/>ReserveResources (5-min hold)"]
    end

    subgraph WK["Workload Cluster Control Plane"]
        k8s[("K8s API Server<br/>etcd")]
        aop["airuntime-operator<br/>AIRuntime Controller"]
        pvc[("PersistentVolumeClaim<br/>/data/workspace")]
        rc[("ResourceClaim<br/>CEL: RTX 5090 >= 31Gi")]
        pod["AIRuntime Pod<br/>State: Pending"]
    end

    cl -->|"1 POST /airuntimes"| gw
    gw -->|"2 forward"| asvc
    asvc -->|"3 resolve template & quota"| db
    asvc -->|"4 reserve GPU"| inv
    asvc -->|"5 insert (provisioning)"| db
    asvc -->|"6 create AIRuntime CR"| k8s
    k8s -->|"7 watch CR"| aop
    aop -->|"8 create PVC"| pvc
    aop -->|"9 create claim"| rc
    aop -->|"10 create Pod (Pending)"| pod

    classDef cp fill:#e7e6fb,stroke:#6b6be0,color:#20233a
    classDef dp fill:#cdeee7,stroke:#12a594,color:#10302b
    classDef star fill:#ffe6a7,stroke:#d99a1c,stroke-width:2px,color:#3a2c07
    classDef store fill:#e6e9ef,stroke:#5b6472,color:#20233a
```

---

## Flow 2: DRA Scheduling, Node Preparation & Container CDI Attachment

This is the core DRA engine. The Kubernetes scheduler matches the CEL claim against physical node inventory, and the GPU worker node prepares the PCIe device and uses the Container Device Interface (CDI) to attach only the allocated GPU to the container.

### Flow Details:

1. **Read Claim:** `kube-scheduler` detects the pending Pod and inspects its associated `ResourceClaim`.
2. **Match Inventory:** The scheduler compares the CEL request with the **ResourceSlice** inventory published by cluster nodes.
3. **Lock & Bind:** The scheduler matches physical `gpu-0` on `worker-3`, writes `worker-3, gpu-0` into `ResourceClaim.status.allocation`, and binds the Pod to `worker-3`.
4. **Node Notification:** The `kubelet` agent on `worker-3` detects the assigned Pod and identifies that it requires a DRA device.
5. **NodePrepareResources Call:** `kubelet` connects to the **NVIDIA DRA Driver DaemonSet pod** running locally on `worker-3` via a local UNIX socket gRPC endpoint (`NodePrepareResources`).
6. **Hardware Reset & CDI Generation:** The NVIDIA DRA driver validates `gpu-0`, clears prior memory/contexts, and writes the host CDI specification file to `/var/run/cdi/nvidia.com-gpu.json`.
7. **Initiate Container Creation:** `kubelet` commands **containerd** to create the container with the allocated CDI device reference (`nvidia.com/gpu=gpu-0`).
8. **Read CDI Sheet:** `containerd` reads `/var/run/cdi/nvidia.com-gpu.json` to extract required device nodes and library bind mounts.
9. **Gate 1 (Device Nodes):** `containerd` injects only `/dev/nvidia0`, `/dev/nvidiactl`, `/dev/nvidia-uvm`, and `/dev/nvidia-uvm-tools` into the container's private `/dev` directory. All other host GPUs (`nvidia1`–`nvidia7`) are completely hidden.
10. **Gate 2 (Cgroups v2):** `containerd` writes the cgroup device filter rules allowing read/write/mknod **only to `gpu-0`**, strictly blocking access to any other device ID on the host.
11. **Mount Storage & Entrypoint:** `containerd` mounts the Ceph PVC to `/data/workspace` and starts `jupyter lab --ip 0.0.0.0 --port 8888`. The Pod transitions from `Pending` to **`Running`**.

```mermaid
flowchart LR
    subgraph WK["Workload Cluster Control Plane"]
        rc[("ResourceClaim<br/>RTX 5090 >= 31Gi")]
        rs[("ResourceSlice<br/>Node GPU Catalog")]
        sched["kube-scheduler<br/>DRA Plugin"]
    end

    subgraph DP["GPU Worker Node - worker-3"]
        klet["kubelet agent"]
        dra["NVIDIA DRA Driver<br/>DaemonSet (UNIX socket)"]
        cdi[("CDI JSON Spec<br/>/var/run/cdi/...")]
        ctrd["containerd runtime"]
        dev["Gate 1: Injected /dev<br/>/dev/nvidia0, nvidiactl"]
        cg["Gate 2: Cgroups v2<br/>Allow ONLY gpu-0"]
        app["AIRuntime Pod<br/>jupyter lab :8888 (Running)"]
    end

    rc -->|"1 read claim"| sched
    rs -.->|"2 match inventory"| sched
    sched -->|"3 lock gpu-0 & bind Pod"| klet
    klet -->|"4 NodePrepareResources"| dra
    dra -->|"5 reset VRAM & write spec"| cdi
    klet -->|"6 run container"| ctrd
    cdi -.->|"7 read instructions"| ctrd
    ctrd -->|"8 populate /dev"| dev
    ctrd -->|"9 set device filter"| cg
    dev --> app
    cg --> app

    classDef cp fill:#e7e6fb,stroke:#6b6be0,color:#20233a
    classDef dp fill:#cdeee7,stroke:#12a594,color:#10302b
    classDef star fill:#ffe6a7,stroke:#d99a1c,stroke-width:2px,color:#3a2c07
    classDef store fill:#e6e9ef,stroke:#5b6472,color:#20233a
```

---

## Flow 3: Interactive Ingress Routing & Release on Stop

This flow illustrates how external clients interactively access the running workload and how resources are safely torn down when the runtime is stopped.

### Flow Details:

1. **Interactive Ingress Setup:** The `airuntime-operator` configures an HTTPRoute on the **Gateway API** pointing to port `8888` and synchronizes user SSH public keys with **SSHPiper**.
2. **JupyterLab Connection:** The user connects via HTTPS to `https://.../jupyter` in their browser, authenticates with the injected `JUPYTER_PASSWORD`, and traffic proxies directly to the container.
3. **SSH Connection:** Users connecting via SSH authenticate through SSHPiper directly into the container bash shell.
4. **Bare-Metal GPU Execution:** PyTorch, TensorFlow, or custom CUDA code communicates directly through `/dev/nvidia0` with near-zero virtualization overhead.
5. **Stop Request:** When the user stops the runtime, a stop request is sent to **airuntime-service**.
6. **Update CR:** `airuntime-service` updates the CR phase in the **K8s API Server**.
7. **reconcileStop Execution:** The `airuntime-operator` controller runs `reconcileStop`, deleting the Pod/Deployment and deleting the `ResourceClaim`.
8. **Release GPU:** Deleting the `ResourceClaim` clears the allocation, unlocking `gpu-0` so it returns immediately to the cluster pool for other users.
9. **Preserve Storage:** The Ceph `PersistentVolumeClaim` and the `AIRuntime` custom resource are preserved, ensuring all files, notebooks, and models remain completely intact.

```mermaid
flowchart LR
    cl["user<br/>Browser / SSH Client"]

    subgraph DP["Data Plane - Ingress & Workload"]
        gw["Gateway API (Envoy)<br/>HTTPS /jupyter"]
        ssh["SSHPiper Proxy<br/>Port 22"]
        pod["Running Pod (worker-3)<br/>Jupyter (:8888) + gpu-0"]
    end

    subgraph CP["Control Plane & Reconciler - Stop Flow"]
        asvc["airuntime-service<br/>Stop Endpoint"]
        k8s[("K8s API Server<br/>etcd")]
        aop["airuntime-operator<br/>reconcileStop"]
        pool[("Cluster GPU Pool<br/>gpu-0 freed")]
        pvc[("Ceph PVC<br/>Data Preserved")]
    end

    cl -->|"1 HTTPS / JUPYTER_PASSWORD"| gw
    cl -->|"2 SSH keys"| ssh
    gw -->|"3 proxy to :8888"| pod
    ssh -->|"4 proxy terminal"| pod

    cl -.->|"5 stop request"| asvc
    asvc -.->|"6 update CR"| k8s
    k8s -.->|"7 reconcileStop"| aop
    aop -->|"8 delete Pod & ResourceClaim"| pool
    aop -.->|"9 retain volume"| pvc

    classDef cp fill:#e7e6fb,stroke:#6b6be0,color:#20233a
    classDef dp fill:#cdeee7,stroke:#12a594,color:#10302b
    classDef star fill:#ffe6a7,stroke:#d99a1c,stroke-width:2px,color:#3a2c07
    classDef store fill:#e6e9ef,stroke:#5b6472,color:#20233a
```
