# Corrections to Existing Questions

## 1. Helper Node

**Replace Q2 with:**

**Q2. Why do we need Red Hat Satellite?**
**A:** Satellite provides controlled RHEL repositories, patches, lifecycle environments, subscriptions, and packages for helper servers and RHOSO RHEL data-plane nodes.

**Replace Q4 with:**

**Q4. Why is HAProxy used?**
**A:** In user-provisioned environments, HAProxy can provide the external load-balancing endpoints for the OpenShift API, Machine Config Server, and application Ingress.

**Replace Q5 with:**

**Q5. What happens if the load balancer fails?**
**A:** API, node installation, or application access can fail depending on which load-balancer endpoint is unavailable. Production deployments require redundant load balancers and a highly available virtual IP.

---

## 2. Deployment Methods

**Replace Q5 with:**

**Q5. Which method is preferred for disconnected bare metal?**
**A:** The Agent-Based Installer is generally preferred because it simplifies host discovery, validation, ISO-based installation, and disconnected deployment.

---

## 3. Connected vs Disconnected

**Replace Q4 with:**

**Q4. Why do enterprises use disconnected installation?**
**A:** To meet isolation, security, compliance, and controlled-content requirements. It also adds operational responsibility for mirroring and maintaining content.

**Replace Q5 with:**

**Q5. How does OpenShift redirect image pulls to a mirror?**
**A:** New deployments use `ImageDigestMirrorSet (IDMS)` and `ImageTagMirrorSet (ITMS)`. `ImageContentSourcePolicy (ICSP)` is deprecated.

---

## 4. RHOSO and Operator Installation

**Replace Q3 with:**

**Q3. Which Operators are required for RHOSO networking and security?**
**A:** Kubernetes NMState, MetalLB, cert-manager, and the OpenStack Operator.

**Replace Q4 with:**

**Q4. Why is cert-manager required?**
**A:** RHOSO uses cert-manager to issue, track, renew, and rotate certificates for TLS-everywhere.

---

# 7. DNS, DHCP, and NTP

**Q1. Why is DNS critical for OpenShift installation?**
**A:** OpenShift depends on DNS for API access, internal API access, application routes, node discovery, certificates, and communication between cluster components.

**Q2. Which DNS records are normally required?**
**A:** `api`, `api-int`, `*.apps`, bootstrap, control-plane, and worker node records.

**Q3. Why are reverse DNS records required?**
**A:** RHCOS can use PTR records to determine node hostnames, and correct hostnames are required for node registration and certificate requests.

**Q4. What is the role of DHCP?**
**A:** DHCP can provide node IP addresses, gateways, DNS servers, boot information, and hostnames.

**Q5. Why must all nodes use synchronized time?**
**A:** Incorrect time can cause TLS certificate failures, authentication errors, etcd problems, and inconsistent logs.

---

# 8. OpenShift Load-Balancer Ports

**Q1. What does port 6443 provide?**
**A:** The Kubernetes and OpenShift API endpoint.

**Q2. What does port 22623 provide?**
**A:** The Machine Config Server endpoint used by nodes to obtain Ignition and machine configuration information.

**Q3. What are ports 80 and 443 used for?**
**A:** HTTP and HTTPS application traffic handled by the OpenShift Ingress Controller.

**Q4. Which nodes receive API load-balancer traffic?**
**A:** Bootstrap and control-plane nodes during installation, and only control-plane nodes after bootstrap completes.

**Q5. Which nodes receive Ingress load-balancer traffic?**
**A:** Nodes running the OpenShift router pods, normally worker or dedicated infrastructure nodes.

The normal load-balancer mappings are:

* `6443` → bootstrap and control-plane nodes
* `22623` → bootstrap and control-plane nodes
* `80/443` → nodes running Ingress Controller pods

The bootstrap node must be removed from the API and Machine Config Server backend pools after bootstrap completes.

---

# 9. Agent-Based Installer

