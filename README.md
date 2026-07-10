# Disconnected UPI — Quick Reference

## Prep (before any node boots)

| Stage | File | Action |
|---|---|---|
| DNS | zone file (e.g. `/var/named/example.com.zone`) | EDIT — A/PTR records for api, api-int, `*.apps`, bootstrap, masters, workers |
| LB | `haproxy.cfg` | EDIT — frontends/backends for 6443, 22623, 80, 443 |
| Mirror registry | `imageset-config.yaml` | CREATE — defines what oc-mirror pulls |
| Mirror registry | `rootCA.pem` (or similar) | OUTPUT — registry's CA cert, needed later |
| Install config | `install-config.yaml` | EDIT — add `imageContentSources` + `additionalTrustBundle` (paste CA cert), `platform: none`, `compute.replicas: 0` |
| Backup | `install-config.yaml.bak` | COPY — before `create manifests` consumes the original |
| Manifests | `manifests/cluster-scheduler-02-config.yml` | EDIT — `mastersSchedulable: false` |
| Ignition | `bootstrap.ign`, `master.ign`, `worker.ign` | GENERATED — never hand-edited |
| Web server | `httpd.conf` / vhost | EDIT — set `DocumentRoot` |
| Web server | ignition files | COPY into DocumentRoot, restart Apache |

## Boot-to-running (no files touched, just node actions)

1. Bootstrap boots → fetches `bootstrap.ign` → Ignition writes systemd units
2. `release-image.service` pulls release image from mirror → `bootkube.service` runs
3. Temp control plane + etcd stand up on bootstrap
4. Masters boot with `master.ign` → take over → `wait-for bootstrap-complete`
5. Remove bootstrap from LB
6. Workers boot with `worker.ign` → approve CSRs (`oc get csr` / `oc adm certificate approve`)
7. Image registry storage set (PVC/StorageClass patch on `configs.imageregistry.operator.openshift.io`)
8. `wait-for install-complete` → `oc get co` all Available

> **One-liner to remember:** you edit config/text files (haproxy, DNS, install-config, scheduler manifest, httpd.conf) — you never hand-edit the `.ign` files themselves, only generate and copy them.

---

# Quick Reference: OpenShift UPI Installation in a Disconnected Environment

**Target:** OpenShift UPI bare-metal/VM install with no internet access inside the cluster network. In UPI, you provision the machines, DNS, load balancer, RHCOS boot, ignition hosting, and registry yourself. Red Hat docs state that UPI requires you to deploy all required machines, including the temporary bootstrap and control-plane hosts. ([Red Hat Documentation][1])

---

## 1. Required components

```
Helper / Bastion VM
├── DNS             -> api, api-int, *.apps, node records
├── HTTP server     -> ignition + RHCOS files
├── HAProxy/LB      -> API + Ingress load balancing
├── Mirror registry -> OpenShift release images
└── DHCP/PXE        -> optional, if used
```

Minimum cluster:

```
1 bootstrap node     temporary
3 control-plane nodes
2+ worker nodes      recommended
```

In disconnected installs, OpenShift must pull from an internal mirror registry. Red Hat recommends oc-mirror as the preferred mirroring method, while `oc adm release mirror` can mirror release/catalog content directly. ([Red Hat Documentation][2])

---

## 2. Example IP plan

```
Cluster name: lab
Base domain:  ocp.lan

Helper VM:       192.168.22.1
API VIP:         192.168.22.10
Ingress VIP:     192.168.22.11

Bootstrap:       192.168.22.200
Master-1:        192.168.22.201
Master-2:        192.168.22.202
Master-3:        192.168.22.203
Worker-1:        192.168.22.211
Worker-2:        192.168.22.212

Registry:        registry.ocp.lan:8443
```

---

## 3. DNS records

Forward records:

```dns
api.lab.ocp.lan          A   192.168.22.10
api-int.lab.ocp.lan      A   192.168.22.10
*.apps.lab.ocp.lan       A   192.168.22.11

ocp-bootstrap.lab.ocp.lan A  192.168.22.200
ocp-cp-1.lab.ocp.lan      A  192.168.22.201
ocp-cp-2.lab.ocp.lan      A  192.168.22.202
ocp-cp-3.lab.ocp.lan      A  192.168.22.203
ocp-w-1.lab.ocp.lan       A  192.168.22.211
ocp-w-2.lab.ocp.lan       A  192.168.22.212
```

