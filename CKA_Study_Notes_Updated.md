# CKA Study Notes

## 1. Upgrading kubeadm version

Must change the repository first:

```bash
nano /etc/apt/sources.list.d/kubernetes.list
```

---

## 2. Backup and restore etcd

### Take a snapshot

```bash
kubectl exec -it etcd-control-node -n kube-system -- etcdctl \
  --endpoints 10.69.0.4:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  snapshot save snapshot.db
```

To see the etcd container config:

```bash
sudo cat /etc/kubernetes/manifests/etcd.yaml
```

Make sure the target directory exists first:

```bash
sudo mkdir -p /opt/backups
```

### Step 3: Verify the snapshot's integrity

```bash
sudo ETCDCTL_API=3 etcdctl snapshot status /opt/backups/etcd-snapshot.db --write-out=table
```

### Restore

```bash
kubectl exec -it etcd-control-node -n kube-system -- etcdutl --data-dir=/var/lib/etcd-restore snapshot restore snapshot.db
```

Then edit the manifest:

```bash
sudo vim /etc/kubernetes/manifests/etcd.yaml
```

Edit **3 locations**: `etcd` → `etcd-restore`.

Wait 40–50s for the static pod to restart.

---

## 3. Static Pod

Default location:

```
/etc/kubernetes/manifests/
```

---

## 4. Scheduling

- **Taint**: prevents a pod from being scheduled on a node. To overcome: add a **toleration**.
- **Node selector**: choose a specific node.
- **Node affinity**: force a pod to run on certain nodes.

### a. Node Selector

Label a node:

```bash
kubectl label nodes worker1 disktype=ssd
```

Choose a node when creating the Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: nginx
    image: nginx
```

### b. Node Affinity

- `requiredDuringSchedulingIgnoredDuringExecution`: **Must** match
- `preferredDuringSchedulingIgnoredDuringExecution`: **Prefer** matching, but not mandatory

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: nginx
    image: nginx
```

Common operators: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`

### c. Label Selector

```bash
kubectl label pod nginx environment=production
```

```yaml
selector:
  matchExpressions:
  - key: environment
    operator: In
    values:
    - production
```

### d. Taint

```yaml
nodeSelector:
  env: prod
tolerations:
- key: "env"
  operator: "Equal"
  value: "prod"
  effect: "NoSchedule"
```

---

## 5. Nginx basic config

```nginx
events {}
http {
  server {
    listen 80;
    server_name localhost;

    location / {
      root /usr/share/nginx/html;
      index index.html;
    }

    location /health {
      return 200 "OK\n";
    }
  }
}
```

Test configuration:

```bash
nginx -t
```

---

## 6. Service Account

### By token

Create token:

```bash
kubectl create token <sa-name> -n <namespace> --duration=24h
```

**How to use**

Use directly:

```bash
kubectl --token=<YOUR_TOKEN> get pods -n default
```

Or create a separate kubeconfig context:

```bash
TOKEN=$(kubectl create token myuser -n default)

# Create a kubeconfig user
kubectl config set-credentials myuser --token="$TOKEN"

# Create a context
kubectl config set-context myuser-context --cluster=<CLUSTER_NAME> --user=myuser --namespace=default

# Switch context
kubectl config use-context myuser-context
```

### By certificate

```bash
kubectl create sa ci-runner -n dev
kubectl create role pod-reader -n dev --verb=get,list,watch --resource=pods
kubectl create rolebinding ci-runner-binding -n dev --role=pod-reader --serviceaccount=dev:ci-runner
```

Verify:

```bash
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:ci-runner
```

### Kube Cert API

```bash
openssl genrsa -out myuser.key 2048

openssl req -new -key myuser.key -out myuser.csr -subj "/CN=myuser/O=developers"

cat myuser.csr | base64 -w 0

CSR=$(cat myuser.csr | base64 -w 0)
echo "$CSR"
```

Create `myuser-csr.yaml`:

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser
spec:
  request: BASE64_ENCODED_CSR_HERE
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
```

```bash
kubectl create -f myuser-csr.yaml
kubectl certificate approve myuser
kubectl get csr
```

### kubectl auth

```bash
# Check to see if I can create pods in any namespace
kubectl auth can-i create pods --all-namespaces

# Check to see if I can list deployments in my current namespace
kubectl auth can-i list deployments.apps

# Check to see if service account "foo" of namespace "dev" can list pods in the namespace "prod"
# You must be allowed to use impersonation for the global option "--as"
kubectl auth can-i list pods --as=system:serviceaccount:dev:foo -n prod
```

