# EX280 Daily Revision Guide

**OpenShift 4.18 practice - task, resolution and verification**

This guide restructures the two supplied DO280/EX280 practice PDFs into a repeatable exam workflow. The source documents contain naming inconsistencies and command typos, so the commands below are normalized. During the exam, the **question and the live cluster are the source of truth**.

## How to use this guide every day

1. Read only the **Memory hook**.
2. Hide the Resolution section and type the commands from memory.
3. Run every Verification command. Configuration without proof is incomplete.
4. Mark any task you cannot complete in 5 minutes and repeat it the next day.

## The exam workflow to memorize

```text
READ -> SCOPE -> INSPECT -> CHANGE -> WAIT -> VERIFY CONFIG -> VERIFY FUNCTION
```

```bash
# 1. Identity and scope
oc whoami
oc project

# 2. Inspect before changing
oc get <resource> -n <project>
oc get <resource> <name> -n <project> -o yaml
oc describe <resource> <name> -n <project>
oc get events -n <project> --sort-by=.lastTimestamp | tail -30

# 3. Change the smallest possible object
oc create ...
oc set ...
oc patch ...
oc edit ...

# 4. Wait for reconciliation
oc rollout status deployment/<name> -n <project>
watch oc get pods -n <project>

# 5. Verify configuration
oc get <resource> <name> -n <project> -o yaml
oc auth can-i ...

# 6. Verify function
oc get endpoints -n <project>
oc logs -n <project> <pod>
curl -s http[s]://<route-host>
```

## Verification command map

| Requirement | Strong verification |
|---|---|
| RBAC | `oc auth can-i <verb> <resource> --as=<user> -n <project>` |
| Group membership | `oc get group <name> -o yaml` |
| Rollout | `oc rollout status deployment/<name>` |
| Service routing | `oc get endpoints <service>` |
| Route output | `curl -s http[s]://<host>` |
| Pod failure | `oc describe pod` + sorted events + logs |
| SCC | Pod `openshift.io/scc` annotation |
| Secret/ConfigMap injection | Deployment `env`/`envFrom` or `secretKeyRef` |
| HPA | `oc describe hpa` |
| Storage | PV/PVC both `Bound` + workload mount |
| Operator | CSV phase `Succeeded` |
| Exact object state | `oc get ... -o yaml` or JSONPath |

## Critical exam rules

- **Never guess the kind:** run `oc get deployment,deploymentconfig` first.
- **Never guess labels:** run `oc get pods --show-labels` first.
- **Never stop at object creation:** verify the object, pod rollout and application output.
- **Do not add/remove components** when a troubleshooting task forbids it.
- **Use exact names, namespaces, units, labels, ports, keys and hostnames.**
- Text inside `<angle-brackets>` is a placeholder; replace it before running the command.
- **Delete kubeadmin only at the end** if explicitly required.

---
## 1. HTPasswd identity provider

**Memory hook:** File -> Secret -> OAuth -> Authentication rollout -> Login test.

### Inspect

```bash
oc whoami
oc get oauth cluster -o yaml
oc get secret -n openshift-config | grep -i idp
which htpasswd
```

### Resolution

```bash
API=https://api.ocp4.example.com:6443

# Only the first user uses -c. -B enables bcrypt.
htpasswd -c -B -b htpasswd.users armstrong indionce
htpasswd    -B -b htpasswd.users collins veraster
htpasswd    -B -b htpasswd.users jobs sestiver

oc create secret generic ex280-idp-secret \
  --from-file=htpasswd=htpasswd.users \
  -n openshift-config

oc get oauth cluster -o yaml > oauth.yaml
vi oauth.yaml

# Add this under spec.identityProviders. Preserve existing providers.
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: ex280-idp-secret
    mappingMethod: claim
    name: ex280-htpasswd
    type: HTPasswd

oc replace -f oauth.yaml
```

### Verification

```bash
oc get oauth cluster -o yaml
watch oc get pods -n openshift-authentication

oc login -u armstrong -p indionce "$API"
oc whoami

# Return to the administrator identity after testing.
oc login -u kubeadmin -p '<password>' "$API"
```