Test:

```bash
dig @192.168.22.1 api.lab.ocp.lan +short
dig @192.168.22.1 api-int.lab.ocp.lan +short
dig @192.168.22.1 test.apps.lab.ocp.lan +short
dig @192.168.22.1 ocp-cp-1.lab.ocp.lan +short
dig -x 192.168.22.201 @192.168.22.1 +short
```

Expected:

```
api/api-int     -> API VIP
*.apps          -> Ingress VIP
node names      -> node IPs
reverse lookup  -> node FQDN
```

> **No DNS, no install.** Do not proceed until this is clean.

---

## 4. Load balancer ports

For bootstrap phase:

```
API VIP 6443       -> bootstrap + masters
API VIP 22623      -> bootstrap + masters
Ingress VIP 80/443  -> workers/control-plane nodes running router pods
```

After bootstrap completes:

```
Remove bootstrap from 6443 and 22623 backends.
Power off/remove bootstrap node.
```

HAProxy example:

```haproxy
frontend api_6443
  bind *:6443
  default_backend api_6443_backend

backend api_6443_backend
  balance roundrobin
  server bootstrap 192.168.22.200:6443 check
  server master1   192.168.22.201:6443 check
  server master2   192.168.22.202:6443 check
  server master3   192.168.22.203:6443 check

frontend mcs_22623
  bind *:22623
  default_backend mcs_22623_backend

backend mcs_22623_backend
  balance roundrobin
  server bootstrap 192.168.22.200:22623 check
  server master1   192.168.22.201:22623 check
  server master2   192.168.22.202:22623 check
  server master3   192.168.22.203:22623 check

frontend ingress_443
  bind *:443
  default_backend ingress_443_backend

backend ingress_443_backend
  balance roundrobin
  server worker1 192.168.22.211:443 check
  server worker2 192.168.22.212:443 check

frontend ingress_80
  bind *:80
  default_backend ingress_80_backend

backend ingress_80_backend
  balance roundrobin
  server worker1 192.168.22.211:80 check
  server worker2 192.168.22.212:80 check
```

---

## 5. Mirror OpenShift release images

Set variables:

```bash
export OCP_RELEASE=4.20.0
export ARCHITECTURE=x86_64
export LOCAL_REGISTRY=registry.ocp.lan:8443
export LOCAL_REPOSITORY=ocp4/openshift4
export PRODUCT_REPO=openshift-release-dev
export RELEASE_NAME=ocp-release
export LOCAL_SECRET_JSON=/root/pull-secret.json
```

Mirror release payload:

```bash
oc adm release mirror -a ${LOCAL_SECRET_JSON} \
  --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
  --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
  --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}
```

Save the output. You need this section for `install-config.yaml`:

```yaml
imageContentSources:
- mirrors:
  - registry.ocp.lan:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.ocp.lan:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

For full disconnected lifecycle including Operators, use oc-mirror v2. Red Hat documentation says oc-mirror is the single preferred tool for mirroring OpenShift content and Operators, while the registry must remain available while the cluster runs. ([Red Hat Documentation][3])

---

## 6. Create install-config.yaml

```bash
mkdir -p /root/ocp4-metal-install
cd /root/ocp4-metal-install
```

Create:

```bash
openshift-install create install-config --dir=/root/ocp4-metal-install
```

Then edit:

```bash
vi /root/ocp4-metal-install/install-config.yaml
```

Example:

```yaml
apiVersion: v1
baseDomain: ocp.lan
metadata:
  name: lab

compute:
- name: worker
  replicas: 0

controlPlane:
  name: master
  replicas: 3

platform:
  none: {}

networking:
  networkType: OVNKubernetes
  machineNetwork:
  - cidr: 192.168.22.0/24
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16

pullSecret: '<MERGED_PULL_SECRET_WITH_MIRROR_REGISTRY_AUTH>'

