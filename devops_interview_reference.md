# DevOps Interview Reference — Complete Questions & Answers

35 questions across 6 sections. Answers are written the way you would speak them in an interview: complete, clear, and self-contained.

---

## Section 1: Linux & Networking

### Q1. Explain the complete request flow from a browser to a Kubernetes pod.

**Answer:**

The flow has two important layers to keep separate: the **control plane** (kube-apiserver, etcd, scheduler, controllers — these manage the cluster) and the **data plane** (nodes, kube-proxy, CNI — these carry user traffic). A user request **never touches the API server**. The API server is only for cluster management: kubectl, kubelet, and controllers talk to it. etcd sits behind the API server as the cluster database — it stores all cluster state, and nothing else talks to it directly.

The actual user request flow:

1. **DNS resolution**: The browser resolves `app.example.com`. This points to the public IP of an external load balancer (a cloud LB, MetalLB, or an HAProxy/F5 in front of the cluster).
2. **TCP and TLS**: The browser opens a TCP connection to that IP on port 443 and performs the TLS handshake. TLS usually terminates at the load balancer or at the Ingress controller.
3. **Load balancer to node**: The external LB forwards the traffic to one of the worker nodes — either to a NodePort that the Ingress controller's Service exposes, or directly to the Ingress controller pods if the LB integrates with the cluster.
4. **Node networking (kube-proxy)**: When the packet arrives on the node, iptables or IPVS rules — programmed by kube-proxy — perform DNAT: they translate the Service's virtual IP (ClusterIP or NodePort) into a real Pod IP. The ClusterIP is purely virtual; it only exists inside these rules.
5. **Ingress controller**: The packet reaches an Ingress controller pod (nginx, HAProxy, or the OpenShift Router). It inspects the HTTP Host header and path, matches it against Ingress/Route rules, and proxies the request to the correct application Service.
6. **Service to Pod**: kube-proxy again picks one healthy backend pod for that Service. Healthy means the pod passed its readiness probe — only Ready pods appear in the Service's EndpointSlice.
7. **CNI delivers the packet**: If the chosen pod is on another node, the CNI plugin (OVN-Kubernetes, Calico, Flannel) carries the packet there — either through an overlay network (Geneve/VXLAN encapsulation) or direct routing. On the destination node, the packet passes through a veth pair into the pod's network namespace.
8. **The container** listening on the targetPort processes the request. The response follows the same path back using connection tracking (conntrack) state.

One-line summary for the interview: *Browser → DNS → external LB → node → kube-proxy DNAT → Ingress controller → Service → EndpointSlice → CNI → veth → pod. The API server and etcd are control plane only and are never in the user traffic path.*

---

### Q2. Difference between TCP and UDP. Where have you used each?

**Answer:**

**TCP** is connection-oriented. Before any data is sent, a three-way handshake (SYN → SYN-ACK → ACK) establishes a session. TCP guarantees delivery (lost packets are retransmitted), guarantees ordering (segments are sequenced), and applies flow and congestion control so a fast sender doesn't overwhelm a slow receiver or the network. The cost of all this is higher latency and overhead.

**UDP** is connectionless. Packets (datagrams) are simply sent — no handshake, no delivery guarantee, no ordering, no retransmission. It is much lighter and faster, and the application is responsible for handling loss if it cares.

Where I've used each in practice:

- **TCP**: everything that must be reliable — HTTP/HTTPS traffic, etcd client and peer communication (ports 2379/2380), database connections, SSH, the Kubernetes API on 6443.
- **UDP**: DNS queries (small, fast, retried by the client if lost), VXLAN and Geneve overlay encapsulation used by CNI plugins to carry pod traffic between nodes, NTP for time sync, and syslog forwarding.

A useful interview line: *DNS uses UDP for speed but falls back to TCP when the response is too large or for zone transfers.*

---

### Q3. How does DNS resolution work internally?

**Answer:**

On a normal Linux host:

1. The application first checks the **local cache** and `/etc/hosts` (order controlled by `/etc/nsswitch.conf`).
2. If not found, the **stub resolver** sends the query to the nameserver listed in `/etc/resolv.conf` — usually a recursive resolver.
3. The **recursive resolver** does the real work if its cache is empty: it asks a **root nameserver**, which refers it to the **TLD nameserver** (for `.com`), which refers it to the **authoritative nameserver** for the domain, which finally returns the A/AAAA record.
4. The answer is cached at every layer according to its **TTL**, so repeat lookups are fast. Negative answers (NXDOMAIN) are also cached.

Inside Kubernetes it works differently, and this is worth explaining in an interview:

- Every pod's `/etc/resolv.conf` points to the **CoreDNS** Service ClusterIP.
- It contains **search domains** — `<namespace>.svc.cluster.local`, `svc.cluster.local`, `cluster.local` — and the option `ndots:5`.
- `ndots:5` means: if a name has fewer than 5 dots, try appending each search domain first before treating it as an absolute name. That's why a pod can reach another Service just by its short name `myservice` — it expands to `myservice.<ns>.svc.cluster.local`.
- CoreDNS answers cluster-internal records itself and forwards everything external to the upstream resolver.
- Side effect worth mentioning: because of `ndots:5`, external lookups like `api.github.com` (only 2 dots) first fail through all the search domains before resolving — a known source of DNS latency in clusters, fixable with a trailing dot (`api.github.com.`) or a lower ndots setting.

---

### Q4. How would you debug high CPU, memory, or disk utilization on a Linux server?

**Answer:**

**High CPU:**
1. Start with `top` or `htop` and read the CPU breakdown carefully: high **user** time means an application is busy; high **system** time means heavy kernel/syscall activity; high **iowait** means the CPU is idle waiting for disk — that's a disk problem, not a CPU problem; high **steal** on a VM means the hypervisor is starving you — a noisy-neighbor problem.
2. Identify the process with `pidstat -u 1` or the top list, then go deeper with `perf top` to see which functions are hot, or `strace` to see if it's stuck in a syscall loop.
3. Check load average vs core count: load 8 on a 4-core box means work is queuing.

**High memory:**
1. `free -m` — and interpret it correctly: memory used by **buffers/cache** is reclaimable and healthy; look at the "available" column, not "free".
2. `ps aux --sort=-%mem | head` to find the consumer.
3. `dmesg | grep -i oom` — if the OOM killer has fired, it logs exactly which process it killed and why.
4. `vmstat 1` — sustained swap-in/swap-out (si/so) means real memory pressure. `/proc/meminfo` for slab or hugepage anomalies if no process explains the usage.

**High disk:**
1. `df -h` for space, and **`df -i` for inodes** — a filesystem can be "full" with plenty of free space if millions of small files exhausted the inodes. Classic trap.
2. `du -xsh /* | sort -h` to walk down to the offending directory (`-x` stays on one filesystem).
3. `lsof +L1` — finds deleted files still held open by a process. Very common: a huge log file was deleted but the daemon still holds the file descriptor, so the space is not freed until the process restarts.
4. For disk *performance* rather than space: `iostat -x 1` — high `%util` and high `await` mean the device is saturated.

---

### Q5. Explain Load Balancer, Reverse Proxy, and Ingress with real examples.

**Answer:**

**Load Balancer** — distributes incoming traffic across multiple backend servers so no single server is overwhelmed and failure of one backend doesn't cause an outage. It can work at **Layer 4** (TCP level — it just forwards connections, no understanding of HTTP) or **Layer 7** (HTTP-aware — can route on paths and headers). It performs health checks and removes dead backends automatically.
*Real example:* an HAProxy in TCP mode in front of the three OpenShift control-plane nodes, load-balancing the API on port 6443 — this is a standard part of a UPI installation.