### Traps

- Using -c for every user overwrites the password file.
- Do not replace an existing identityProviders list accidentally.
- The secret key must be named htpasswd.

---

## 2. Cluster permissions and project self-provisioning

**Memory hook:** Cluster role = oc adm policy. Prove access with oc auth can-i.

### Inspect

```bash
oc get clusterrolebinding self-provisioners -o yaml
oc adm policy who-can create projectrequests.project.openshift.io
oc adm policy who-can get nodes
```

### Resolution

```bash
# Cluster administrator permission
oc adm policy add-cluster-role-to-user cluster-admin jobs

# Stop automatic restoration before changing the default binding.
oc annotate clusterrolebinding self-provisioners \
  rbac.authorization.kubernetes.io/autoupdate=false --overwrite

# Remove project creation from all OAuth users.
oc adm policy remove-cluster-role-from-group \
  self-provisioner system:authenticated:oauth

# Allow only the requested user to create projects.
oc adm policy add-cluster-role-to-user self-provisioner wozniak
```

### Verification

```bash
oc auth can-i get nodes --as=jobs
oc auth can-i create projectrequests.project.openshift.io --as=wozniak
oc auth can-i create projectrequests.project.openshift.io --as=armstrong
oc auth can-i '*' '*' --all-namespaces --as=wozniak

oc get clusterrolebinding self-provisioners -o yaml
```

### Note

```text
Final destructive step, only when explicitly required:
oc delete secret kubeadmin -n kube-system
oc get secret kubeadmin -n kube-system
```

### Traps

- The correct removal verb is remove-cluster-role-from-group.
- Do not grant cluster-admin merely to let a user create projects.
- Delete kubeadmin only after all work and verification are complete.

---

## 3. Projects and user RBAC

**Memory hook:** Create projects -> bind roles in the correct namespace -> test allowed and denied actions.

### Inspect

```bash
oc get projects
oc get rolebinding -n apollo
oc auth can-i --list -n apollo --as=armstrong
```

### Resolution

```bash
for p in apollo manhattan gemini bluebook titan; do
  oc get project "$p" >/dev/null 2>&1 || oc new-project "$p"
done

oc adm policy add-role-to-user admin armstrong -n apollo
oc adm policy add-role-to-user admin armstrong -n gemini
oc adm policy add-role-to-user view  wozniak  -n titan
```

### Verification

```bash
oc get rolebinding -n apollo
oc get rolebinding -n gemini
oc get rolebinding -n titan

oc auth can-i create deployments -n apollo --as=armstrong
oc auth can-i get pods -n titan --as=wozniak
oc auth can-i delete deployments -n titan --as=wozniak
```

### Traps

- A namespaced role binding without -n is usually placed in the wrong project.
- Verify one allowed action and one forbidden action.

---

## 4. Groups and group RBAC

**Memory hook:** Create group -> add users -> bind role to group -> test a member.

### Inspect

```bash
oc get groups
oc get rolebinding -n apollo
```

### Resolution

```bash
oc adm groups new commander
oc adm groups new pilot

oc adm groups add-users commander armstrong
oc adm groups add-users pilot collins aldrin

oc adm policy add-role-to-group edit commander -n apollo
oc adm policy add-role-to-group view pilot -n apollo
```

### Verification

```bash
oc get group commander -o yaml
oc get group pilot -o yaml
oc get rolebinding -n apollo

oc auth can-i update deployments -n apollo --as=armstrong
oc auth can-i get pods -n apollo --as=collins
oc auth can-i update deployments -n apollo --as=collins
```

### Traps

- User membership and role binding are separate steps.
- Do not bind the role to individual users when the question asks for a group.

---

## 5. ResourceQuota

**Memory hook:** Read the wording: limits.* controls limits; requests.* controls requests; object counts use resource names.

### Inspect

```bash
oc get quota -n manhattan
oc describe quota -n manhattan
```

### Resolution