**Q1. Which main configuration files are used by the Agent-Based Installer?**
**A:** `install-config.yaml` defines the cluster, while `agent-config.yaml` defines host and network discovery information.

**Q2. What is the rendezvous IP?**
**A:** The IP address of the control-plane host that initially coordinates the Agent-Based installation.

**Q3. Can the gateway IP be used as the rendezvous IP?**
**A:** No. It must be an IP assigned to one of the intended control-plane hosts.

**Q4. How are hosts identified?**
**A:** Hosts can be matched using MAC addresses, interface details, hostnames, and other hardware information.

**Q5. What does the Agent ISO contain?**
**A:** RHCOS, discovery agents, cluster configuration, trust information, and the data needed to start installation.

Useful installation monitoring commands:

```bash
openshift-install --dir install-dir agent wait-for bootstrap-complete \
  --log-level=info

openshift-install --dir install-dir agent wait-for install-complete \
  --log-level=info
```

---

# 10. Bootstrap and Ignition

**Q1. What is the bootstrap process?**
**A:** It temporarily starts the components needed to initialize the permanent OpenShift control plane.

**Q2. Is there always a separate bootstrap server?**
**A:** No. Traditional UPI commonly uses a separate bootstrap node, while Agent-Based installation uses a control-plane host as the rendezvous host.

**Q3. What is Ignition?**
**A:** Ignition is the first-boot configuration system used by RHCOS to configure disks, files, users, systemd units, certificates, and cluster membership.

**Q4. What are bootstrap, master, and worker Ignition configurations?**
**A:** They contain role-specific first-boot configuration for bootstrap, control-plane, and worker nodes.

**Q5. What does bootstrap complete mean?**
**A:** The permanent control-plane nodes are running the services required to continue operating without the temporary bootstrap function.

---

# 11. OpenShift Control Plane and etcd

**Q1. What runs on an OpenShift control-plane node?**
**A:** The API server, etcd, scheduler, controller managers, OAuth services, Operators, CRI-O, and kubelet.

**Q2. What is etcd?**
**A:** A distributed key-value database that stores the authoritative state of the cluster.

**Q3. Why are three control-plane nodes normally used?**
**A:** Three members provide etcd quorum and allow the cluster to tolerate one control-plane failure.

**Q4. What is etcd quorum?**
**A:** The majority of etcd members that must agree before writes can be committed.

**Q5. What happens when etcd loses quorum?**
**A:** The cluster cannot safely process most state-changing API requests even if some existing workloads continue running.

---

# 12. Scheduler and Controllers

**Q1. What does the Kubernetes scheduler do?**
**A:** It selects the most appropriate node for each unscheduled pod.

**Q2. What does the controller manager do?**
**A:** It continuously compares desired state with actual state and takes action to remove differences.

**Q3. What is desired state?**
**A:** The configuration declared in Kubernetes and OpenShift API objects.

**Q4. What is reconciliation?**
**A:** The repeated process of moving the current environment toward the desired state.

**Q5. Why are controllers continuously running?**
**A:** Because nodes, pods, networks, storage, and applications can change or fail at any time.

---

# 13. Machine, MachineSet, MachineConfig, and MCP

**Q1. What is a Machine object?**
**A:** It represents the lifecycle of a cluster node when that platform is managed through the Machine API.

**Q2. What is a MachineSet?**
**A:** It maintains a desired number of similar Machine objects, similar to how a ReplicaSet maintains pods.

**Q3. What is a MachineConfig?**
**A:** A declarative object used to configure the RHCOS operating system, files, systemd services, kernel settings, CRI-O, and kubelet.

**Q4. What is a MachineConfigPool?**
**A:** A group of nodes that receive the same rendered MachineConfig.

**Q5. What is the Machine Config Operator?**
**A:** The Operator that renders MachineConfigs and safely applies them to nodes, including draining and rebooting when required.

Simple relationship:

```text
MachineSet
    └── creates Machine
            └── represents Node

MachineConfig
    └── rendered into MachineConfigPool
            └── applied to matching Nodes by MCO
```