**Reverse Proxy** — a server that sits in front of your application servers and receives all client requests on their behalf. Beyond forwarding, it adds value: TLS termination, caching, compression, header manipulation, hiding backend topology.
*Real example:* Nginx in front of an application, terminating HTTPS and forwarding plain HTTP to the app on localhost.
Relationship: every L7 load balancer is a reverse proxy, but a reverse proxy with a single backend isn't load balancing.

**Ingress** — this is a **Kubernetes API object**, not a server. It declares L7 routing rules: "traffic for host X, path Y goes to Service Z." On its own it does nothing — it needs an **Ingress controller** (nginx-ingress, HAProxy, Traefik, or the OpenShift Router), which is itself a reverse proxy running as pods inside the cluster. The controller watches Ingress objects through the API server and reconfigures itself whenever rules change.
*Real example:* wildcard DNS `*.apps.cluster.example.com` points at the router pods; a Route/Ingress for `myapp.apps.cluster.example.com` sends matching requests to the `myapp` Service, which forwards to the pods.

So the full chain in production is usually: external Load Balancer → Ingress controller (a reverse proxy) → Service → Pods.

---

## Section 2: Docker & Kubernetes

### Q1. What happens internally when you run `docker run`?

**Answer:**

Step by step:

1. **CLI → daemon**: the `docker` CLI sends the request to `dockerd` over the Unix socket `/var/run/docker.sock`. dockerd delegates container lifecycle to **containerd**.
2. **Image pull**: containerd checks the local content store. If the image isn't present, it pulls it from the registry — layer by layer. Each layer is a content-addressed tarball; layers already present locally are skipped, which is why pulls of related images are fast.
3. **Filesystem assembly**: the image's read-only layers are stacked using **overlayfs**, and a fresh **writable layer** is placed on top. Everything the container writes goes into that writable layer; the image layers are never modified. This is why containers are disposable — delete the container and only the writable layer is lost.
4. **Isolation setup**: the low-level runtime **runc** creates the container using two kernel features:
   - **Namespaces** give the process its own isolated view: PID namespace (it sees itself as PID 1), NET (own network stack), MNT (own mounts), UTS (own hostname), IPC, and optionally USER.
   - **cgroups** enforce resource limits — how much CPU and memory the container may use.
   - Security is applied: Linux capabilities are dropped, seccomp and AppArmor/SELinux profiles restrict syscalls.
5. **Networking**: a **veth pair** is created — one end goes inside the container's network namespace as `eth0`, the other attaches to the `docker0` bridge on the host. The container gets an IP from the bridge subnet, and `-p 8080:80` port publishing is implemented with iptables NAT rules.
6. **Process start**: runc executes the image's ENTRYPOINT/CMD as **PID 1** inside the container. A small **containerd-shim** process stays as its parent, so the container keeps running even if dockerd restarts.

Key insight to state: *a container is not a VM — it's just a normal Linux process wrapped in namespaces and cgroups, sharing the host kernel.*

---

### Q2. Difference between CMD and ENTRYPOINT.

**Answer:**

Both define what runs when the container starts, but they behave differently when you pass arguments:

- **ENTRYPOINT** is the fixed executable. Arguments given to `docker run` are **appended** to it. It can only be replaced explicitly with `--entrypoint`.
- **CMD** provides **default arguments**. Anything you pass to `docker run` **completely replaces** CMD.

They are designed to be used together:

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

- `docker run myimage` → runs `nginx -g "daemon off;"` (ENTRYPOINT + default CMD)
- `docker run myimage -T` → runs `nginx -T` (your argument replaced CMD, ENTRYPOINT stayed)

If only CMD is set, `docker run myimage bash` replaces the whole command with `bash` — useful for debugging images.

One more point that shows real experience: always use the **exec form** (JSON array: `["nginx", "-g", ...]`), not the shell form (`CMD nginx -g ...`). The shell form wraps the command in `/bin/sh -c`, so the shell becomes PID 1 and your application never receives SIGTERM — which breaks graceful shutdown in Kubernetes, because pods then wait out the full grace period and get SIGKILLed.

---

### Q3. Why is a pod stuck in CrashLoopBackOff? How would you debug it?

**Answer:**

**What it means:** the container starts, exits (crashes or completes), kubelet restarts it, it exits again — and kubelet applies an exponential backoff between restarts (10s, 20s, 40s… capped at 5 minutes). So CrashLoopBackOff is not itself the error; it's the symptom of a container that repeatedly dies.

**Debugging sequence:**

1. **`kubectl describe pod <pod>`** — look at "Last State" and especially the **exit code**:
   - **Exit 1** — generic application error; go to logs.
   - **Exit 137** — killed by SIGKILL: either **OOMKilled** (check the reason field; container exceeded its memory limit) or killed after a failed liveness probe.
   - **Exit 139** — segmentation fault.
   - **Exit 143** — SIGTERM; something asked it to stop.
   - **Exit 126/127** — command not executable / command not found: wrong `command:` in the manifest or missing binary in the image.
2. **`kubectl logs <pod> --previous`** — the crucial flag: it shows logs from the **crashed** container instance, not the fresh one that hasn't failed yet.
3. **Check the usual root causes:**
   - Wrong command/args or missing environment variable.
   - Missing ConfigMap or Secret that the pod mounts (describe shows mount failures).
   - **Liveness probe too aggressive** — a slow-starting app gets killed before it's ready, forever. Fix with a `startupProbe` or longer `initialDelaySeconds`.
   - Memory limit too low → OOMKill loop.
   - A dependency (DB, another service) unavailable at startup and the app exits instead of retrying.
   - Permission issues — the app writes somewhere it can't, common under OpenShift's restricted SCC with arbitrary UIDs.
4. **If it dies too fast to inspect**: temporarily override the entrypoint — `command: ["sleep", "infinity"]` — then `kubectl exec` in and run the real command manually to see what happens.

---

### Q4. Difference between Deployment, StatefulSet, DaemonSet, and Job.

**Answer:**

All four are workload controllers; they differ in what guarantees they give.

**Deployment** — for **stateless** applications. Pods are identical and interchangeable, get random names, can start/stop in any order, and typically share nothing. Manages rolling updates by creating a new ReplicaSet and shifting pods over gradually. Use for: web frontends, REST APIs, stateless workers. This is the default choice.

**StatefulSet** — for applications that need a **stable identity**:
- Predictable, ordered pod names: `db-0`, `db-1`, `db-2`.
- Stable per-pod DNS through a headless Service: `db-0.db.namespace.svc.cluster.local` — other pods can address a *specific* replica.
- A **dedicated PVC per pod** from `volumeClaimTemplates`; when `db-1` is rescheduled to another node, it reattaches to *its own* volume.
- Ordered operations: pods are created, scaled, and deleted in sequence.
Use for: databases, Kafka, Zookeeper, etcd — anything where replicas are *not* interchangeable.

**DaemonSet** — runs **exactly one pod on every node** (or every node matching a selector/toleration). When a node joins the cluster, the pod is added automatically. Use for: node-level agents — log collectors (Fluentd), monitoring (node-exporter), CNI pods, CSI node plugins, kube-proxy itself.

**Job** — runs pods **to completion** rather than keeping them alive. Supports `completions` (how many successful runs needed), `parallelism` (how many at once), and `backoffLimit` (retries before marking failed). A **CronJob** creates Jobs on a schedule. Use for: database migrations, batch processing, backups.

Quick memory hook: *Deployment = interchangeable, StatefulSet = identity + own storage, DaemonSet = one per node, Job = run and finish.*

---

### Q5. Readiness Probe vs Liveness Probe.

**Answer:**

Both are periodic health checks kubelet runs against a container, but they answer different questions and have completely different consequences on failure:

**Liveness probe** — "Is this container **alive**, or is it hung?"
On failure: kubelet **kills and restarts** the container. It exists to recover from deadlocks and hangs — situations where the process is running but will never make progress again, and a restart is the fix.