### Role / RoleBinding

**Role**

```bash
# Create a role named "foo" with API Group specified
kubectl create role foo --verb=get,list,watch --resource=rs.apps

# Create a role named "foo" with SubResource specified
kubectl create role foo --verb=get,list,watch --resource=pods,pods/status
```

From <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_role/>

**RoleBinding**

```bash
# Create a role binding for user1, user2, and group1 using the admin cluster role
kubectl create rolebinding admin --clusterrole=admin --user=user1 --user=user2 --group=group1

# Create a role binding for service account monitoring:sa-dev using the admin role
kubectl create rolebinding admin-binding --role=admin --serviceaccount=monitoring:sa-dev
```

---

## 7. CoreDNS, Deployments, Kustomize

### CoreDNS (Corefile)

```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

| Directive | Meaning |
|---|---|
| `kubernetes` | Enables Kubernetes DNS resolution |
| `cluster.local` | Default Kubernetes DNS domain |
| `in-addr.arpa` | IPv4 reverse DNS lookup |
| `ip6.arpa` | IPv6 reverse DNS lookup |
| `pods insecure` | Allows Pod IP-based DNS records without verification |
| `fallthrough` | Passes unmatched reverse DNS queries to the next plugin |
| `ttl 30` | Sets DNS record TTL to 30 seconds |

Check the DNS ConfigMap:

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Restart CoreDNS to ensure the configuration is loaded:

```bash
kubectl rollout restart deployment coredns -n kube-system
```

Test DNS resolution:

```bash
kubectl run dns-test --image=busybox:1.36 --restart=Never -it --rm -- sh
nslookup kubernetes.default.svc.cluster.local
nslookup google.com
```

### Deployments

```bash
kubectl create deployment frontend --image=nginx:1.24

kubectl rollout status deployment nginx
kubectl rollout history deployment nginx
kubectl rollout undo deployment nginx
kubectl scale deployment nginx --replicas=3
kubectl set image deployment/nginx nginx=nginx:1.27
kubectl create job test-job --image=busybox -- echo hello
kubectl create cronjob test --image=busybox --schedule="*/5 * * * *" -- echo hello
```

### Kustomize

```
~/cka/kustomize/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    └── production/
        └── kustomization.yaml
```

---

## 8. PersistentVolume

Reference: <https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/>

```yaml
accessModes:
  - ReadWriteOnce
```

---

## 9. PersistentVolumeClaim

**Bound vs Pending** depends on the binding mode:

- `Immediate`: Provision/bind PV immediately after PVC creation
- `WaitForFirstConsumer`: Wait until a Pod consumes the PVC

---

## 10. Helm

| Action | Command |
|---|---|
| Add repo | `helm repo add stable URL` |
| Update repo | `helm repo update` |
| Search chart | `helm search repo nginx` |
| Install chart | `helm install nginx stable/nginx` |
| List releases | `helm list -A` |
| Upgrade release | `helm upgrade nginx stable/nginx` |
| Uninstall release | `helm uninstall nginx` |
| Check status | `helm status nginx` |
| Check history | `helm history nginx` |
| Rollback | `helm rollback nginx 1` |
| Render YAML | `helm template nginx ./chart` |
| Create chart | `helm create nginx` |
| Validate chart | `helm lint ./chart` |

**Important distinction:**

- `--skip-crds`: Skip installing CRDs from a chart's `crds/` directory. → `--crds.install=false`
- `--include-crds`: Include CRDs when rendering templates with `helm template`.
- `--debug`: Show additional debugging information.
- `--dry-run`: Simulate an installation or upgrade.

### Autoscaler (HPA)

```bash
kubectl autoscale deployment DEPLOYMENT_NAME \
  --min=MIN \
  --max=MAX \
  --cpu-percent=TARGET_PERCENTAGE
```

For CPU utilization-based HPA to work correctly, containers need CPU resource requests.

Example:

```yaml
resources:
  requests:
    cpu: 100m
  limits:
    cpu: 500m