A MachineSet creates or replaces infrastructure. An MCP configures the operating system of existing nodes. They are not the same function.

---

# 14. OVN-Kubernetes Internals

**Q1. What is OVN-Kubernetes?**
**A:** OpenShift’s default network plugin that implements pod networking, service networking, network policies, routing, NAT, and load-balancing rules.

**Q2. What is Open vSwitch?**
**A:** The software switch running on each node that forwards traffic according to OpenFlow rules programmed by OVN.

**Q3. What is `br-int`?**
**A:** The OVS integration bridge where pod interfaces and OVN logical networking are connected.

**Q4. What is `br-ex`?**
**A:** The OVS external bridge that connects OVN-managed traffic to the node’s physical or external network.

**Q5. What is Geneve?**
**A:** The tunnel protocol used by OVN-Kubernetes to carry overlay traffic between nodes.

Simplified cross-node pod flow:

```text
Source Pod
   ↓
Pod virtual interface
   ↓
br-int on source node
   ↓
Geneve tunnel
   ↓
br-int on destination node
   ↓
Destination pod virtual interface
   ↓
Destination Pod
```

OVN stores the intended logical topology in its databases. `ovn-controller` converts that topology into OpenFlow rules and programs OVS on each node.

---

# 15. Pod IP, Service IP, and Machine IP

**Q1. What is a Machine IP?**
**A:** A real IP assigned to a physical or virtual cluster node on the machine network.

**Q2. What is a Pod IP?**
**A:** An IP allocated from the cluster network and assigned to a pod interface.

**Q3. What is a Service IP?**
**A:** A virtual stable IP allocated from the service network.

**Q4. Is a ClusterIP assigned to a physical interface?**
**A:** No. It is a virtual IP implemented using OVN load-balancing and forwarding rules.

**Q5. How are these networks connected?**
**A:** OVN programs routing, load balancing, NAT, and overlay forwarding between pod, service, and machine networks.

Simplified flow:

```text
Machine IP
    Physical node connectivity

Pod IP
    Workload-to-workload connectivity

Service IP
    Stable virtual frontend for a changing group of pod IPs
```

---

# 16. Services, Ingress, and Routes

**Q1. What is a Kubernetes Service?**
**A:** A stable virtual frontend that selects one or more backend pods.

**Q2. What is an EndpointSlice?**
**A:** An API object containing the current backend pod IPs and ports for a Service.

**Q3. What is an OpenShift Route?**
**A:** An OpenShift API object that exposes a Service using a DNS hostname.

**Q4. What is an Ingress object?**
**A:** A Kubernetes object that defines HTTP or HTTPS routing rules to Services.

**Q5. Why do both Route and Ingress exist?**
**A:** Ingress is the Kubernetes-standard API, while Route is OpenShift’s native API with OpenShift-specific TLS and HAProxy capabilities.

Simplified application flow:

```text
Client
  ↓
External DNS
  ↓
Ingress Load Balancer
  ↓
OpenShift Router Pod
  ↓
Route or Ingress rule
  ↓
Backend application endpoint
  ↓
Application Pod
```

A Route is configuration. The router pod running HAProxy performs the actual traffic forwarding.

---

# 17. CoreDNS and External DNS

**Q1. What is external DNS?**
**A:** The DNS system that resolves public or corporate names such as `api.cluster.example.com` and `*.apps.cluster.example.com`.

**Q2. What is CoreDNS in OpenShift?**
**A:** The internal DNS service used by pods to resolve Kubernetes Services and other internal names.

**Q3. What is the normal Service DNS format?**
**A:** `<service>.<namespace>.svc.cluster.local`.

**Q4. What happens when a pod queries an external name?**
**A:** CoreDNS forwards the request to the configured upstream DNS servers.

**Q5. What happens if internal DNS fails?**
**A:** Applications might be unable to locate Services even when the target pods are healthy.

---

# 18. Multus and Secondary Networks

**Q1. What is Multus?**
**A:** A meta-CNI that allows a pod to have more than one network interface.