```bash
oc create quota ex280-quota \
  --hard=limits.cpu=2,limits.memory=160Mi,pods=3,services=6,replicationcontrollers=3 \
  -n manhattan
```

### Verification

```bash
oc get resourcequota ex280-quota -n manhattan -o yaml
oc describe quota ex280-quota -n manhattan
```

### Note

```text
Useful key mapping:
- Container limits: limits.cpu, limits.memory
- Container requests: requests.cpu, requests.memory
- Counts: pods, services, secrets, configmaps, replicationcontrollers
```

### Traps

- memory and cpu can mean requests; use limits.memory and limits.cpu when the question explicitly says container limits.
- Copy Mi/Gi and m/core values exactly.
- A quota may block pods that do not declare required requests or limits.

---

## 6. Manual application scaling

**Memory hook:** Set desired replicas -> wait for rollout -> compare desired and ready.

### Inspect

```bash
oc get deployment,deploymentconfig -n gru
oc get pods -n gru
```

### Resolution

```bash
oc scale deployment/minion --replicas=5 -n gru
```

### Verification

```bash
oc rollout status deployment/minion -n gru
oc get deployment minion -n gru \
  -o jsonpath='{.spec.replicas}{" desired / "}{.status.readyReplicas}{" ready
"}'
oc get pods -n gru
```

### Traps

- Do not assume DeploymentConfig. Check whether the existing object is a Deployment or DeploymentConfig.
- Five desired replicas is not enough; verify five ready replicas.

---

## 7. Horizontal Pod Autoscaler

**Memory hook:** CPU request first -> HPA second -> verify target, min, max and metrics.

### Inspect

```bash
oc get deployment hydra -n lerna -o yaml
oc get hpa -n lerna
oc top pods -n lerna
```

### Resolution

```bash
oc set resources deployment/hydra \
  --requests=cpu=25m --limits=cpu=180m \
  -n lerna

oc autoscale deployment/hydra \
  --min=1 --max=9 --cpu-percent=60 \
  -n lerna
```

### Verification

```bash
oc get hpa hydra -n lerna
oc describe hpa hydra -n lerna
oc get deployment hydra -n lerna \
  -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
```

### Traps

- CPU utilization HPA requires a CPU request.
- TARGETS may show <unknown> briefly while metrics become available.
- Do not set only the CPU limit.

---

## 8. Secure edge route

**Memory hook:** Edge route = TLS from client to router; plain HTTP from router to service.

### Inspect

```bash
oc get route -n area51
oc get service,endpoints -n area51
ls -l certificate.crt private.key
```

### Resolution

```bash
oc delete route oxcart -n area51 --ignore-not-found

oc create route edge oxcart \
  --service=oxcart \
  --hostname=classified.apps.ocp4.example.com \
  --cert=certificate.crt \
  --key=private.key \
  -n area51

# Add --ca-cert=ca.crt only when the supplied certificate chain requires it.
```

### Verification

```bash
oc get route oxcart -n area51 \
  -o jsonpath='{.spec.host}{" termination="}{.spec.tls.termination}{"\n"}'
oc get endpoints oxcart -n area51
curl -vk https://classified.apps.ocp4.example.com
```

### Traps

- edge, passthrough and reencrypt are not interchangeable.
- The certificate CN/SAN must match the requested hostname.
- A valid route with no service endpoints still fails.

---

## 9. ConfigMap-backed application data

**Memory hook:** Create ConfigMap -> inject into Deployment -> wait for new pod -> curl output.

### Inspect

```bash
oc get deployment,service,route -n acid
oc get configmap -n acid
oc get endpoints -n acid
```

### Resolution

```bash
oc create configmap sedicen \
  --from-literal="RESPONSE=Soda pop won't stop can't stop" \
  -n acid

oc set env deployment/phosphoric --from=configmap/sedicen -n acid

# Create the route only when it does not already exist.
oc expose service phosphoric \
  --hostname=phosphoric.apps.ocp4.example.com \
  -n acid
```

### Verification