```

**HPA scaling formula** (know this concept for troubleshooting):

```
DesiredReplicas = ceil( CurrentReplicas × CurrentMetric / TargetMetric )
```

Example:

- Current replicas = 3
- Current CPU utilization = 90%
- Target CPU utilization = 60%

`DesiredReplicas = 3 × 90 / 60 = 4.5` → rounded up: **5 desired replicas**

Actual scaling can be affected by stabilization windows, scaling policies, and configured min/max replicas.

| Autoscaler | What it scales |
|---|---|
| HPA | Number of Pods |
| VPA | CPU/memory requests of Pods |
| Cluster Autoscaler | Number of worker nodes |

---

## 11. Secrets and ConfigMaps

```yaml
envFrom:
- secretRef:
    name: db-credentials
```

Or, if you need specific env var names mapped from specific keys:

```yaml
env:
- name: DB_USERNAME
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: username
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

---

## 12. Liveness and Readiness

- Nginx listens on port 80 inside the container → port `80`.
- Your application listens on port 8080 → port `8080`.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
```

---

## 12. Service

| Field | Meaning |
|---|---|
| `containerPort` | Application listening port |
| Probe `httpGet.port` | Application listening port |
| Service `port` | Port exposed by the Service |
| Service `targetPort` | Destination application port |
| `nodePort` | External node access port |

Remember:

- Probe → container application port.
- Service → service `port` maps to `targetPort`.
- NodePort → external access.

NodePort example:

```yaml
ports:
- nodePort: 30080
  port: 80
  protocol: TCP
  targetPort: 80
selector:
  app: web
sessionAffinity: None
type: NodePort
```

```bash
curl http://<node-ip>:30080
```

---

## 13. Sidecar and initContainers

### a. initContainers

CKA exam scenarios:

- **Wait for a Service**: ensure the application starts only after another service becomes available.
- To verify: `kubectl describe pod …` and look for "pulling image … x … pulling image … y".

```yaml
initContainers:
- name: wait-for-service
  image: busybox:1.36
  command:
  - sh
  - -c
  - |
    until nslookup mysql-service; do
      echo "Waiting for MySQL..."
      sleep 2
    done
```

**Create a file before the application starts:**

```yaml
spec:
  initContainers:
  - name: init
    image: busybox
    command: ["sh", "-c", "echo ready > /work/status"]
    volumeMounts:
    - name: workdir
      mountPath: /work
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /work/status && sleep 3600"]
    volumeMounts:
    - name: workdir
      mountPath: /work
  volumes:
  - name: workdir
    emptyDir: {}
```

### b. Sidecar

Kubernetes supports native sidecars using `initContainers` with:

```yaml
restartPolicy: Always
```

Example (shared log volume):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  containers:
  - name: main-app
    image: busybox
    command: ["sh", "-c", "while true; do echo 'Application log' >> /var/log/app.log; sleep 5; done"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "tail -f /var/log/app.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  volumes:
  - name: shared-logs
    emptyDir: {}
```

---

## 14. Linux commands

### Keep containers running / loops

| Command | Purpose |
|---|---|
| `sleep infinity` | Keep container running indefinitely |
| `sleep 3600` | Sleep for 1 hour |
| `tail -f /dev/null` | Keep container running indefinitely |
| `while true; do sleep 3600; done` | Infinite loop |
| `sh -c 'while true; do echo hello; sleep 5; done'` | Repeatedly print output |

### BusyBox

```bash
nslookup kubernetes.default
nslookup my-service
wget -O- http://my-service
wget -qO- http://my-service:8080
ping 10.0.0.10
```

### Curl image

```bash
curl http://my-service
curl -v http://my-service:8080
curl -I http://my-service
curl -k https://my-service
```

### Ubuntu/Alpine (if utilities are installed)

```bash
ip addr
ip route
cat /etc/resolv.conf
cat /etc/hosts
printenv
```

### Debug pod, CSR, services, logs

```bash
kubectl run debug --image=busybox -it --rm -- sh

kubectl get csr
kubectl certificate approve my-csr

systemctl status kubelet
systemctl status containerd

journalctl -u kubelet -xe
journalctl -u containerd -xe

crictl ps -a
crictl pods
crictl logs CONTAINER_ID

df -h
free -h
top
ss -tulpn
ip route
```

### Useful tools

- `jq`
- `dnsutils`
- `net-tools`
- `tmux`

---

## TLS Secret

Reference: <https://kubernetes.io/docs/concepts/configuration/secret/>

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

If `tls.crt` and `tls.key` already exist:

```bash
kubectl create secret tls app-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

Create manually:

```bash
openssl genrsa -out tls.key 2048

openssl req -x509 -new -nodes \
  -key tls.key \
  -sha256 \
  -days 365 \
  -out tls.crt \
  -subj "/CN=app.example.com/O=app.example.com" \
  -addext "subjectAltName=DNS:app.example.com"

base64 -w 0 tls.crt
base64 -w 0 tls.key
```

PowerShell:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("tls.crt"))
[Convert]::ToBase64String([IO.File]::ReadAllBytes("tls.key"))
```

---

## Ingress

Structure:

```
rules
└── host
    └── http
        └── paths
            └── path
                └── backend
                    └── service
                        ├── name
                        └── port
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## Network Policy

Template:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: <policy-name>
  namespace: <namespace-name>
spec:
  podSelector:
    matchLabels:
      <key>: <value>
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from: [...]
    ports: [...]
  egress:
  - to: [...]
    ports: [...]
```

### Critical parameters of a NetworkPolicy

**`podSelector`**

- `{}` (empty): selects all pods in the namespace.
- `matchLabels`: selects specific pods matching the given labels.
- Omitted: selects no pods.

**`policyTypes`**

- Specifies whether the policy applies to Ingress, Egress, or both.
- **Critical rule:** must be explicitly declared alongside empty rules (`ingress: []` or `egress: []`) to create default-deny behavior.

**`ingress` (incoming traffic)**

- `from`: defines allowed sources using:
  - `podSelector`: pods within the same namespace.
  - `namespaceSelector`: all pods in namespaces matching specific labels.
  - `ipBlock`: specific CIDR IP ranges (with optional `except`).
- `ports`: restricts traffic by protocol (TCP, UDP, SCTP) and port number or name.
- Behavior: empty `ingress: []` blocks all incoming traffic.

**`egress` (outgoing traffic)**

- `to`: defines allowed destinations using the same selectors as ingress (`podSelector`, `namespaceSelector`, `ipBlock`).
- `ports`: restricts outgoing traffic to specific protocols and ports.
- Behavior: empty `egress: []` blocks all outgoing traffic.

---

# CRD, CNI, CSI, CRI — CKA Notes

## CRD — CustomResourceDefinition
- Extends Kubernetes API with custom resource types.
- CRD = Defines resource type; CR = Instance.
- Usually paired with a controller/operator.

```bash
kubectl apply -f database-crd.yaml
kubectl get crd
kubectl api-resources
kubectl get databases
```

**CRD example:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              engine:
                type: string
```

## CNI — Container Network Interface
- Provides Pod networking and IP addresses.
- Examples: Calico, Flannel, Cilium.

```bash
ls /etc/cni/net.d/     # Configuration
ls /opt/cni/bin/      # Plugin binaries
```

**Troubleshooting:** Pod network failure, `Failed to create pod sandbox`, `CNI plugin not initialized`.

## CSI — Container Storage Interface
- Integrates Kubernetes with storage providers.
- Examples: Azure Disk, Azure Files, AWS EBS.
- Controller: Provision/attach/detach.
- Node: Mount/unmount.

```bash
kubectl get pv,pvc
kubectl get sc
kubectl get csidrivers
kubectl describe pvc <pvc-name>
```

**Troubleshooting:** PVC Pending, volume provisioning or mounting failure.

## CRI — Container Runtime Interface
- Allows kubelet to communicate with container runtime.
- Examples: containerd, CRI-O.

```bash
crictl info
crictl ps -a
crictl pods
crictl images
crictl logs <container-id>
systemctl status containerd
```

Basic commands
# Explain a resource
kubectl explain pod

# Explain specific fields
kubectl explain pod.spec
kubectl explain pod.spec.containers

# Go deeper into nested fields
kubectl explain pod.spec.containers.resources

# Show available fields recursively
kubectl explain pod --recursive

# Explain Deployment
kubectl explain deployment.spec

# Explain Service
kubectl explain service.spec


Practical example: Pod securityContext
kubectl explain pod.spec.securityContext
kubectl explain pod.spec.containers.securityContext


Find fields such as runAsUser, runAsGroup, privileged, and readOnlyRootFilesystem.
Practical example: Probes
kubectl explain pod.spec.containers.livenessProbe
kubectl explain pod.spec.containers.readinessProbe
kubectl explain pod.spec.containers.startupProbe


Useful options
# Show all nested fields
kubectl explain pod --recursive

# Specify API version
kubectl explain deployment --api-version=apps/v1


CKA exam tip: When unsure about YAML syntax, use:
kubectl explain <resource>.<field> --recursive

## 15. Miscellaneous

**Style 1 – Indented list (recommended for readability)**

```yaml
volumes:
  - name: logs
    emptyDir: {}