**Q2. What is the primary pod network?**
**A:** The default OVN-Kubernetes network used for normal Kubernetes pod and Service communication.

**Q3. What is a secondary network?**
**A:** An additional network attached to selected pods for storage, management, provider, or high-performance traffic.

**Q4. What is a NetworkAttachmentDefinition?**
**A:** A custom resource that describes how Multus should attach a secondary network to a pod.

**Q5. Why does RHOSO use Multus?**
**A:** OpenStack services must communicate over multiple isolated networks in addition to the primary OpenShift pod network.

---

# 19. RHOSO Network Types

**Q1. What is the Control Plane network?**
**A:** The network used to manage and provision RHOSO data-plane nodes.

**Q2. What is the Internal API network?**
**A:** The network used for communication between internal OpenStack service endpoints.

**Q3. What is the Storage network?**
**A:** The network used for storage traffic between OpenStack services, Compute nodes, and Ceph.

**Q4. What is the Tenant network?**
**A:** The network used to transport tenant virtual-machine traffic, commonly using OVN encapsulation.

**Q5. What is the External network?**
**A:** The network that provides north-south connectivity between OpenStack workloads and external networks.

Common RHOSO networks:

```text
ctlplane     → node management and provisioning
internalapi  → private OpenStack API communication
storage      → storage client traffic
tenant       → tenant overlay traffic
external     → public/provider connectivity
```

---

# 20. NMState, NAD, and MetalLB in RHOSO

**Q1. What does NMState do for RHOSO?**
**A:** It configures worker-node interfaces, Linux bonds, VLANs, bridges, routes, and network addressing.

**Q2. What does an NNCP do?**
**A:** A `NodeNetworkConfigurationPolicy` declares the required host network configuration for selected nodes.

**Q3. What does a NAD do in RHOSO?**
**A:** It attaches an OpenStack service pod to an isolated RHOSO network.

**Q4. What does MetalLB do in RHOSO?**
**A:** It assigns and advertises virtual IP addresses for OpenStack service endpoints on isolated networks.

**Q5. How are RHOSO public APIs exposed by default?**
**A:** Public service endpoints are normally exposed through OpenShift Routes, while internal endpoints use isolated networks and MetalLB VIPs.

Simplified relationship:

```text
NNCP
  Configures the network on the OpenShift node

NAD
  Connects a pod to that network

MetalLB
  Advertises a service VIP on that network
```

---

# 21. RHOSO Control Plane and Data Plane

**Q1. What is the RHOSO control plane?**
**A:** Containerized OpenStack API, database, messaging, networking, identity, image, and management services running as OpenShift workloads.

**Q2. What is the RHOSO data plane?**
**A:** RHEL-based nodes that run Compute, networking, storage client, and supporting OpenStack services.

**Q3. What is `OpenStackControlPlane`?**
**A:** The custom resource that declares the required RHOSO control-plane services.

**Q4. What is `OpenStackDataPlaneNodeSet`?**
**A:** The custom resource that defines groups of data-plane nodes, their networks, roles, and configuration.

**Q5. What is `OpenStackDataPlaneDeployment`?**
**A:** The custom resource that triggers service deployment to one or more data-plane node sets.

Simplified architecture:

```text
OpenShift Cluster
    └── RHOSO Control Plane Pods
          ├── Keystone
          ├── Nova API
          ├── Neutron Server
          ├── Glance
          ├── Cinder API
          ├── MariaDB
          └── RabbitMQ

RHEL Data Plane
    ├── Compute nodes
    ├── OVN controllers
    ├── Libvirt
    └── Storage clients
```

---

# 22. EDPM

**Q1. What does EDPM mean?**
**A:** External Data Plane Management.

**Q2. Why is it called external?**
**A:** The RHEL data-plane nodes exist outside the OpenShift cluster but are managed by RHOSO Operators and Ansible jobs running from OpenShift.

**Q3. How are EDPM nodes accessed?**
**A:** Commonly through SSH using credentials stored in Kubernetes Secrets.