```bash
oc get configmap sedicen -n acid -o yaml
oc rollout status deployment/phosphoric -n acid
oc get deployment phosphoric -n acid \
  -o jsonpath='{.spec.template.spec.containers[0].env}{"\n"}'

curl -s http://phosphoric.apps.ocp4.example.com
```

### Traps

- Use shell quotes around values containing spaces or apostrophes.
- Changing a ConfigMap does not always restart existing pods; oc set env changes the pod template and triggers rollout.

---

## 10. Helm chart deployment

**Memory hook:** Repository -> search -> inspect values -> install -> verify Helm and OpenShift objects.

### Inspect

```bash
helm repo list
helm list -A
oc get all,route -n ascii-movie
```

### Resolution

```bash
oc get project ascii-movie >/dev/null 2>&1 || oc new-project ascii-movie

helm repo add do280-repo http://helm.ocp4.example.com/charts
helm repo update
helm search repo do280-repo
helm show values do280-repo/redhat-movie > values-reference.yaml

helm install redhat-movie do280-repo/redhat-movie \
  -n ascii-movie

# When values/version are specified:
# helm install redhat-movie do280-repo/redhat-movie \
#   -n ascii-movie -f values.yaml --version "<version>"
```

### Verification

```bash
helm list -n ascii-movie
helm status redhat-movie -n ascii-movie
oc get all,route -n ascii-movie

HOST=$(oc get route -n ascii-movie -o jsonpath='{.items[0].spec.host}')
curl -I "http://$HOST"
```

### Traps

- Chart name, release name and repository alias are different things.
- Use helm show values before inventing value keys.
- A deployed Helm release is not proven until pods and route work.

---

## 11. Create a Secret

**Memory hook:** Match the secret name, key spelling and value exactly.

### Inspect

```bash
oc get secret -n math
oc get secret magic -n math -o yaml
```

### Resolution

```bash
oc create secret generic magic \
  --from-literal=decoder_ring=XpWy9KdcP3T \
  -n math
```

### Verification

```bash
oc get secret magic -n math \
  -o jsonpath='{.data.decoder_ring}' | base64 -d; echo
oc describe secret magic -n math
```

### Traps

- decoder-ring and decoder_ring are different keys.
- Do not use oc get secret -o yaml as proof of the clear-text value without decoding.

---

## 12. Inject a Secret into an application

**Memory hook:** Secret key -> environment variable reference -> rollout -> application output.

### Inspect

```bash
oc get secret magic -n math -o yaml
oc get deployment qed -n math -o yaml
oc get route,endpoints -n math
```

### Resolution

```bash
# When the required environment variable name equals the secret key:
oc set env deployment/qed --from=secret/magic -n math

# When the required environment variable must be DECODER_RING
# but the Secret key is decoder_ring:
oc edit deployment/qed -n math

# Add under spec.template.spec.containers[0].env:
- name: DECODER_RING
  valueFrom:
    secretKeyRef:
      name: magic
      key: decoder_ring
```

### Verification

```bash
oc rollout status deployment/qed -n math
oc get deployment qed -n math -o jsonpath='{range .spec.template.spec.containers[0].env[*]}{.name}{" <- "}{.valueFrom.secretKeyRef.name}{"/"}{.valueFrom.secretKeyRef.key}{"\n"}{end}'

HOST=$(oc get route -n math -o jsonpath='{.items[0].spec.host}')
curl -s "http://$HOST"
```

### Traps

- Do not copy the secret value into the Deployment as plain text when the task says the application must use the Secret.
- A secretKeyRef proves the workload is referencing the Secret.

---

## 13. ServiceAccount and SCC

**Memory hook:** Create SA -> grant SCC to SA -> assign SA to workload -> prove pod SCC.

### Inspect

```bash
oc get sa -n apples
oc get deployment,deploymentconfig -n apples
oc get pods -n apples --show-labels
oc get scc anyuid
```

### Resolution

```bash
oc create serviceaccount ex280sa -n apples
oc adm policy add-scc-to-user anyuid -z ex280sa -n apples

# Use the kind that actually exists.
oc set serviceaccount deployment/oranges ex280sa -n apples
# For DeploymentConfig:
# oc set serviceaccount dc/oranges ex280sa -n apples
```