**Readiness probe** — "Is this container **ready to serve traffic** right now?"
On failure: the pod is **removed from the Service's endpoints** (EndpointSlice), so no new traffic is routed to it — but the container is **not restarted**. When the probe passes again, traffic returns. It exists for temporary conditions: the app is still warming up, or it lost its database connection and needs a moment.

**Startup probe** — the third one, worth mentioning: while a startup probe hasn't succeeded yet, liveness and readiness checks are suspended. It protects **slow-starting applications** (large Java services, apps doing migrations at boot) from being killed by the liveness probe before they ever finish starting.

Two practical points that show experience:
- An over-aggressive liveness probe is a classic *self-inflicted* CrashLoopBackOff: the app is briefly slow under load, liveness times out, kubelet restarts it, the restart causes more load — a death spiral. Liveness should be a cheap, always-fast check; put dependency checks in readiness instead.
- Readiness is what makes zero-downtime rolling updates work: a new pod only receives traffic after readiness passes.

---

### Q6. Kubernetes pods are Running but users receive 503 errors. What will you check?

**Answer:**

First, understand what a 503 means here: it comes from the **proxy layer** (Ingress controller / router) and it means "I have **no healthy backend** to send this request to." So the investigation is about why the traffic path is broken even though pods show Running.

Ordered checklist:

1. **`kubectl get endpoints <service>`** (or `kubectl get endpointslice`) — this is the single most informative command. If the endpoints list is **empty**, there are exactly two possible causes:
   - The **Service selector doesn't match the pod labels** — a typo means the Service selects nothing. Compare `spec.selector` on the Service with the pod labels.
   - The pods are Running but **not Ready** — readiness probes are failing. Remember: *Running is a container state; Ready is what gets you traffic.* Check `kubectl get pods` — is the READY column `0/1`?
2. **Port chain**: verify the whole path — Ingress backend port → Service `port` → Service `targetPort` → the `containerPort` the app actually listens on. Also check the app binds to `0.0.0.0`, not `127.0.0.1` — an app listening only on localhost is unreachable from outside the pod.
3. **Ingress/Route configuration**: is it pointing at the right Service name and namespace? Ingress objects are namespaced and cannot reference a Service in another namespace.
4. **NetworkPolicy**: a policy might be blocking traffic from the ingress-controller namespace to the app pods.
5. **Isolate the layer** with a test pod: `kubectl exec` into another pod and curl the **Pod IP directly** — works? The app is fine. Then curl the **ClusterIP** — fails? kube-proxy/endpoints problem. Then test through the **Ingress** — fails only there? Ingress config problem. This three-step curl cleanly separates app vs Service vs Ingress.
6. **Ingress controller logs** — they state explicitly which backend they tried and why it failed.

---

### Q7. How does Kubernetes Service Discovery work?

**Answer:**

The problem it solves: pods are ephemeral — their IPs change constantly — so applications need a stable way to find each other. Kubernetes provides two mechanisms:

**1. DNS via CoreDNS (the primary mechanism):**
- CoreDNS runs in the cluster and watches Services and Endpoints through the API server.
- Every Service automatically gets a DNS record: `<service>.<namespace>.svc.cluster.local` → its ClusterIP.
- From a pod in the *same* namespace, the short name `myservice` is enough — the search domains in the pod's `/etc/resolv.conf` expand it. Cross-namespace, you use `myservice.other-namespace`.
- **Headless Services** (`clusterIP: None`) are the special case: DNS returns the **individual Pod IPs** instead of one virtual IP. This is how StatefulSets give each replica its own address — `db-0.db.ns.svc.cluster.local` — so a client can talk to a specific replica (e.g., write to the primary).

**2. Environment variables (legacy):** kubelet injects `MYSERVICE_SERVICE_HOST`/`_PORT` variables into pods — but only for Services that existed *before* the pod started, so DNS is preferred.

**How traffic actually reaches a pod after discovery:**
- The **EndpointSlice controller** continuously maintains the list of healthy (Ready) Pod IPs behind each Service.
- **kube-proxy** on every node watches those EndpointSlices and programs **iptables or IPVS rules** that translate the ClusterIP to one of the real Pod IPs, with random load balancing.
- Important detail: the **ClusterIP is virtual** — no interface has it, it never answers ping. It exists only inside the iptables/IPVS rules on each node.

So: *CoreDNS gives you the stable name → ClusterIP; kube-proxy turns the ClusterIP into a real, healthy Pod IP at connection time.*

---

### Q8. Explain ConfigMaps and Secrets. How do you manage them across environments?

**Answer:**

**ConfigMap** — stores **non-sensitive** configuration as key-value pairs or whole files: application properties, feature flags, config files. Decouples configuration from the image, so the same image runs in every environment with different config.

**Secret** — same idea for **sensitive** data: passwords, tokens, TLS certificates. Critical honesty point for interviews: Secrets are only **base64-encoded, which is encoding, not encryption** — anyone with read access decodes them instantly. Real protection requires: **etcd encryption at rest** (an `EncryptionConfiguration` on the API server using aescbc or a KMS provider), and **RBAC** restricting who can get/list Secrets in each namespace.

**How pods consume them, and why it matters:**
- **Environment variables** — simple, but values are fixed at container start; a ConfigMap change requires a pod restart. Env vars also leak more easily (visible in `kubectl describe`, crash dumps, child processes).
- **Volume mounts** — kubelet refreshes mounted files automatically after a change (within about a minute), so an app that re-reads its config can pick up changes without restart. Exception: **subPath mounts never update**.

**Managing them across environments (dev/QA/prod):**
- Keep one base set of manifests and apply per-environment differences with **Kustomize overlays** or **Helm values files** (`values-dev.yaml`, `values-prod.yaml`). The structure stays identical; only values differ.
- **Secrets never go into git in plaintext.** Options, in increasing maturity:
  - **SealedSecrets** or **SOPS** — encrypt them so the encrypted form can safely live in git (fits GitOps).
  - **External Secrets Operator** — the cluster syncs Secrets from an external source of truth (Vault, AWS Secrets Manager) per environment; git only contains a *reference*, never the value.
- Same key names in every environment, different values — so application code never changes between environments.

---

## Section 3: CI/CD

### Q1. Explain your CI/CD pipeline from code commit to production.

**Answer:**

End to end:

1. **Commit / Pull Request** — a developer pushes; the git server fires a webhook that triggers the CI pipeline.
2. **CI — build and validate:**
   - Checkout the code.
   - **Lint + static analysis (SAST)** — fail fast on cheap checks before spending time on builds.
   - **Unit tests** with a coverage gate.
   - **Build the container image** using a multi-stage Dockerfile (build tools in the first stage, minimal runtime image as the result).
   - **Scan the image** for CVEs (Trivy) — fail on critical vulnerabilities.
   - **Sign the image** (cosign) and **push to the registry tagged with the git SHA** — an immutable tag, never `latest`, so every running container is traceable to an exact commit.
3. **Deploy to staging** — done GitOps-style: the pipeline doesn't run `kubectl apply` itself; it **commits the new image tag to a config repository**, and ArgoCD (or Flux) detects the change and syncs the cluster to match. Git is the single source of truth and the audit log.
4. **Automated verification in staging** — integration tests and smoke tests against the deployed environment.
5. **Promotion gate** — a manual approval for production, or automated promotion if canary analysis passes.
6. **Production deploy** — rolling update or canary, followed by **post-deployment verification**: health endpoints plus error-rate and latency metrics compared against the pre-deploy baseline. A breach triggers automatic rollback.

The principle that ties it together: **build once, promote the same immutable artifact everywhere**. The image tested in staging is byte-for-byte the image that reaches production — never rebuilt per environment, because a rebuild is an untested artifact.