**Q4. What configures EDPM nodes?**
**A:** Operator-generated Ansible jobs install packages, configure networking, deploy services, and validate the nodes.

**Q5. Are EDPM nodes OpenShift worker nodes?**
**A:** No. They are RHEL OpenStack data-plane nodes, not Kubernetes nodes.

---

# 23. RHOSO Storage and Ceph

**Q1. Why is Ceph commonly used with RHOSO?**
**A:** It provides scalable and redundant block, image, file, and object storage.

**Q2. Which OpenStack service provides block storage?**
**A:** Cinder.

**Q3. Which service stores virtual-machine images?**
**A:** Glance.

**Q4. Which service provides shared file systems?**
**A:** Manila.

**Q5. Why should storage traffic use a separate network?**
**A:** To isolate high-volume storage traffic from API, management, and tenant traffic.

Common mappings:

```text
Nova     → creates and manages Compute instances
Cinder   → block volumes
Glance   → VM images
Manila   → shared file systems
Ceph RBD → storage backend for images, volumes, and VM disks
```

---

# 24. Disconnected Operator Installation

**Q1. What must be mirrored for Operator installation?**
**A:** Operator index images, bundle images, related operand images, and any additional required container images.

**Q2. What is a CatalogSource?**
**A:** An object that makes an Operator catalog available to Operator Lifecycle Manager.

**Q3. What is an ImageSetConfiguration?**
**A:** The declarative file that tells `oc-mirror` which OpenShift releases, Operators, Helm charts, and additional images to mirror.

**Q4. Which oc-mirror version should new deployments use?**
**A:** `oc-mirror` plugin v2.

**Q5. Why must the mirror registry remain available after installation?**
**A:** Nodes continue using it for pod scheduling, scaling, upgrades, recovery, and Operator lifecycle operations.

Typical fully disconnected process:

```text
Internet-connected environment
          ↓
oc-mirror to disk
          ↓
Transfer archive
          ↓
oc-mirror disk to registry
          ↓
Apply IDMS, ITMS and CatalogSource
          ↓
Install OpenShift and Operators
```

---

# 25. OpenShift Installation Sequence

**Q1. What should be prepared before installing OpenShift?**
**A:** DNS, load balancing, IP addressing, NTP, registry, certificates, firewall rules, node hardware, and installation configuration.

**Q2. When should the OpenShift images be mirrored?**
**A:** Before starting a disconnected installation.

**Q3. When should the cluster Operators be validated?**
**A:** Immediately after installation and before installing RHOSO.

**Q4. When should RHOSO networking be configured?**
**A:** After OpenShift is healthy and before creating the RHOSO control plane.

**Q5. When should RHOSO data-plane nodes be deployed?**
**A:** After the OpenStack Operator, networking, secrets, and RHOSO control plane are ready.

Recommended sequence:

```text
1. Prepare DNS, NTP and load balancers
2. Prepare Satellite and the mirror registry
3. Mirror OpenShift and Operator images
4. Configure node networks and switches
5. Install OpenShift
6. Validate OpenShift Operators and nodes
7. Install cert-manager, NMState and MetalLB
8. Install the OpenStack Operator
9. Configure RHOSO isolated networks
10. Create the RHOSO control plane
11. Create the RHOSO data plane
12. Deploy and validate data-plane services
13. Create provider networks
14. Run OpenStack smoke tests
```

---

# 26. OpenShift Validation

**Q1. How do you verify cluster Operators?**
**A:** Run `oc get clusteroperators`.

**Q2. How do you verify nodes?**
**A:** Run `oc get nodes -o wide`.

**Q3. How do you verify MachineConfigPools?**
**A:** Run `oc get mcp` and confirm they are updated and not degraded.

**Q4. How do you verify the Ingress Controller?**
**A:** Check the Ingress Controller resource, router pods, wildcard DNS, and application route access.