### Verification

```bash
oc get deployment oranges -n apples \
  -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'
oc rollout status deployment/oranges -n apples

oc get pods -n apples --show-labels
POD="<target-pod-name>"
oc get pod "$POD" -n apples \
  -o jsonpath='{.spec.serviceAccountName}{" scc="}{.metadata.annotations.openshift\.io/scc}{"\n"}'

oc get pod "$POD" -n apples -o yaml | \
  oc adm policy scc-subject-review -f -
```

### Traps

- Grant the SCC to -z serviceaccount, not to a similarly named user.
- Assigning an SA changes the pod template; wait for the replacement pod.
- Verify the actual pod annotation openshift.io/scc.

---

## 14. Troubleshoot: Service has no endpoints

**Memory hook:** Pod labels must match Service selectors. Endpoints are the proof.

### Inspect

```bash
oc get all -n apples --show-labels
oc get service oranges -n apples -o yaml
oc get endpoints oranges -n apples
oc get pods -n apples --show-labels
oc logs -n apples "<pod-name>"
oc get events -n apples --sort-by=.lastTimestamp | tail -30
```

### Resolution

```bash
# Copy an actual key/value label from the target pod.
oc patch service oranges -n apples --type=merge -p \
  '{"spec":{"selector":{"<pod-label-key>":"<pod-label-value>"}}}'

# Example only, when the pod really has deployment=oranges:
# oc patch service oranges -n apples --type=merge -p \
#   '{"spec":{"selector":{"deployment":"oranges"}}}'
```

### Verification

```bash
oc get endpoints oranges -n apples -o wide
oc describe service oranges -n apples
oc get route -n apples

HOST=$(oc get route -n apples -o jsonpath='{.items[0].spec.host}')
curl -s "http://$HOST"
```

### Traps

- Do not create a new Service or Deployment when the task says no components may be added or removed.
- Never guess selector labels; copy them from the pod.
- Endpoint <none> means the Service currently reaches no ready pod.

---

## 15. Troubleshoot: Pod Pending from a bad resource value

**Memory hook:** Pending -> describe pod -> events -> inspect resources -> correct only the wrong value.

### Inspect

```bash
oc get pods -n mercury
oc describe pod "<pending-pod>" -n mercury
oc get events -n mercury --sort-by=.lastTimestamp | tail -30
oc get deployment atlas -n mercury -o yaml
```

### Resolution

```bash
oc edit deployment atlas -n mercury

# Correct the typo exactly, for example:
# 80Gi -> 80Mi
# Do not remove the entire resources section unless the task explicitly says so.
```

### Verification

```bash
oc rollout status deployment/atlas -n mercury
oc get pods -n mercury
oc describe pod "<new-pod>" -n mercury
oc get route,endpoints -n mercury

HOST=$(oc get route -n mercury -o jsonpath='{.items[0].spec.host}')
curl -s "http://$HOST"
```

### Traps

- The event usually tells you why scheduling failed.
- Fix the unit, not the whole configuration.
- No route test until the pod is Running/Ready and the Service has endpoints.

---

## 16. NetworkPolicy

**Memory hook:** Select destination pods -> allow exact source namespace AND source pods -> allow exact port.

### Inspect

```bash
oc get pods -n database --show-labels
oc get namespace checker --show-labels
oc get pods -n checker --show-labels
oc get networkpolicy -n database
```

### Resolution

```bash
cat > db-allow-sysql-conn.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-sysql-conn
  namespace: database
spec:
  podSelector:
    matchLabels:
      network.openshift.io/policy-group: mysql
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          team: devsecops
      podSelector:
        matchLabels:
          deployment: db-mysql
    ports:
    - protocol: TCP
      port: 3306
EOF

oc apply -f db-allow-sysql-conn.yaml
```

### Verification

```bash
oc get networkpolicy db-allow-sysql-conn -n database -o yaml
oc describe networkpolicy db-allow-sysql-conn -n database

oc get pods -n checker
oc logs -n checker "<checker-pod-or-deployment>"
```

