# CKA Notes

## 1. Upgrading kubeadm version

Must change the repository first:

```bash
nano /etc/apt/sources.list.d/kubernetes.list
```

---

## 2. Backup and restore etcd

### Step 1: Take a snapshot

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

### Step 2: Make sure the target directory exists

```bash
sudo mkdir -p /opt/backups
```

### Step 3: Verify the snapshot's integrity

```bash
sudo ETCDCTL_API=3 etcdctl snapshot status /opt/backups/etcd-snapshot.db --write-out=table
```

Or use:

```bash
kubectl exec -it etcd-control-node -n kube-system -- etcdutl --data-dir=/var/lib/etcd-restore snapshot restore snapshot.db
```

### Step 4: Point etcd at the restored data

```bash
sudo vim /etc/kubernetes/manifests/etcd.yaml
```

Edit **3 locations**: `etcd` → `etcd-restore`

Wait 40–50s.

---

## 3. Static Pod

Default location:

```
/etc/kubernetes/manifests/
```

---

## 4. Scheduling

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

- **requiredDuringSchedulingIgnoredDuringExecution**: Must match
- **preferredDuringSchedulingIgnoredDuringExecution**: Prefer matching, but not mandatory

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
kubectl create token <SA_NAME> -n <NAMESPACE> --duration=24h
```

**Use directly:**

```bash
kubectl --token=<YOUR_TOKEN> get pods -n default
```

**Create a separate kubeconfig context:**

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

## 7. DNS / CoreDNS

Corefile:

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

Check DNS ConfigMap:

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

---

## Deployments

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

---

## Kustomize

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

Docs: <https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/>

```yaml
accessModes:
  - ReadWriteOnce
```

---

## 9. PersistentVolumeClaim

**Bound vs Pending**

Binding mode:

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

**HPA Scaling Formula**

Know this concept for troubleshooting:

```
DesiredReplicas = ceil[CurrentReplicas x (CurrentMetric / TargetMetric)]
```

Example:

- Current replicas = 3
- Current CPU utilization = 90%
- Target CPU utilization = 60%

```
DesiredReplicas = 3 x 90/60 = 4.5
```

Rounded up: **5 desired replicas**

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

- Nginx listens on port 80 inside the container → port 80.
- Your application listens on port 8080 → port 8080.

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
| Service `port` | Port exposed by Service |
| Service `targetPort` | Destination application port |
| NodePort | External node access port |

Remember:

- Probe → container application port.
- Service → service `port` maps to `targetPort`.
- NodePort → external access.

![[Pasted image 20261104204105.png]]

---

## 13. Sidecar and initContainers

### a. initContainers

CKA exam scenarios:

**Wait for a Service:** Ensure the application starts only after another service becomes available.

To verify: `kubectl describe pod …`, look for `pulling image … x … pulling image … y`.

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

Kubernetes supports native sidecars using `initContainers` with `restartPolicy: Always`.

Classic two-container example:

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

| Command | Purpose |
|---|---|
| `sleep infinity` | Keep container running indefinitely |
| `sleep 3600` | Sleep for 1 hour |
| `tail -f /dev/null` | Keep container running indefinitely |
| `while true; do sleep 3600; done` | Infinite loop |
| `sh -c 'while true; do echo hello; sleep 5; done'` | Repeatedly print output |

**BusyBox:**

```bash
nslookup kubernetes.default
nslookup my-service
wget -O- http://my-service
wget -qO- http://my-service:8080
ping 10.0.0.10
```

**Curl image:**

```bash
curl http://my-service
curl -v http://my-service:8080
curl -I http://my-service
curl -k https://my-service
```

**Ubuntu/Alpine (if utilities are installed):**

```bash
ip addr
ip route
cat /etc/resolv.conf
cat /etc/hosts
printenv
```

**Debugging and cluster ops:**

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

**Useful tools:** `jq`, `dnsutils`, `net-tools`, `tmux`

---

## 15. Miscellaneous

**YAML list style 1 – Indented list (recommended for readability)**

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