---

### Q2. How do you implement zero-downtime deployments?

**Answer:**

Zero downtime is a stack of requirements — miss any one and requests get dropped:

1. **Rolling update strategy**: `maxUnavailable: 0` and `maxSurge: 1` — Kubernetes creates one new pod, waits for it to become Ready, and only then terminates an old one. Capacity never dips below 100%.
2. **A meaningful readiness probe**: traffic only shifts to a new pod when readiness passes, so the probe must actually verify the app can serve — check a real health endpoint, not just "TCP port open."
3. **Graceful shutdown on the old pods** — the most commonly missed part:
   - The application must handle **SIGTERM**: stop accepting new requests, finish in-flight ones, then exit.
   - Add a **`preStop` hook with a short sleep (~5 seconds)**. Reason: when a pod starts terminating, its removal from Service endpoints propagates to kube-proxy and Ingress controllers **asynchronously** — for a brief window, traffic still arrives at a pod that's shutting down. The sleep keeps the app serving through that window.
   - Set `terminationGracePeriodSeconds` longer than your worst-case request drain time.
4. **Backward-compatible database migrations** — during a rolling update, old and new code run *simultaneously* against the same database. Use the **expand → deploy → contract** pattern: first add new columns/tables (compatible with old code), then deploy the new code, then remove old structures in a later release. Never a breaking migration in the same step as the deploy.
5. **Alternatives for stricter needs**:
   - **Blue-green**: run a complete duplicate environment, switch the load balancer instantly, keep the old one for instant rollback. Costs double resources during the switch.
   - **Canary**: shift a small traffic percentage (5% → 25% → 100%) to the new version while comparing metrics — Argo Rollouts automates the analysis and aborts automatically if metrics degrade.

---

### Q3. How would you optimize a pipeline that takes 25 minutes to complete?

**Answer:**

**Step zero — measure.** Get per-stage timing and find where the 25 minutes actually go. Optimizing without measuring wastes effort on the wrong stage. Typically it's tests and image builds.

Then apply, in order of impact:

1. **Parallelize independent stages.** Lint, unit tests, and security scanning don't depend on each other — run them concurrently instead of sequentially. Often the single biggest win.
2. **Cache aggressively:**
   - **Dependency caches** (Maven/npm/pip) keyed on the lockfile — stop re-downloading the internet every run.
   - **Docker layer caching** (BuildKit, `--cache-from` with a registry-backed cache). Crucially, **order the Dockerfile correctly**: copy dependency manifests and install dependencies *before* copying the source code — then a source-only change reuses the cached dependency layer instead of rebuilding it.
3. **Split the test suite** across parallel runners (4 shards ≈ 4× faster). Additionally, on pull requests run only tests **affected by the changed files**, and keep the full suite for the main branch.
4. **Incremental builds** in monorepos — build and test only the modules the diff touches (Bazel, Nx, Turborepo, or per-directory triggers).
5. **Faster runners** — bigger CI machines, or self-hosted runners that keep warm caches and pre-pulled base images between runs.
6. **Move slow suites out of the blocking path** — full end-to-end tests run post-merge or nightly, not on every PR. The PR gate keeps fast unit/integration coverage.

Realistic outcome to quote: *25 minutes down to 8–10 without losing any coverage — the coverage just runs at smarter times.*

---

### Q4. How do you implement rollback if deployment fails?

**Answer:**

Two halves: **detecting** the failure and **executing** the rollback.

**Detection — automated, not a human watching dashboards:**
- Post-deploy verification in the pipeline: health endpoints plus key metrics (error rate, p99 latency) compared against the pre-deploy baseline for a few minutes.
- Threshold breach → rollback triggers automatically.

**Execution options:**

1. **Kubernetes native**: `kubectl rollout undo deployment/<name>`. Works because a Deployment keeps its previous ReplicaSets (controlled by `revisionHistoryLimit`) — rollback simply scales the old ReplicaSet back up. Near-instant. `--to-revision=N` for going further back.
2. **Helm**: `helm rollback <release> <revision>` — restores the entire chart state (all resources, not just the image), since Helm stores each release revision.
3. **GitOps**: `git revert` the commit that changed the image tag — ArgoCD syncs the cluster back to the previous state. Slightly slower, but the rollback itself is versioned and audited, and the cluster never diverges from git.
4. **Argo Rollouts canary**: fully automated — the canary ships to a small traffic slice, an AnalysisTemplate watches Prometheus metrics, and on failure the rollout **aborts and reverts by itself**, having only ever affected a fraction of users.

**Prerequisites that make rollback trustworthy:**
- **Immutable image tags** — rolling back re-pulls the exact previous image. With a mutable `latest` tag you cannot know what you're rolling back to.
- **Backward-compatible database migrations** — `rollout undo` reverts pods, **not the database**. If the new release ran a breaking migration, the old code won't work after rollback. This is exactly why the expand→contract migration pattern exists.

---

### Q5. How do you manage secrets securely in CI/CD pipelines?

**Answer:**

Layered approach, from baseline to best practice:

1. **Never in the repository** — not in code, not in Dockerfiles, not baked into image layers (layers are inspectable forever, even after the file is "deleted" in a later layer). Enforce it with **secret scanning** (gitleaks/truffleHog) as a CI step and pre-commit hook, so an accidentally committed credential fails the build instead of shipping.
2. **CI-native secret stores as the baseline** — GitHub Actions secrets, GitLab masked & protected variables: encrypted at rest, masked in logs, and **scoped** — protected variables only exist on protected branches, and environment scoping means a dev pipeline physically cannot read prod secrets.
3. **Short-lived dynamic credentials instead of stored static ones — the real upgrade:**
   - **OIDC federation**: the pipeline exchanges its own identity token directly with the cloud provider for temporary credentials (AWS STS / GCP WIF). **No stored cloud keys at all** — nothing to leak, nothing to rotate.
   - **Vault dynamic secrets**: the pipeline authenticates (JWT/AppRole) and receives credentials generated on demand with a TTL of minutes — expired before an attacker could reuse them.
4. **Runtime secrets don't pass through the pipeline at all**: application secrets are delivered to the cluster by **External Secrets Operator** or a Vault agent, pulled from the source of truth at deploy/run time. The pipeline only ever handles a *reference* to a secret, never the value. For build-time needs, BuildKit `--secret` mounts a credential during the build without writing it into any layer.
5. **Hygiene**: least-privilege scope per pipeline (deploy credentials can deploy — nothing more), rotation for anything static, and audit logging of every secret access.

---

## Section 4: Terraform & Cloud

### Q1. How does Terraform state locking work?

**Answer:**

**The problem:** the state file is Terraform's record of what it manages. If two people (or two CI jobs) run `apply` at the same moment, both read the same state, both make changes, and the last writer silently overwrites the other's — corrupted state, or duplicate/orphaned resources.

**The mechanism:** before any operation that could modify state, Terraform **acquires a lock in the backend**:

- With the classic **S3 backend**, the lock is an item written to a **DynamoDB table** (keyed by LockID) — S3 alone can't lock, which is why the DynamoDB table is part of the standard setup. (Newer Terraform versions also support S3-native locking via a lockfile.)
- **GCS, Consul, and Terraform Cloud** backends have locking built in.
- Purely local state uses an OS-level file lock — fine alone, useless for a team.

**Behavior:** while one run holds the lock, a second run fails immediately with "Error acquiring the state lock," including who holds it and since when. The lock is released automatically when the run finishes.

**The failure case interviewers ask about:** a run crashes mid-apply and leaves a **stale lock**. Fix: `terraform force-unlock <lock-id>` — but only after confirming no run is genuinely still in progress; force-unlocking an active run recreates exactly the corruption locking prevents.

---

### Q2. Difference between count and for_each.

**Answer:**