```

**Style 2 – List aligned with parent key**

```yaml
volumes:
- name: logs
  emptyDir: {}
```

All of the volume entries need the `-` dash.


---
## METRICS-SERVER
Installation
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

Disable TLS verifications
```bash
kubectl edit deployment metrics-server -n kube-system
```
add to args:
```bash
  - --kubelet-insecure-tls
``` 
kubectl rollout restart deployment/metrics-server -n kube-system

Get top pods by CPU, by Memory

```bash
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
```

# 16. Missing CKA Topics — High-Priority Additions

## Cluster installation and maintenance

### kubeadm essentials

```bash
# Initialize control plane
sudo kubeadm init --pod-network-cidr=<CIDR>

# Generate worker join command
kubeadm token create --print-join-command

# Join a worker (run the printed command on the worker)
kubeadm join <control-plane-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# Reset a node (destructive)
sudo kubeadm reset
```

- Install a CNI plugin after `kubeadm init` before expecting normal Pod networking.
- `kubeadm join` adds a node to the cluster.
- Keep Kubernetes package versions aligned with the supported upgrade path.

### Node maintenance

```bash
kubectl cordon <node>                 # Stop scheduling new Pods
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>               # Allow scheduling again
```

- `cordon`: marks node unschedulable; existing Pods remain.
- `drain`: evicts eligible Pods for maintenance.
- `uncordon`: makes node schedulable again.
- Drain can be blocked by PodDisruptionBudgets (PDBs).

## Workloads and scheduling

### Workload types

| Kind | Purpose |
|---|---|
| Deployment | Stateless applications; manages ReplicaSets |
| StatefulSet | Stateful Pods with stable identities and storage |
| DaemonSet | One Pod per eligible node |
| Job | Runs a task to completion |
| CronJob | Runs Jobs on a schedule |
| ReplicaSet | Maintains the desired number of Pod replicas |

```bash
kubectl create deployment web --image=nginx
kubectl scale deployment web --replicas=3
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web

kubectl create job test --image=busybox -- echo hello
kubectl create cronjob backup --image=busybox \
  --schedule="*/5 * * * *" -- echo hello
```

### Resource requests and limits

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

- Requests: resources used by the scheduler when placing a Pod.
- Limits: maximum resource usage enforced at runtime.
- CPU over limit: throttling.
- Memory over limit: container may be OOMKilled.

### Pod QoS

- `Guaranteed`: every container has CPU and memory requests equal to limits.
- `Burstable`: at least one request/limit is set, but not all meet Guaranteed criteria.
- `BestEffort`: no CPU or memory requests/limits are set.

### PodDisruptionBudget (PDB)

Protects an application during voluntary disruptions such as node drain.

```bash
kubectl create pdb web-pdb --selector=app=web --min-available=1
kubectl get pdb
```

## Services and networking

### Service types

| Type | Purpose |
|---|---|
| ClusterIP | Internal cluster access; default |
| NodePort | Exposes a port on each node |
| LoadBalancer | Requests an external load balancer |
| ExternalName | Maps a Service name to an external DNS name |

```bash
kubectl expose deployment web --port=80 --target-port=8080
kubectl expose deployment web --type=NodePort --port=80
kubectl get svc
kubectl get endpoints
kubectl get endpointslices
kubectl describe svc web
```

- Service `port`: port exposed by the Service.
- `targetPort`: port on the selected Pod.
- `nodePort`: port exposed on each node.
- Service selector must match Pod labels.
- No matching Pods usually means no ready endpoints.

### Service DNS

```text
<service>.<namespace>.svc.cluster.local
```

- Same namespace: `web`
- Different namespace: `web.production`
- Full name: `web.production.svc.cluster.local`

```bash
kubectl run dns-test --image=busybox:1.36 -it --rm --restart=Never -- sh
nslookup web
nslookup web.production.svc.cluster.local
```

### Gateway API

Know the relationship:

- `GatewayClass`: defines the gateway implementation.
- `Gateway`: defines the traffic entry point/listeners.
- `HTTPRoute`: defines host/path routing to Services.

Minimal HTTPRoute example (assumes `web-gateway` already exists):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - app.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service
      port: 80
```