### Traps

- Copy labels exactly from the cluster; the PDFs contain inconsistent example spelling.
- namespaceSelector and podSelector in the same from item are ANDed.
- Two separate from items would allow either source and make the policy too broad.

---

## 17. Default project template with LimitRange

**Memory hook:** Generate bootstrap template -> add object -> store in openshift-config -> point cluster config -> create test project.

### Inspect

```bash
oc get project.config.openshift.io cluster -o yaml
oc get template -n openshift-config
oc get pods -n openshift-apiserver
```

### Resolution

```bash
oc adm create-bootstrap-project-template -o yaml > project-request.yaml
vi project-request.yaml

# Add this item under the existing objects: list.
- apiVersion: v1
  kind: LimitRange
  metadata:
    name: ${PROJECT_NAME}-limits
    namespace: ${PROJECT_NAME}
  spec:
    limits:
    - type: Container
      min:
        memory: 128Mi
      max:
        memory: 1Gi
      default:
        memory: 512Mi
      defaultRequest:
        memory: 256Mi

oc apply -f project-request.yaml -n openshift-config

oc patch project.config.openshift.io/cluster --type=merge -p \
  '{"spec":{"projectRequestTemplate":{"name":"project-request"}}}'
```

### Verification

```bash
oc get project.config.openshift.io cluster \
  -o jsonpath='{.spec.projectRequestTemplate.name}{"\n"}'
oc get template project-request -n openshift-config
watch oc get pods -n openshift-apiserver

oc new-project template-test
oc get limitrange template-test-limits -n template-test -o yaml
```

### Traps

- The LimitRange must be inside the Template objects list.
- Use project.config.openshift.io/cluster, not projects.config.
- Always prove the template by creating a fresh project.

---

## 18. Install an Operator and enable namespace monitoring

**Memory hook:** Namespace + monitoring label -> OperatorGroup -> Subscription -> CSV Succeeded.

### Inspect

```bash
oc get packagemanifest -n openshift-marketplace | grep -i file-integrity
oc get subscription,csv -A | grep -i file-integrity
oc get ns openshift-file-integrity --show-labels
```

### Resolution

```bash
PACKAGE=file-integrity-operator
NS=openshift-file-integrity

oc get project "$NS" >/dev/null 2>&1 || oc new-project "$NS"
oc label namespace "$NS" openshift.io/cluster-monitoring=true --overwrite

CHANNEL=$(oc get packagemanifest "$PACKAGE" -n openshift-marketplace \
  -o jsonpath='{.status.defaultChannel}')
SOURCE=$(oc get packagemanifest "$PACKAGE" -n openshift-marketplace \
  -o jsonpath='{.status.catalogSource}')
SOURCE_NS=$(oc get packagemanifest "$PACKAGE" -n openshift-marketplace \
  -o jsonpath='{.status.catalogSourceNamespace}')

cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: file-integrity-operator-group
  namespace: ${NS}
spec:
  targetNamespaces:
  - ${NS}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ${PACKAGE}
  namespace: ${NS}
spec:
  channel: ${CHANNEL}
  installPlanApproval: Automatic
  name: ${PACKAGE}
  source: ${SOURCE}
  sourceNamespace: ${SOURCE_NS}
EOF
```

### Verification

```bash
oc get namespace openshift-file-integrity --show-labels
oc get operatorgroup,subscription,installplan,csv -n openshift-file-integrity
oc get csv -n openshift-file-integrity \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase
```

### Traps

- Do not hard-code a channel when the question does not provide one; discover the default channel.
- Automatic approval belongs in Subscription spec.installPlanApproval.
- Installed does not mean ready; CSV phase must be Succeeded.

---

## 19. CronJob

**Memory hook:** Minute hour day-of-month month day-of-week -> SA -> history limit -> manual test.

### Inspect

```bash
oc get cronjob -n elementum
oc get sa -n elementum
```

### Resolution