**Q5. What is the minimum healthy state before RHOSO installation?**
**A:** Nodes must be Ready, required ClusterOperators Available, MCPs updated, storage operational, and DNS and Ingress functional.

Useful commands:

```bash
oc get nodes -o wide
oc get clusteroperators
oc get mcp
oc get pods -A | grep -vE 'Running|Completed'
oc get clusterversion
oc get ingresscontroller -n openshift-ingress-operator
oc get authentication.config.openshift.io cluster -o yaml
```

---

# 27. RHOSO Validation

**Q1. How do you verify the OpenStack Operator?**
**A:** Check the Operator CSV, Operator pods, and the status of the OpenStack initialization resource.

**Q2. How do you verify the RHOSO control plane?**
**A:** Check `OpenStackControlPlane` conditions and confirm that all expected service pods are ready.

**Q3. How do you verify the RHOSO data plane?**
**A:** Check the NodeSet, DataPlaneDeployment conditions, Ansible jobs, and services on each RHEL node.

**Q4. How do you test OpenStack APIs?**
**A:** Enter the OpenStackClient pod and run OpenStack CLI commands.

**Q5. What is a basic RHOSO smoke test?**
**A:** Create a network, subnet, router, flavor, image, security group, key pair, instance, volume, and floating IP, then verify connectivity.

Useful commands:

```bash
oc get openstackcontrolplane -n openstack
oc get openstackdataplanenodeset -n openstack
oc get openstackdataplanedeployment -n openstack
oc get pods -n openstack
oc get jobs -n openstack
oc get events -n openstack --sort-by=.lastTimestamp
```

---

# 28. TLS and Security

**Q1. What is TLS-everywhere in RHOSO?**
**A:** Encryption of public and internal OpenStack service communication using TLS certificates.

**Q2. Can TLS-e be disabled in RHOSO?**
**A:** No. RHOSO requires TLS for supported service communication.

**Q3. Where are credentials normally stored?**
**A:** In Kubernetes Secrets.

**Q4. Why should default certificates be replaced?**
**A:** Production environments usually require certificates issued by the organization’s trusted certificate authority.

**Q5. Why should access be role-based?**
**A:** RBAC limits users and service accounts to only the operations they require.

---

# 29. High Availability

**Q1. What does high availability mean?**
**A:** The service continues operating when an individual component fails.

**Q2. Which helper services require HA?**
**A:** DNS, load balancing, registry, storage, time services, and critical content infrastructure.

**Q3. Is running multiple HAProxy processes enough?**
**A:** No. A redundant frontend IP mechanism such as VRRP or an external appliance is also required.

**Q4. Why must the mirror registry be highly available?**
**A:** Registry failure can prevent pod restarts, node recovery, scaling, installation, and upgrades.

**Q5. Why does RHOSO use multiple replicas?**
**A:** Multiple replicas reduce service interruption when pods, nodes, or processes fail.

---

# 30. Common Troubleshooting Questions

**Q1. The API hostname resolves but port 6443 is unreachable. What should be checked?**
**A:** Load-balancer listeners, backend health, firewall rules, API server pods, and routing.

**Q2. A node cannot join the cluster. What should be checked?**
**A:** DNS, NTP, Ignition access, port 22623, certificates, network configuration, and kubelet logs.

**Q3. Pods cannot communicate across nodes. What should be checked?**
**A:** OVN pods, OVS bridges, Geneve connectivity, MTU, firewall rules, routes, and node annotations.

**Q4. An RHOSO service pod cannot reach an isolated network. What should be checked?**
**A:** NNCP status, NAD configuration, Multus annotations, VLAN configuration, routes, and the pod’s secondary interface.

**Q5. A data-plane deployment fails. What should be checked?**
**A:** DataPlaneDeployment conditions, Kubernetes Jobs, Ansible logs, SSH access, Satellite repositories, DNS, and node network configuration.

---

# Expanded Rapid Fire