## Storage

### PV / PVC essentials

| Setting | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Read-write mount by one node |
| `ReadOnlyMany` (ROX) | Read-only mount by many nodes |
| `ReadWriteMany` (RWX) | Read-write mount by many nodes |
| `ReadWriteOncePod` (RWOP) | Read-write mount by one Pod |
| `Filesystem` | Mount volume as a filesystem |
| `Block` | Present volume as a raw block device |

### Reclaim policy

- `Retain`: keep the PV and underlying data after PVC deletion; manual cleanup/reclaim is needed.
- `Delete`: delete the PV and, when supported by the provisioner, the underlying storage.

### StorageClass

Important fields:

```yaml
provisioner: <csi-driver>
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

- `Immediate`: provision/bind as soon as the PVC is created.
- `WaitForFirstConsumer`: delay provisioning/binding until a Pod uses the PVC; helps with topology-aware scheduling.

```bash
kubectl get pv,pvc,sc
kubectl describe pvc <pvc>
kubectl describe pv <pv>
```

## Troubleshooting — highest-weight exam domain

### Fast diagnostic sequence

```bash
kubectl get nodes -o wide
kubectl describe node <node>
kubectl get pods -A -o wide
kubectl describe pod <pod> -n <namespace>
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
kubectl top nodes
kubectl top pods -A
```

### Node / control-plane checks

```bash
sudo systemctl status kubelet
sudo systemctl status containerd
sudo journalctl -u kubelet -xe
sudo journalctl -u containerd -xe
sudo crictl ps -a
sudo crictl pods
```

- Static Pod manifests: `/etc/kubernetes/manifests/`
- Control-plane components commonly run as static Pods.
- If a control-plane component is unhealthy, inspect its Pod, logs, manifest, and kubelet.
- Check node conditions with `kubectl describe node`.

### Common Pod states

| Symptom | First checks |
|---|---|
| `Pending` | Scheduling events, requests, taints, PVC |
| `ImagePullBackOff` | Image name/tag, registry access, imagePullSecrets |
| `CrashLoopBackOff` | `kubectl logs --previous`, command, probes, configuration |
| `ContainerCreating` | Events, CNI, CSI/volume mounts, runtime |
| `OOMKilled` | Memory limits and application usage |
| `CreateContainerConfigError` | Missing Secret, ConfigMap, or referenced key |

### API server health

```bash
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/livez?verbose'
```

- `readyz`: whether the API server is ready to serve requests.
- `livez`: whether the API server is alive.

### Network troubleshooting

```bash
kubectl get svc,endpoints
kubectl get endpointslices
kubectl get networkpolicy -A
kubectl get pods -n kube-system -o wide
```

Test from a temporary Pod:

```bash
kubectl run net-test --image=busybox:1.36 -it --rm \
  --restart=Never -- sh
```

Inside the Pod:

```bash
nslookup <service>
wget -qO- http://<service>:<port>
```

## ConfigMaps and Secrets — creation commands

```bash
kubectl create configmap app-config --from-literal=MODE=prod
kubectl create configmap app-config --from-file=config.txt

kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=change-me
```

- ConfigMap: non-sensitive configuration.
- Secret: sensitive configuration (base64 encoding is not encryption).
- `envFrom`: imports all keys as environment variables.
- `valueFrom`: imports a specific key.

## CKA domain coverage checklist

| Official domain | Weight | Main topics to practice |
|---|---:|---|
| Cluster Architecture, Installation & Configuration | 25% | kubeadm, RBAC, upgrades, maintenance, HA, Helm, Kustomize, CRDs/operators |
| Workloads & Scheduling | 15% | Deployments, rollouts, Jobs, CronJobs, ConfigMaps, Secrets, requests/limits, HPA, scheduling |
| Services & Networking | 20% | Pod networking, Services, endpoints, NetworkPolicy, Ingress, Gateway API, CoreDNS |
| Storage | 10% | PV, PVC, StorageClass, dynamic provisioning, access modes, reclaim policies |
| Troubleshooting | 30% | Nodes, control plane, applications, logs, resource usage, Services and networking |

**Priority:** Spend the most troubleshooting practice time here: it is 30% of the published CKA exam weighting.