```bash
oc create serviceaccount magna -n elementum

oc create cronjob job-runner \
  --image=registry.domain20.example.com/library/job-runner:latest \
  --schedule='5 4 2 * *' \
  -n elementum

oc patch cronjob job-runner -n elementum --type=merge -p \
'{"spec":{"successfulJobsHistoryLimit":14,"jobTemplate":{"spec":{"template":{"spec":{"serviceAccountName":"magna"}}}}}}'
```

### Verification

```bash
oc get cronjob job-runner -n elementum \
  -o custom-columns=NAME:.metadata.name,SCHEDULE:.spec.schedule,SA:.spec.jobTemplate.spec.template.spec.serviceAccountName,HISTORY:.spec.successfulJobsHistoryLimit

# Optional functional test when safe:
TEST_JOB=job-runner-test-$(date +%s)
oc create job "$TEST_JOB" --from=cronjob/job-runner -n elementum
oc wait --for=condition=complete "job/$TEST_JOB" -n elementum --timeout=5m
oc logs "job/$TEST_JOB" -n elementum
```

### Traps

- 5 4 2 * * means 04:05 on day 2 of every month.
- The serviceAccountName is inside jobTemplate.spec.template.spec.
- Use the exact image tag and registry from the question.

---

## 20. Liveness probe

**Memory hook:** Probe type + port + delay + timeout -> rollout -> inspect pod template.

### Inspect

```bash
oc get deployment atlas -n mercury -o yaml
oc get pods -n mercury --show-labels
POD="<target-pod-name>"
oc describe pod "$POD" -n mercury
```

### Resolution

```bash
oc set probe deployment/atlas \
  --liveness \
  --open-tcp=8080 \
  --initial-delay-seconds=10 \
  --timeout-seconds=30 \
  -n mercury
```

### Verification

```bash
oc rollout status deployment/atlas -n mercury
oc get deployment atlas -n mercury -o jsonpath='{.spec.template.spec.containers[0].livenessProbe.tcpSocket.port}{" delay="}{.spec.template.spec.containers[0].livenessProbe.initialDelaySeconds}{" timeout="}{.spec.template.spec.containers[0].livenessProbe.timeoutSeconds}{"\n"}'

POD="<new-atlas-pod-name>"
oc describe pod "$POD" -n mercury | grep -A6 -i Liveness
```

### Traps

- Liveness, readiness and startup probes solve different problems.
- Check the actual container if the pod has more than one container.
- A wrong port causes repeated container restarts.

---

## 21. LimitRange

**Memory hook:** Pod limits and Container limits are separate entries; defaultRequest is not default.

### Inspect

```bash
oc get limitrange -n sydney
oc describe limitrange -n sydney
```

### Resolution

```bash
cat > ex280-limits.yaml <<'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: ex280-limits
  namespace: sydney
spec:
  limits:
  - type: Pod
    min:
      cpu: 5m
      memory: 300Mi
    max:
      cpu: 500m
      memory: 500Mi
  - type: Container
    min:
      cpu: 100m
      memory: 200Mi
    max:
      cpu: 500m
      memory: 600Mi
    defaultRequest:
      cpu: 300m
      memory: 400Mi
EOF

oc apply -f ex280-limits.yaml
```

### Verification

```bash
oc get limitrange ex280-limits -n sydney -o yaml
oc describe limitrange ex280-limits -n sydney
```

### Traps

- Do not add default limits when only default requests are requested.
- Check whether a value belongs to Pod or Container.
- Use Mi/Gi and m exactly.

---

## 22. Collect support data with must-gather

**Memory hook:** Cluster ID -> must-gather -> archive -> size check -> upload.

### Inspect

```bash
oc whoami
oc get clusterversion version
pwd
df -h .
```

### Resolution

```bash
CLUSTER_ID=$(oc get clusterversion version \
  -o jsonpath='{.spec.clusterID}')

rm -rf must-gather
mkdir must-gather
oc adm must-gather --dest-dir=must-gather

tar -czf "ex280-ocp-${CLUSTER_ID}.tar.gz" must-gather
```

### Verification