* **Helper Node:** Hosts installation and management support services.
* **Satellite:** Manages RHEL RPM content and lifecycle.
* **Mirror Registry:** Stores OpenShift, Operator, and application images.
* **HAProxy:** Software-based TCP and HTTP load balancer.
* **Keepalived:** Provides a floating virtual IP using VRRP.
* **API VIP:** Stable frontend address for the OpenShift API.
* **Ingress VIP:** Stable frontend address for application traffic.
* **Port 6443:** Kubernetes API.
* **Port 22623:** Machine Config Server.
* **Ports 80/443:** OpenShift application Ingress.
* **IPI:** Installer provisions infrastructure.
* **UPI:** Administrator provisions infrastructure.
* **Agent-Based Installer:** ISO-based host discovery and installation.
* **Rendezvous Host:** Control-plane host coordinating Agent installation.
* **Ignition:** First-boot RHCOS configuration.
* **RHCOS:** Immutable operating system used by OpenShift nodes.
* **CRI-O:** OpenShift container runtime.
* **Kubelet:** Node agent responsible for running pods.
* **etcd:** Stores authoritative cluster state.
* **Scheduler:** Chooses nodes for pods.
* **Controller:** Reconciles actual state with desired state.
* **Operator:** Kubernetes controller containing operational knowledge.
* **Machine:** Infrastructure representation of a node.
* **MachineSet:** Maintains multiple similar Machines.
* **MachineConfig:** Declares host operating-system configuration.
* **MCP:** Groups nodes receiving the same MachineConfig.
* **MCO:** Applies operating-system configuration to nodes.
* **OVN:** Logical network control system.
* **OVS:** Software switch on each node.
* **br-int:** OVN integration bridge.
* **br-ex:** External OVS bridge.
* **Geneve:** OVN overlay tunnel protocol.
* **Machine Network:** Real node network.
* **Cluster Network:** Pod IP network.
* **Service Network:** Virtual Service IP network.
* **ClusterIP:** Internal virtual Service address.
* **EndpointSlice:** Current backend pod addresses.
* **Ingress Controller:** Implements external HTTP and HTTPS routing.
* **Route:** OpenShift hostname-routing configuration.
* **CoreDNS:** Internal Kubernetes DNS.
* **Multus:** Adds secondary interfaces to pods.
* **NAD:** Defines a secondary pod network.
* **NMState:** Declarative host network configuration.
* **NNCP:** Applies NMState configuration to selected nodes.
* **MetalLB:** Provides and advertises service VIPs.
* **L2Advertisement:** Controls MetalLB Layer 2 VIP advertisement.
* **RHOSO:** OpenStack control plane running on OpenShift.
* **EDPM:** External Data Plane Management.
* **OpenStackControlPlane:** Declares RHOSO control-plane services.
* **OpenStackDataPlaneNodeSet:** Declares RHOSO data-plane nodes.
* **OpenStackDataPlaneDeployment:** Starts data-plane service deployment.
* **Keystone:** Identity service.
* **Nova:** Compute service.
* **Neutron:** Networking service.
* **Glance:** Image service.
* **Cinder:** Block-storage service.
* **Manila:** Shared-file-system service.
* **Barbican:** Secrets-management service.
* **Ceilometer:** Telemetry data collection.
* **Horizon:** OpenStack web dashboard.
* **Ceph:** Distributed storage platform.
* **RBD:** Ceph block-device storage.
* **Control Plane Network:** Manages RHOSO data-plane nodes.
* **Internal API Network:** Private OpenStack API traffic.
* **Storage Network:** Storage client communication.
* **Tenant Network:** Virtual-machine tenant traffic.
* **External Network:** North-south provider connectivity.
* **CatalogSource:** Local Operator catalog configuration.
* **IDMS:** Digest-based registry mirror configuration.
* **ITMS:** Tag-based registry mirror configuration.
* **oc-mirror v2:** Preferred disconnected-content mirroring tool.
* **TLS-e:** Encryption for RHOSO service communication.
* **RBAC:** Role-based authorization.
* **Quorum:** Majority required for distributed consensus.
* **Reconciliation:** Repeated enforcement of desired state.