Both create multiple instances of a resource; the difference is **how Terraform identifies each instance**, and that difference has real destructive consequences.

**`count`** — instances are identified by **numeric position**: `aws_instance.web[0]`, `[1]`, `[2]`.
The trap: identity is positional. If you have `["a", "b", "c"]` and remove `"b"`, then `"c"` shifts from index 2 to index 1 — and Terraform sees that as *destroy the old [1] and [2], create new ones*. **Removing one item from the middle of a list destroys and recreates every resource after it.** On stateful resources that's an outage.

**`for_each`** — instances are identified by **map key or set value**: `aws_instance.web["a"]`, `["b"]`, `["c"]`.
Remove `"b"` and only `web["b"]` is destroyed; `"a"` and `"c"` are untouched, because their identity is their key, not their position.

**Practical rules:**
- Use `count` only for genuinely identical copies, or the idiom for a conditional resource: `count = var.enabled ? 1 : 0`.
- Use `for_each` whenever instances have any individual identity (names, per-env settings) — which in real infrastructure code is most of the time.
- Constraint worth mentioning: `for_each` keys must be **known at plan time** — you can't key on an attribute that only exists after apply.
- Migrating an existing resource between the two changes its state address — handle it with `moved` blocks or `terraform state mv` to avoid recreation.

---

### Q3. How do you migrate Terraform state without recreating resources?

**Answer:**

The scenario: you refactor code or reorganize projects, and without intervention Terraform sees "old address disappeared, new address appeared" = **destroy and recreate**. The tools to avoid that, by case:

**1. Renaming/refactoring within the same configuration** (rename a resource, move it into a module):
- Modern way: a **`moved` block** (Terraform ≥ 1.1) committed in the code:
  ```hcl
  moved {
    from = aws_instance.old_name
    to   = module.compute.aws_instance.new_name
  }
  ```
  The next plan shows a *move*, not destroy/create — and because it's in code, it's reviewed and works for everyone who applies.
- Older way: `terraform state mv <old> <new>` — same effect, but a manual CLI operation on the state.

**2. Moving resources between separate state files** (splitting a monolithic project):
- `terraform state mv -state-out=../other/terraform.tfstate '<addr>' '<addr>'`, or `terraform state pull`/`push` for manual surgery. Then remove the resource block from the source config and add it to the target.

**3. Bringing existing (unmanaged or externally created) infrastructure under Terraform:**
- `terraform import <address> <real-resource-id>`, or better, Terraform ≥ 1.5 **`import` blocks**, which are plannable — you see exactly what will be imported before it happens. Then write the matching configuration and iterate until `terraform plan` shows **no changes**.

**4. Migrating the backend itself** (local → S3, or between buckets):
- Change the `backend` block and run `terraform init -migrate-state` — Terraform copies the state to the new backend.

**Safety rules for all cases:** back up the state file first (`terraform state pull > backup.tfstate`), stop other runs during the migration, and the definition of done is always the same — **a plan with zero changes** afterward.

---

### Q4. What is Terraform Drift? How do you detect it?

**Answer:**

**Definition:** drift is when the **real infrastructure no longer matches what Terraform's state and configuration say** — the world changed outside of Terraform.

**How it happens:**
- Someone makes a "quick fix" in the cloud console during an incident and never backports it.
- Another automation tool or script modifies the same resources.
- The cloud provider itself changes something (auto-applied minor version upgrades, default policy changes).
- A resource is deleted manually.

**Why it's dangerous:** the next `terraform apply` will silently **revert the manual change** (possibly undoing an emergency fix), or fail unexpectedly. Your code also no longer describes reality, which defeats the point of IaC.

**Detection:**
1. **`terraform plan`** — refreshes state against reality and shows any differences as proposed changes. Any unexpected diff on an unchanged codebase = drift.
2. **`terraform plan -refresh-only`** — the purpose-built command: it shows *what changed in the real world* without proposing to fix anything. Cleanest way to answer "did anyone touch this outside Terraform?"
3. **Continuous detection**: a scheduled CI job (nightly) running `terraform plan -detailed-exitcode` — exit code 2 means "changes present," which triggers an alert. Terraform Cloud/Spacelift offer this as built-in health checks.

**Remediation — a decision per drifted resource:**
- The manual change was wrong → `terraform apply` to enforce the code.
- The manual change was right → update the configuration to match it (or `terraform apply -refresh-only` to accept reality into state), so code and world agree again.

**Prevention:** remove console write access for humans in production, route all changes through the pipeline, and keep the drift-detection job as the safety net.

---

### Q5. Explain your Terraform module structure for multiple environments.

**Answer:**

The layout I use:

```
repo/
├── modules/                  # reusable building blocks — no environment logic inside
│   ├── network/              #   vpc, subnets, routing
│   ├── compute/              #   instances / node groups
│   └── storage/
└── envs/
    ├── dev/
    │   ├── main.tf           # calls the modules with dev-sized inputs
    │   ├── backend.tf        # its own state: key = "dev/terraform.tfstate"
    │   └── terraform.tfvars
    └── prod/
        ├── main.tf
        ├── backend.tf        # key = "prod/terraform.tfstate"
        └── terraform.tfvars
```

The principles behind it:

1. **Modules are generic; environments are just inputs.** A module never contains `if env == "prod"` logic — the environment root passes different variable values (instance sizes, replica counts, CIDRs). Same code path everywhere, so dev genuinely tests what prod will run.
2. **Each environment is its own root module with its own separate state file.** This is the critical part — it gives **blast-radius isolation**: a bad apply in dev physically cannot touch prod resources, state locks don't collide across environments, and prod state access can be permissioned more strictly.
3. **Modules are versioned** (git tags or a module registry): prod pins a known-good version (`?ref=v1.4.0`) while dev tries `v1.5.0` first. Changes promote through environments like application releases.
4. **DRY for shared settings**: common provider/backend boilerplate generated or shared (Terragrunt is one option) so adding an environment doesn't mean copy-paste drift.

**Why not workspaces?** Terraform workspaces put all environments' states in the same backend, selected by an easy-to-forget CLI command — `terraform apply` while pointed at the wrong workspace is a prod incident waiting to happen, and there's no way to give prod state stricter permissions. Directory-per-environment makes the target explicit and the isolation physical. Workspaces are fine for short-lived variants (feature-branch environments), not for the dev/prod boundary.

---

### Q6. Difference between IAM Roles and IAM Policies.

**Answer:**

They answer two different questions:

**IAM Policy — "what is allowed?"**
A policy is a **JSON permission document**: statements of Effect (Allow/Deny), Action (`s3:GetObject`), Resource (which ARNs), and optional Conditions (source IP, MFA, tags). A policy by itself does nothing — it must be **attached** to an identity (user, group, or role). Explicit Deny always beats Allow.

**IAM Role — "who is acting?"**
A role is an **identity with no permanent credentials** that trusted parties can temporarily **assume**. It has two distinct parts, and naming both is what interviewers listen for:
1. A **trust policy** — *who may assume this role*: an EC2 service, a Lambda function, a specific other AWS account, or an external identity via OIDC (a Kubernetes ServiceAccount, a GitHub Actions pipeline).
2. **Permission policies** attached to it — *what the assumer can do* once inside.

When something assumes a role, **STS issues temporary credentials** (minutes to hours) that expire on their own.

**Why roles are the security backbone:** they eliminate long-lived access keys — the number-one leaked credential type. Concrete patterns:
- EC2 **instance profile**: the app on the instance gets rotating temporary credentials from metadata — no keys on disk.
- **IRSA** (IAM Roles for Service Accounts) in EKS: a specific pod's ServiceAccount maps to a specific role — per-pod cloud permissions with zero stored secrets.
- **Cross-account access**: account B's role trusts account A — no shared users.
- **CI via OIDC**: the pipeline's identity token is exchanged for role credentials — no cloud keys stored in CI at all.