```bash
ls -lh "ex280-ocp-${CLUSTER_ID}.tar.gz"
tar -tzf "ex280-ocp-${CLUSTER_ID}.tar.gz" | head

/usr/local/bin/upload-cluster-data \
  "ex280-ocp-${CLUSTER_ID}.tar.gz"
```

### Traps

- Archive the generated must-gather directory, not an empty parent directory.
- Use the cluster ID from clusterversion; do not type it from memory.
- Verify the archive is non-empty before upload.

---

## 23. Static NFS PV, PVC and application mount

**Memory hook:** Inspect storage details -> PV -> PVC bound -> mount -> replicas -> route -> curl.

### Inspect

```bash
oc get storageclass -o yaml
oc get pv
oc get pvc -A
# Locate the NFS server, export path and reclaim policy supplied by the lab.
```

### Resolution

```bash
cat > landing-storage.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: landing-pv
spec:
  capacity:
    storage: 16Mi
  accessModes:
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Delete
  storageClassName: ""
  volumeMode: Filesystem
  nfs:
    server: 192.168.50.254
    path: /exports-ocp4
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: landing-pvc
  namespace: page
spec:
  accessModes:
  - ReadOnlyMany
  resources:
    requests:
      storage: 16Mi
  volumeName: landing-pv
  storageClassName: ""
EOF

oc get project page >/dev/null 2>&1 || oc new-project page
oc apply -f landing-storage.yaml

# Use the image given in the question.
oc new-app --name=landing --image="<image-from-question>" -n page
oc scale deployment/landing --replicas=3 -n page

oc set volume deployment/landing --add \
  --name=landing-content \
  --type=pvc --claim-name=landing-pvc \
  --mount-path=/var/www/html \
  -n page

oc expose service landing -n page
```

### Verification

```bash
oc get pv landing-pv
oc get pvc landing-pvc -n page
oc describe pv landing-pv
oc describe pvc landing-pvc -n page
oc rollout status deployment/landing -n page
oc get pods -n page

oc get deployment landing -n page -o yaml | grep -A12 -E 'volumes:|volumeMounts:'
HOST=$(oc get route landing -n page -o jsonpath='{.spec.host}')
curl -s "http://$HOST"
```

### Traps

- Set the reclaim policy to the exact value requested or discovered.
- For static binding, PV and PVC storageClassName must match; empty string on both avoids dynamic provisioning.
- PVC status must be Bound before debugging the application.
- Use the exact NFS server, export path, mount path, image and replica count from the task.

---

# Daily 10-minute command drill

Say the purpose before typing each command.

```bash
# Context
oc whoami
oc project

# Find truth
oc get all -n <project>
oc get pods -n <project> --show-labels
oc get events -n <project> --sort-by=.lastTimestamp | tail -30

# RBAC proof
oc auth can-i <verb> <resource> --as=<user> -n <project>

# Rollout proof
oc rollout status deployment/<name> -n <project>

# Service proof
oc get service <name> -n <project> -o yaml
oc get endpoints <name> -n <project>

# Workload proof
oc logs -n <project> <pod>
oc get deployment <name> -n <project> -o yaml

# Functional proof
HOST=$(oc get route <name> -n <project> -o jsonpath='{.spec.host}')
curl -s "http://$HOST"
```

# Final exam review checklist

- [ ] Correct admin identity and correct project
- [ ] Required object exists with exact name
- [ ] Correct namespace
- [ ] Correct labels/selectors
- [ ] Correct role and scope
- [ ] Correct CPU/memory units
- [ ] Correct ServiceAccount and SCC
- [ ] Deployment rollout completed
- [ ] Pods Running and Ready
- [ ] Service has endpoints
- [ ] Route hostname and TLS termination are correct
- [ ] Application output verified with curl/logs
- [ ] PV and PVC Bound where applicable
- [ ] Operator CSV Succeeded where applicable
- [ ] kubeadmin deletion postponed until the end

# Sources reworked

- `DO280 4.18V.pdf`
- `DO280 Sample Questions and Answers.pdf`
