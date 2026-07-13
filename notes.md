## Disconnected UPI — Quick Reference

**Prep (before any node boots):**

| Stage | File | Action |
|---|---|---|
| DNS | zone file (e.g. `/var/named/example.com.zone`) | EDIT — A/PTR records for api, api-int, *.apps, bootstrap, masters, workers |
| LB | `haproxy.cfg` | EDIT — frontends/backends for 6443, 22623, 80, 443 |
| Mirror registry | `imageset-config.yaml` | CREATE — defines what oc-mirror pulls |
| Mirror registry | `rootCA.pem` (or similar) | OUTPUT — registry's CA cert, needed later |
| Install config | `install-config.yaml` | EDIT — add `imageContentSources` + `additionalTrustBundle` (paste CA cert), `platform: none`, `compute.replicas: 0` |
| Backup | `install-config.yaml.bak` | COPY — before `create manifests` consumes the original |
| Manifests | `manifests/cluster-scheduler-02-config.yml` | EDIT — `mastersSchedulable: false` |
| Ignition | `bootstrap.ign`, `master.ign`, `worker.ign` | GENERATED — never hand-edited |
| Web server | `httpd.conf`/vhost | EDIT — set `DocumentRoot` |
| Web server | ignition files | COPY into DocumentRoot, restart Apache |

**Boot-to-running (no files touched, just node actions):**

1. Bootstrap boots → fetches `bootstrap.ign` → Ignition writes systemd units
2. `release-image.service` pulls release image from mirror → `bootkube.service` runs
3. Temp control plane + etcd stand up on bootstrap
4. Masters boot with `master.ign` → take over → `wait-for bootstrap-complete`
5. Remove bootstrap from LB
6. Workers boot with `worker.ign` → approve CSRs (`oc get csr` / `oc adm certificate approve`)
7. Image registry storage set (PVC/StorageClass patch on `configs.imageregistry.operator.openshift.io`)
8. `wait-for install-complete` → `oc get co` all Available

**One-liner to remember:** you edit config/text files (haproxy, DNS, install-config, scheduler manifest, httpd.conf) — you never hand-edit the `.ign` files themselves, only generate and copy them.

for  PXEboots we need to have different port and for cluster different

agent-config will have randezvous IP (Boot starp node IP) +
all other VM details including names, macid network and IP

openshift agent create image 

satellite
DNS forward and reverse look up should be done
all necessary redhat limks should be opened in proxy
all necessary ports should be opened in firewall 
then register with subsription-manager register 
disbaled default repositories
enable specif repos for sattellite 
update the system "dnf upgrade" 
dnf install satellite -v
sattelite-installer --senario satellite 
navigate to url 
we can group hosts for patching with specific filter with relevant errata 
there is an option to run prescript and post script

HAproxy
apt install haproxy
etc hapry cong

frontend stats
 mode tcp (ip and port)
		http (balancing with diffenert app endpont)
 default backend   myserver
backend  myserver
  server  node1  ip:port
  
reload haproxy
  
 
## OpenShift network layers

| Layer | What it is | CIDR (your cluster) | Who facilitates it | Purpose |
|---|---|---|---|---|
| **Node network** | Real IPs on the LAN | 192.168.0.0/24 | OS + `br-ex` OVS bridge | Node-to-node, kubelet ↔ API |
| **Pod network** | Virtual IPs for pods | 10.128.0.0/14 | OVN-Kubernetes + Multus + Geneve tunnels | Pod-to-pod across nodes |
| **Service network** | Stable ClusterIPs | 172.30.0.0/16 | kube-proxy / OVN load balancers | Fixed IP for a group of pods |
| **Ingress (apps)** | External VIP for app traffic | 192.168.0.7 | keepalived + router pod (HAProxy) | Outside world → app1 |
| **API access** | External VIP for cluster admin | api-int VIP :6443 | keepalived + HAProxy on masters | `oc` / `kubectl` → kube-apiserver |

Each layer rides on the one above it. 

## Networking cast — think of, and *why we need it*

| Term | Think of it as… | Why we need it |
|---|---|---|
| **OS + kernel** | The **road** — the real network. | Without it, packets have nowhere to physically travel. |
| **OVS** | A **software switch** in each node. | Real switches can't reach inside a node; we need one *inside* to connect pods. |
| **OVN** | The **brain** behind OVS. | OVS by itself is dumb muscle — OVN gives it a cluster-wide plan. |
| **OVN-Kubernetes** | The **translator** — Kubernetes ↔ OVN. | OVN doesn't understand "Pod" or "Service"; someone has to convert. |
| **Multus** | The **gatekeeper** at the pod's door. | Some pods need multiple networks (e.g. storage, SR-IOV); Multus lets that happen. |
| **Geneve** | The **envelope** wrapping pod packets. | Pod IPs (10.x) aren't routable on the LAN — wrap them so the LAN can carry them. |
| **kube-proxy** | The **redirector**, Service IP → pod IP. | Pods die and change IPs; Services give a *stable* address in front of them. |
| **CoreDNS** | The **cluster phonebook**. | So apps can call `mysql-svc` by name instead of memorising IPs. |
| **HAProxy** | The **traffic cop**. | To spread load across healthy backends and skip dead ones. |
| **keepalived** | The **on-call rota** for a VIP. | So the entry-point IP never dies with one node — someone else grabs it. |
| **Router pod** | The **reception desk for apps**. | So `app1.apps...` and `app2.apps...` both hit port 443 but reach the right app. |

## Whole story in one line, with the "why"
**keepalived** keeps the door alive → **HAProxy / router pod** picks a healthy destination → **kube-proxy** hides pod churn behind a stable Service IP → **OVN-K8s → OVN → OVS** builds the virtual network pods live on → **Geneve** carries their traffic over the real LAN → **Multus** ensures every pod has the right cables → **CoreDNS** lets everyone find each other by name.

Each piece exists to solve a problem the layer below couldn't.