One-liner: *a policy defines permissions; a role is an assumable identity those policies get attached to, used through temporary credentials instead of stored keys.*

---

### Q7. Explain VPC, Subnets, NAT Gateway, and Internet Gateway.

**Answer:**

**VPC (Virtual Private Cloud)** — your own **isolated virtual network** inside the cloud, defined by a CIDR block (e.g. `10.0.0.0/16`). Nothing gets in or out except through gateways you explicitly attach. It's the network boundary everything else lives inside.

**Subnets** — slices of the VPC's CIDR (e.g. `10.0.1.0/24`), each bound to **one availability zone**. The public/private distinction is not a checkbox — it's defined purely by routing:
- A **public subnet**'s route table has a route `0.0.0.0/0 → Internet Gateway`.
- A **private subnet**'s route table doesn't. Resources there have no direct internet path.

**Internet Gateway (IGW)** — attached to the VPC, it enables **two-way** internet connectivity for resources that have public IPs in public subnets. Inbound traffic can reach them; outbound traffic exits through it. One IGW per VPC, horizontally scaled and managed by the provider.

**NAT Gateway** — solves the private subnet's problem: instances there (app servers, databases, cluster nodes) still need **outbound** internet — pulling container images, OS updates, calling external APIs — but must never be reachable **inbound**. The NAT Gateway:
- sits **in a public subnet** (it needs the IGW itself),
- is the target of the private route tables' `0.0.0.0/0` route,
- translates the private source IPs to its own Elastic IP for outbound connections — and because NAT is stateful, only *response* traffic flows back in; unsolicited inbound is impossible.

**Standard production layout (worth drawing/stating):** per availability zone, one public subnet (load balancers, NAT GW, bastion) and one private subnet (application and database tier). One **NAT Gateway per AZ** so an AZ outage doesn't cut outbound connectivity for the surviving AZs. Traffic: internet → IGW → LB (public subnet) → app (private subnet); app's outbound → NAT GW → IGW → internet.

---

## Section 5: Monitoring & Troubleshooting

### Q1. Difference between logs, metrics, and traces.

**Answer:**

The three pillars of observability — each answers a different question, and the real skill is using them together:

**Metrics — "Is something wrong?"**
Numeric time-series: counters (requests served), gauges (memory in use), histograms (latency distribution → p50/p99). Extremely cheap to store, so you keep them long-term and alert on them. They show trends and anomalies but carry no detail about any individual event. Stack: Prometheus + Grafana.

**Logs — "Why is it wrong?"**
Discrete, timestamped event records with full context — stack traces, error messages, request parameters. Maximum detail, but expensive at volume and hard to see trends in. Structured (JSON) logs make them searchable. Stack: Loki or EFK (Elasticsearch/Fluentd/Kibana).

**Traces — "Where is it wrong?"**
The path of **one request across all services** it touched, recorded as a tree of **spans** (each span = one operation with start time and duration, linked parent→child). In a microservice architecture, a trace instantly shows which hop out of ten contributed 900ms of a 1-second request — something neither metrics nor logs of any single service can show. Context propagates between services via trace headers (W3C traceparent). Stack: OpenTelemetry instrumentation → Jaeger/Tempo.

**How they work together in an incident:**
1. A **metric** alert fires: p99 latency SLO breach.
2. **Traces** localize it: the payment-service → database span is where the time goes.
3. **Logs** of that service at those timestamps give the root cause: connection pool exhausted.

The glue that makes step 2→3 fast: inject the **trace ID into every log line**, so you jump from a slow trace straight to its exact logs.

---

### Q2. How do you investigate a sudden spike in application latency?

**Answer:**

Systematically, broad to narrow:

**1. Scope the blast radius first** — the shape of the problem points at the cause:
- **All endpoints or one?** One endpoint → that code path or its specific dependency. All → something systemic (infra, shared DB, deploy).
- **All pods or a subset?** One pod/node affected → bad node, noisy neighbor. All → application or dependency level.
- **p50 up, or only p99?** Median up = everything is slower (saturation, downstream slowness). Only tail = intermittent stalls — GC pauses, one slow replica, lock contention.

**2. Ask "what changed?" at spike time** — most incidents follow a change: deployments, config/feature-flag changes, a traffic surge (compare request rate), scheduled jobs (backups, cron) landing at that timestamp.

**3. Check resource saturation:**
- **CPU throttling** — the underrated one: check `container_cpu_cfs_throttled_periods_total`. A container can be throttled by its CPU *limit* while node CPU looks fine, adding latency invisibly.
- Memory pressure → GC frequency/duration up (JVM/Go apps).
- **Connection pool exhaustion** — requests queue waiting for a DB connection; pool wait time metrics reveal it.

**4. Follow the request downstream with traces** — open slow traces and see which span grew: the database (slow query — check `pg_stat_statements`, lock waits, missing index after data growth), an external API, a cache that started missing (Redis eviction → every miss hits the DB).

**5. Infra layer if still unexplained:** node-level pressure on the affected nodes, network issues — **conntrack table full** (drops connections silently), **DNS slowness** (the Kubernetes `ndots:5` behavior taxing external lookups), overlay/MTU problems.

**6. Mitigate before root-causing** — if a deploy correlates, roll back first; if it's load, scale out first. Restore the SLO, then dig with the pressure off.

---

### Q3. Explain your monitoring and alerting strategy.

**Answer:**

**Instrumentation — standard methods so coverage is systematic, not ad hoc:**
- **RED** per service: **R**ate (req/s), **E**rrors (failure rate), **D**uration (latency percentiles) — the user-facing health of every service.
- **USE** per resource: **U**tilization, **S**aturation (queue depth — the leading indicator; utilization can look fine while queues grow), **E**rrors — for nodes, disks, network.
- Collected by Prometheus from application `/metrics` endpoints, node-exporter, and kube-state-metrics; long-term storage and multi-cluster view via Thanos or VictoriaMetrics.