sshKey: |
  ssh-rsa AAAA...

additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  YOUR-REGISTRY-CA-CERT
  -----END CERTIFICATE-----

imageContentSources:
- mirrors:
  - registry.ocp.lan:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.ocp.lan:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

For restricted installs, Red Hat docs require the mirror registry info and registry certificate/trust bundle in the install configuration. ([Red Hat Documentation][4])

Backup it now:

```bash
cp install-config.yaml install-config.yaml.bak
```

---

## 7. Generate manifests and ignition

```bash
openshift-install create manifests --dir=/root/ocp4-metal-install
```

For normal worker-based scheduling:

```bash
sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' \
  /root/ocp4-metal-install/manifests/cluster-scheduler-02-config.yml
```

Generate ignition files:

```bash
openshift-install create ignition-configs --dir=/root/ocp4-metal-install
```

You should now have:

```
bootstrap.ign
master.ign
worker.ign
metadata.json
auth/kubeconfig
auth/kubeadmin-password
```

---

## 8. Host ignition and RHCOS files

Example HTTP root:

```bash
mkdir -p /var/www/html/ocp4
cp /root/ocp4-metal-install/*.ign /var/www/html/ocp4/
```

Copy RHCOS files:

```bash
cp rhcos-live-kernel-x86_64 /var/www/html/ocp4/
cp rhcos-live-initramfs.x86_64.img /var/www/html/ocp4/
cp rhcos-metal.x86_64.raw.gz /var/www/html/ocp4/
```

Start HTTP server:

```bash
cd /var/www/html
python3 -m http.server 8080
```

Verify:

```bash
curl -I http://192.168.22.1:8080/ocp4/bootstrap.ign
curl -I http://192.168.22.1:8080/ocp4/master.ign
curl -I http://192.168.22.1:8080/ocp4/worker.ign
curl -I http://192.168.22.1:8080/ocp4/rhcos-metal.x86_64.raw.gz
```

Expected:

```
HTTP/1.0 200 OK
```

---

## 9. Boot RHCOS nodes

Bootstrap node:

```
coreos.inst=yes
coreos.inst.install_dev=sda
coreos.inst.image_url=http://192.168.22.1:8080/ocp4/rhcos-metal.x86_64.raw.gz
coreos.inst.ignition_url=http://192.168.22.1:8080/ocp4/bootstrap.ign
```

Master nodes:

```
coreos.inst=yes
coreos.inst.install_dev=sda
coreos.inst.image_url=http://192.168.22.1:8080/ocp4/rhcos-metal.x86_64.raw.gz
coreos.inst.ignition_url=http://192.168.22.1:8080/ocp4/master.ign
```

Worker nodes:

```
coreos.inst=yes
coreos.inst.install_dev=sda
coreos.inst.image_url=http://192.168.22.1:8080/ocp4/rhcos-metal.x86_64.raw.gz
coreos.inst.ignition_url=http://192.168.22.1:8080/ocp4/worker.ign
```

With static IP example:

```
rd.neednet=1 ip=192.168.22.201::192.168.22.1:255.255.255.0:ocp-cp-1.lab.ocp.lan:ens192:none nameserver=192.168.22.1
```

Main rule:

```
bootstrap -> bootstrap.ign
masters   -> master.ign
workers   -> worker.ign
```

---

## 10. Watch bootstrap

```bash
export KUBECONFIG=/root/ocp4-metal-install/auth/kubeconfig
```

Wait:

```bash
openshift-install wait-for bootstrap-complete \
  --dir=/root/ocp4-metal-install \
  --log-level=debug
```

When bootstrap completes:

```
1. Remove bootstrap from HAProxy backends.
2. Power off/delete bootstrap VM.
3. Keep masters behind 6443 and 22623.
```

---

## 11. Approve worker CSRs

```bash
oc get csr
```

For lab/POC:

```bash
oc get csr -o name | xargs -r oc adm certificate approve
```

Watch:

```bash
watch -n 5 'oc get csr; echo; oc get nodes -o wide'
```

Workers usually need two CSR rounds:

```
1st CSR -> kubelet client cert
2nd CSR -> kubelet serving cert
```

---

## 12. Wait for install complete

```bash
openshift-install wait-for install-complete \
  --dir=/root/ocp4-metal-install \
  --log-level=debug
```

Get credentials:

```bash
cat /root/ocp4-metal-install/auth/kubeadmin-password
export KUBECONFIG=/root/ocp4-metal-install/auth/kubeconfig
```

Validate:

```bash
oc get nodes -o wide
oc get co
oc get clusterversion
oc get pods -A | grep -v Running | grep -v Completed
```

Expected:

```
All nodes Ready
All cluster operators Available=True
ClusterVersion Available=True
```

---

## 13. Post-install disconnected validation

Check image mirror config:

```bash
oc get image.config.openshift.io/cluster -o yaml
```

Check registry pull from node:

```bash
oc debug node/$(oc get nodes -o name | head -n1 | cut -d/ -f2) -- chroot /host bash -c '
cat /etc/containers/registries.conf
ls -l /etc/pki/ca-trust/source/anchors/
'
```

Check marketplace:

```bash
oc get catalogsource -n openshift-marketplace
oc get pods -n openshift-marketplace
```

For oc-mirror v2 content, apply generated cluster resources after install:

```bash
oc apply -f working-dir/cluster-resources/
```

Then verify:

```bash
oc get imagedigestmirrorset
oc get imagetagmirrorset
oc get catalogsource -n openshift-marketplace
```

Red Hat documents these generated resources as the expected post-mirroring validation objects for disconnected clusters. ([Red Hat Documentation][3])

---

## 14. Common failure points

| Symptom | Root cause |
|---|---|
| Bootstrap waits forever | DNS wrong |
| Masters fail to join | `api-int` missing |
| Image pull x509 error | Registry CA missing |
| Unauthorized image pull | Pull secret wrong |
| Bootstrap never completes | Bootstrap not in LB |
| Install stuck or unstable API routing | Bootstrap not removed |
| Ignition/MCS failures | 22623 blocked |
| RHCOS install fails | HTTP path wrong |
| Node joins wrong role or fails | Wrong ignition file |
| Workers stay NotReady | CSRs not approved |

Fast checks:

```bash
dig @192.168.22.1 api.lab.ocp.lan +short
dig @192.168.22.1 api-int.lab.ocp.lan +short
curl -k https://api.lab.ocp.lan:6443/version
curl -I http://192.168.22.1:8080/ocp4/master.ign
curl -k https://registry.ocp.lan:8443/v2/
oc get csr
oc get nodes
oc get co
```

---

## Final flow

```
1.  Prepare helper VM
2.  Configure DNS
3.  Configure LB / HAProxy / keepalived
4.  Configure mirror registry
5.  Mirror OCP release images
6.  Create install-config.yaml with pull secret, CA, imageContentSources
7.  Generate manifests
8.  Generate ignition files
9.  Host ignition + RHCOS files over HTTP
10. Boot bootstrap with bootstrap.ign
11. Boot masters with master.ign
12. Wait for bootstrap-complete
13. Remove bootstrap
14. Boot workers with worker.ign
15. Approve CSRs
16. Wait for install-complete
17. Validate COs, nodes, image pulls, routes
```

This is the UPI disconnected install skeleton. Every failure usually comes from **DNS, LB, registry trust, pull secret, or wrong ignition**.

[1]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_bare_metal/user-provisioned-infrastructure "Chapter 2. User-provisioned infrastructure | Installing on bare metal | OpenShift Container Platform | 4.20 | Red Hat Documentation"
[2]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/disconnected_environments/installing-mirroring-disconnected-about "Chapter 3. About disconnected installation mirroring | Disconnected environments | OpenShift Container Platform | 4.20 | Red Hat Documentation"
[3]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/disconnected_environments/index "Disconnected environments | OpenShift Container Platform | 4.20 | Red Hat Documentation"
[4]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/installing_an_on-premise_cluster_with_the_agent-based_installer/index "Installing an on-premise cluster with the Agent-based Installer | OpenShift Container Platform | 4.20 | Red Hat Documentation"