**Alerting philosophy — symptoms over causes:**
- Page on **what users experience** (error rate, latency SLO), not on internal states ("CPU 80%" — which might be perfectly fine, or a well-functioning autoscaler's normal operating point). Cause-based alerts are noisy and miss novel failures; symptom alerts catch every failure mode by definition.
- Formalized as **SLOs with error budgets**, alerting on **burn rate** with multi-window rules: a fast-burn alert (budget draining in hours → page immediately) and a slow-burn alert (draining over days → ticket). This catches both sudden outages and slow degradation without flapping.

**Severity tiers — protect the pager:**
- **Page**: user-impacting and needs a human *now*.
- **Ticket**: needs action within days — never wakes anyone.
- **Neither**: it's a dashboard. If an alert needs no action, it shouldn't exist.

**Operational hygiene:**
- Every page links a **runbook** — 3 a.m. is not the time to figure out what an alert means.
- Regular alert review: anything frequently firing without action gets fixed, tuned, or deleted. **Alert fatigue is the biggest real-world monitoring failure** — people ignore pagers that cry wolf.
- Dashboards structured top-down: user-facing SLOs on top, drill-down to service and infra beneath.

---

### Q4. Walk me through a production issue you resolved.

**Answer:**

Use the structure interviewers listen for: **Symptom → Investigation → Root cause → Fix → Prevention.** Worked example (adapt to your own strongest story):

**Symptom:** After a planned storage migration of a control-plane node to new SSD-backed volumes, the Kubernetes API became intermittently unavailable. On the affected master node, the etcd container was crash-looping.

**Investigation:** etcd logs showed **WAL (write-ahead log) corruption** — etcd refused to start with an inconsistent data directory. The data directory lives on a dedicated Cinder volume mounted at `/var/lib/etcd`. Correlating kubelet and mount timestamps from the boot after migration showed the smoking gun: **kubelet had started the etcd static pod *before* the Cinder volume finished attaching and mounting.** For a brief window, etcd wrote into the empty mountpoint directory on the root disk; when the real volume mounted over it, etcd's on-disk state was inconsistent — corrupted WAL.

**Root cause:** a race condition — no ordering dependency between the kubelet service and the etcd data mount. Nothing guaranteed the mount was up before kubelet launched static pods.

**Fix (immediate):** recovered etcd from the healthy peers — removed the corrupted member from the cluster (`etcdctl member remove`), cleaned its data directory, re-added it as a new member, and let it sync a fresh snapshot from the leader. Quorum was maintained throughout (2 of 3 healthy), so the control plane recovered without data loss.

**Prevention (permanent):** added a **systemd drop-in for kubelet** with `After=var-lib-etcd.mount` and `Requires=var-lib-etcd.mount` — kubelet now cannot start unless the etcd data mount is active, and stops if the mount fails. Baked the drop-in into control-plane node provisioning so every current and future master has it, and documented the incident in a runbook.

**Why this story works:** it shows methodical timeline correlation rather than guessing, a fix at the actual root cause (ordering) rather than the symptom (corruption), safe recovery using quorum, and prevention baked into automation.

---

## Section 6: Security & Production

### Q1. How do you secure container images before deployment?

**Answer:**

Layered — build-time hygiene, verification, and enforcement:

**1. Build the image to have a minimal attack surface:**
- **Minimal base images** — distroless, Alpine, or UBI-minimal. No shell and no package manager means far fewer CVEs and far less for an attacker to use post-compromise.
- **Multi-stage builds** — compilers, SDKs, and build tools live in the build stage; the final image contains only the binary and runtime deps.
- **Run as non-root** (`USER` directive). Root in a container + any container escape = root on the node.
- **Pin base images by digest**, not floating tags, for reproducibility.
- **No secrets in layers, ever** — layers are permanently inspectable; even a file deleted in a later layer still exists in the earlier one. Build-time credentials go through **BuildKit `--secret`** mounts, which never persist into any layer.

**2. Verify in the pipeline:**
- **CVE scanning** (Trivy/Clair/Grype) as a blocking CI step with a severity gate — critical findings fail the build. Also **rescan images already in the registry continuously**: new CVEs are published daily against old images.
- **Generate an SBOM** (software bill of materials) — when the next Log4Shell drops, you can answer "which of our images contain this library?" in minutes instead of days.
- **Sign the image** (cosign/Sigstore) so provenance is cryptographically provable.

**3. Enforce at the cluster gate — policy, not trust:**
- **Admission control** (Kyverno / OPA Gatekeeper) rejects at the API server anything that violates policy: unsigned images, images from outside the trusted private registry, `:latest` tags, privileged containers, containers running as root.
- This is the piece that turns "we scan images" into "unscanned images *cannot run*."

---

### Q2. How do you implement RBAC in Kubernetes?

**Answer:**

**The four objects and how they compose:**
- **Role** — a *namespaced* set of permissions: rules of verbs (`get`, `list`, `create`, `delete`…) on resources (`pods`, `secrets`…) in API groups. Grants only — there is no deny in Kubernetes RBAC; anything not granted is denied.
- **ClusterRole** — the same, but cluster-scoped: for cluster-wide resources (nodes, PVs), or as a reusable permission set.
- **RoleBinding** — attaches a Role (or a ClusterRole) to subjects — users, groups, or ServiceAccounts — *within one namespace*.
- **ClusterRoleBinding** — attaches a ClusterRole across the whole cluster.

**The pattern that matters:** a **RoleBinding can reference a ClusterRole**, granting its permissions *only inside that binding's namespace*. So you define a permission set like "developer" once as a ClusterRole and bind it per-namespace — no duplicated Role objects in every namespace.

**ServiceAccounts — where RBAC meets workloads:**
- Every pod runs as a ServiceAccount (default: the `default` SA of its namespace). The **default SA should have zero permissions** — apps that don't call the Kubernetes API shouldn't even mount a token (`automountServiceAccountToken: false`).
- An app that does need API access gets its **own SA** with a Role granting exactly the verbs/resources it uses — so a compromised pod's blast radius is precisely that grant, nothing more.

**Where users come from:** Kubernetes has no user database — users and groups arrive from certificates or **OIDC** (corporate IdP). Best practice: bind roles to **groups** (`team-backend` → edit in their namespaces) and manage membership in the IdP.

**Least-privilege hygiene:**
- No wildcards (`*` verbs/resources), no cluster-admin for workloads or daily human use.
- Treat `get` on Secrets as sensitive as any credential grant — that's what it is.
- Verify grants with `kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa>` and audit periodically for permission creep.

---

### Q3. How do you manage secrets in production?

**Answer:**

**Principle: the cluster is not the source of truth — an external secret manager is.**

**1. Source of truth:** HashiCorp Vault or the cloud provider's secret manager (AWS Secrets Manager / Azure Key Vault). This gives central access control, audit logging of every read, versioning, and rotation — none of which raw Kubernetes Secrets provide.

**2. Delivery into the cluster:**
- **External Secrets Operator (ESO)**: an `ExternalSecret` object references a path in Vault; the operator fetches the value and maintains a native K8s Secret in sync. Git contains only the reference — never the value — which is what makes secrets compatible with GitOps.
- **Secrets Store CSI driver**: mounts secrets straight from the external store into the pod filesystem, optionally without creating a K8s Secret at all — the value never rests in etcd.

**3. Harden the cluster layer (because K8s Secrets are only base64 — encoding, not encryption):**
- **etcd encryption at rest** — an `EncryptionConfiguration` on the API server, ideally with a **KMS provider** so the data key isn't sitting on the control-plane node.
- **RBAC**: `get`/`list` on Secrets is a credential grant — restrict it per namespace; nothing should read Secrets broadly.
- **Prefer volume mounts over env vars**: env vars leak through `kubectl describe`, crash dumps, and child processes; mounted files also support live rotation, env vars don't.

**4. Rotation:**
- Best: **dynamic secrets** — Vault generates per-pod database credentials with a TTL of hours; leaked credentials expire on their own.
- Static secrets: scheduled rotation, with the app reloading (or pods rolling) on change — ESO's refresh handles propagation.

**5. Prevention around the edges:** secret scanning in CI (gitleaks), no plaintext in git ever (SOPS/SealedSecrets where GitOps storage is needed), and audit alerts on unusual access patterns in the secret manager.

---

### Q4. How would you design a disaster recovery strategy?

**Answer:**

**Start with the two numbers everything else derives from:**
- **RPO (Recovery Point Objective)** — how much data loss is tolerable → dictates backup/replication frequency.
- **RTO (Recovery Time Objective)** — how long you may be down → dictates how much standby infrastructure you pay for.
These are **per-service business decisions**: the payment system may need minutes/near-zero; an internal reporting tool may tolerate a day. Setting them per tier is what keeps DR affordable.

**Backups — the foundation:**
- **etcd snapshots** on schedule (the cluster's entire state), **database backups** (periodic full + continuous WAL/incremental shipping for low RPO — WAL shipping gets RPO to seconds), **Velero** for Kubernetes objects + persistent volume snapshots.
- Stored **off-site/cross-region**, **encrypted**, and **immutable** (object-lock/WORM) — because ransomware targets backups first.
- **The rule that separates real DR from theater: an untested backup is not a backup.** Scheduled restore drills into an isolated environment, verifying both integrity and that the *measured* restore time actually fits the RTO.

**Redundancy tier — chosen per RTO, in increasing cost:**
1. **Backup & restore** — rebuild from backups on demand. Cheapest; RTO in hours.
2. **Pilot light** — data replicates continuously; minimal compute stands ready to scale up. RTO well under an hour.
3. **Warm standby** — a scaled-down copy runs constantly; scale up + switch traffic. RTO in minutes.
4. **Active-active multi-region** — full capacity everywhere, traffic shifts instantly. Near-zero RTO/RPO; highest cost and data-consistency complexity. Reserved for the truly critical tier.

**The IaC advantage — compute is disposable, only data needs DR:** with the entire environment defined in Terraform + GitOps configs, a full cluster rebuild is `terraform apply` + ArgoCD sync + data restore. You're not reconstructing hand-built infrastructure from memory during the worst day of the year.

**The human layer:** runbooks with exact commands, defined incident roles, and **game days** — scheduled DR exercises, because a plan executed for the first time during a real disaster will fail. Verify dependencies too: DNS failover, certificates, secrets, and container registry must all be reachable from the recovery site.

---

### Q5. How do you reduce cloud costs without affecting availability?

**Answer:**

**Step 1 — visibility before action:** cost allocation by team/service/namespace (resource tags, Kubecost for Kubernetes). Optimizing blind wastes effort; usually a few workloads dominate spend. Make teams see their own costs — behavior changes by itself.

**Step 2 — right-sizing (the biggest single win in K8s):**
- Compare **requested** vs **actually used** CPU/memory (VPA recommendations, Kubecost). Overprovisioned *requests* are the hidden cost: the scheduler reserves node capacity by requests, so a cluster can pay for nodes that are 80% "reserved" while 30% *used*.
- Same at the VM level: utilization-based instance recommendations.

**Step 3 — pay for actual load, not for peak 24/7:**
- **HPA** scales pods with demand; **cluster-autoscaler or Karpenter** scales nodes with pods (Karpenter also bin-packs and picks cheapest matching instance types).
- **Scale non-production down on a schedule** — dev/QA at nights and weekends is pure waste; that alone is commonly 20–30% of compute.

**Step 4 — pricing levers on what remains:**
- **Spot/preemptible instances** (60–90% off) for interruption-tolerant workloads: stateless services, batch, CI runners. Availability is protected by design: **PDBs**, graceful termination handling, spread across instance types, and keeping stateful/critical workloads on on-demand nodes.
- **Reserved instances / Savings Plans** for the stable baseline that runs regardless (up to ~70% off). The pattern: *reserved for the base, spot for the tolerant, on-demand for the spiky remainder.*

**Step 5 — housekeeping (pure waste, zero risk):** unattached volumes, aged snapshots, idle load balancers, unused elastic IPs — automated cleanup jobs. **Cross-AZ traffic charges** (topology-aware routing keeps chatty traffic in-zone). **Storage lifecycle tiering** — logs and backups age from hot storage to infrequent-access to archive automatically.

**Why availability survives:** every mechanism is capacity-aware — autoscalers add before saturation, PDBs floor the replica count during spot reclaims, and interruptible instances only ever carry interruptible work. Cost falls because *idle* capacity is eliminated, not *needed* capacity.

---

### Q6. Explain one challenging production incident and how you resolved it.

**Answer:**

Same structure: **Symptom → Investigation → Root cause → Fix → Prevention.** Worked example:

**Symptom:** After routine node maintenance and reboots, application pods with persistent volumes were stuck in `ContainerCreating` across **six worker nodes** simultaneously. Stateless pods scheduled and ran fine — only anything needing a volume mount was affected, which made it look like a storage outage.

**Investigation:**
- `kubectl describe pod` on stuck pods: volume attach/mount operations timing out.
- The CSI driver pods (Cinder CSI node plugin, running as a DaemonSet) on the affected nodes were **Running and healthy** — so not a crashed driver.
- **kubelet logs** on the affected nodes had the real signal: CSI **plugin registration failures with `DeadlineExceeded`** — kubelet's plugin-registration mechanism had timed out waiting for the CSI node plugin to register over its socket after the reboot, and never successfully retried. So kubelet on those nodes didn't *know* a CSI driver existed → every volume operation hung.
- Confirmed the scope matched exactly the six rebooted nodes — consistent with a reboot-sequence race, not a storage-backend problem.

**Root cause:** a race between kubelet startup and CSI node-plugin socket registration after node reboot — the registration window expired before the plugin was up, and the registration state never recovered on its own.

**Fix:** restarted kubelet on each affected node — `oc debug node/<node> -- chroot /host systemctl restart kubelet` — which re-ran plugin discovery; the CSI plugin registered immediately, pending volume attachments proceeded, and stuck pods started within minutes. Rolled through the six nodes one at a time, verifying pod recovery on each before the next, with no workload data impact.

**Prevention:**
- **Alerting on the actual failure mode**: kubelet CSI registration errors and a sustained backlog of `VolumeAttachment` objects — so next time it pages before users file tickets.
- **Runbook** documenting the signature (pods stuck ContainerCreating + healthy CSI pods + `DeadlineExceeded` in kubelet logs) and the per-node fix.
- Adjusted maintenance procedure: post-reboot health verification now includes confirming CSI registration per node before returning it to service.

**Why this story works:** the misleading initial signal (looks like storage, is actually kubelet-side), evidence-driven narrowing across layers (pod events → CSI pods → kubelet logs), a controlled low-risk fix, and prevention that catches the recurrence automatically.

---

## Self-Evaluation of This Document

**Coverage check — every question from the source screenshots:**

| Section | Questions in source | Covered | Complete question text included |
|---|---|---|---|
| Linux & Networking | 5 | 5 ✅ | Yes — verbatim as headers |
| Docker & Kubernetes | 8 | 8 ✅ | Yes |
| CI/CD | 5 | 5 ✅ | Yes |
| Terraform & Cloud | 7 | 7 ✅ | Yes |
| Monitoring & Troubleshooting | 4 | 4 ✅ | Yes |
| Security & Production | 6 | 6 ✅ | Yes |
| **Total** | **35** | **35 ✅** | |

**Quality check against interview needs:**

1. **Control plane vs data plane** — Q1.1 now explicitly places the API server and etcd outside the user-traffic path, with etcd's role (cluster database behind the API server) stated. Previous gap closed.
2. **Depth level** — each answer is a complete spoken-interview answer: mechanism, the *why*, and at least one practical detail that signals hands-on experience (exit codes, `ndots:5`, `lsof +L1`, preStop sleep, expand→contract migrations, DynamoDB lock table, IRSA/OIDC, burn-rate alerting).
3. **Readability** — every answer is self-contained; no answer assumes another was read first. Numbered flows for processes, bold for the load-bearing terms, one-line summaries where a crisp closer helps in a live interview.
4. **Experience questions (5.4 and 6.6)** — structured as Symptom → Investigation → Root cause → Fix → Prevention with fully worked examples drawn from your real etcd WAL and Cinder CSI incidents; each ends with *why the story works*, so you can adapt the frame to any other incident.
5. **Honest gaps to be aware of before the interview:**
   - Answers are cloud-generic with AWS terminology in the IAM/VPC questions (matching the source questions). If the interviewer is Azure/GCP-focused, translate: IAM Role→Managed Identity/Service Account, VPC→VNet, NAT GW→NAT Gateway (same concept).
   - Q3.1 (CI/CD pipeline) and Q5.3 (monitoring strategy) describe a reference architecture — before the interview, map each stage to the *specific tools you actually used* (your Jira/Prometheus/VictoriaMetrics tooling) so it sounds lived-in, not textbook.
   - For Q5.4/Q6.6, rehearse the two incident stories out loud once — the structure is on paper, but delivery timing (2–3 minutes each) needs one practice pass.

**Verdict: ready as a pre-interview reference.** Suggested use: skim section headers day-before; re-read only Sections 1–2 (most commonly probed) plus your two incident stories on interview morning.
