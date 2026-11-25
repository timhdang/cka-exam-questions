# CKA Study Notes

> **Legend:** 💡 = new tip added in this edition · ⚠️ = common trap or correction. All original content is kept; anything that was wrong, inconsistent or duplicated was fixed or merged.

## Contents

| Part | Topic | Exam weight |
|---|---|---:|
| [Exam overview](#exam-overview) | Domain checklist, exam facts, setup and strategy | |
| [Part 1](#part-1--cluster-architecture-installation--configuration-25) | Cluster Architecture, Installation & Configuration | 25% |
| [Part 2](#part-2--workloads--scheduling-15) | Workloads & Scheduling | 15% |
| [Part 3](#part-3--services--networking-20) | Services & Networking | 20% |
| [Part 4](#part-4--storage-10) | Storage | 10% |
| [Part 5](#part-5--troubleshooting-30) | Troubleshooting | 30% |
| [Part 6](#part-6--tools-tricks--reference) | Tools, tricks & reference | |

---

## Exam overview

| Official domain | Weight | Main topics to practice |
|---|---:|---|
| Cluster Architecture, Installation & Configuration | 25% | kubeadm, RBAC, upgrades, maintenance, HA, Helm, Kustomize, CRDs/operators |
| Workloads & Scheduling | 15% | Deployments, rollouts, Jobs, CronJobs, ConfigMaps, Secrets, requests/limits, HPA, scheduling |
| Services & Networking | 20% | Pod networking, Services, endpoints, NetworkPolicy, Ingress, Gateway API, CoreDNS |
| Storage | 10% | PV, PVC, StorageClass, dynamic provisioning, access modes, reclaim policies |
| Troubleshooting | 30% | Nodes, control plane, applications, logs, resource usage, Services and networking |

**Priority:** Spend the most troubleshooting practice time here: it is 30% of the published CKA exam weighting.

### Exam facts

| Item | Value |
|---|---|
| Format | Performance-based, remotely proctored, live terminal on real clusters (no multiple choice) |
| Duration | 2 hours |
| Tasks | Roughly 15–20 (sources report up to ~25) |
| Passing score | 66% |
| Kubernetes version | Recent sources say v1.35 (late 2026). The exam version can lag the newest Kubernetes release, so check the CNCF/Linux Foundation CKA page near your exam date |
| Allowed docs | `kubernetes.io/docs`, `kubernetes.io/blog`, `github.com/kubernetes`; Helm docs are normally allowed for the CKA too. Confirm on the LF "Resources allowed" page |
| Exam terminal | `kubectl` (alias `k` + bash completion usually preconfigured), `helm`, `vim` (no config), `jq`, `yq`, `curl`, `wget`, `man` |

> 💡 **Tip:** Because the exam may run an older version than the newest release, do not assume brand-new features exist. Check with `kubectl version` and `kubectl api-resources` on the exam cluster.

### Exam-day setup (do this in the first 2 minutes)

```bash
alias k=kubectl                       # usually already set; harmless to repeat
export do="--dry-run=client -o yaml"  # k run web --image=nginx $do > pod.yaml
export now="--force --grace-period=0" # k delete pod web $now
source <(kubectl completion bash)
complete -o default -F __start_kubectl k   # completion for the alias (only if missing)
```

```bash
# ~/.vimrc: YAML must use spaces, never tabs
cat >> ~/.vimrc <<'EOF'
set number
set expandtab
set tabstop=2
set shiftwidth=2
set softtabstop=2
EOF
```

### Exam strategy

- 💡 **Switch context first.** Every task lists a `kubectl config use-context <name>` command. Run it before anything else and confirm with `kubectl config current-context`. Working in the wrong cluster scores zero.
- 💡 **Read the namespace.** Use `-n <ns>` explicitly instead of changing the default namespace and forgetting about it.
- 💡 **Node tasks need SSH.** `ssh <node>`, then `sudo -i`. `kubectl` is normally only configured on the base host, so `exit` back before running `kubectl` again.
- 💡 **Budget time.** 120 min ÷ ~17 tasks ≈ 7 min each. Tasks show their weight: do heavy and easy ones first, flag hard ones and return.
- 💡 **Scoring is per requirement.** Partial work earns partial credit, so never leave a task blank.
- 💡 **Prefer imperative + `$do`** to generate YAML, then edit. Copy examples from the docs only for objects with no imperative command (PV, PVC, StorageClass, NetworkPolicy, Gateway, HTTPRoute...).
- 💡 **Verify every task** with `get`, `describe`, or `curl`. Many failures are typos (wrong namespace, label, port).
- 💡 **Back up before editing** system files: `sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/`. Never leave a backup copy *inside* `/etc/kubernetes/manifests/` (the kubelet will run it as another static Pod).
- 💡 **Terminal copy/paste** in the exam browser is `Ctrl+Shift+C` / `Ctrl+Shift+V` (Linux/Windows) or `Cmd+C` / `Cmd+V` (Mac).

---

# Part 1 — Cluster Architecture, Installation & Configuration (25%)

## 1.1 kubeadm: install, join, reset

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

> 💡 **Tip: node prerequisites** (before `kubeadm init`/`join`): swap off (`sudo swapoff -a` and remove it from `/etc/fstab`; the kubelet refuses to start with swap by default), kernel modules `overlay` and `br_netfilter`, sysctls `net.ipv4.ip_forward=1` and `net.bridge.bridge-nf-call-iptables=1`, a container runtime (containerd with `SystemdCgroup = true` in `/etc/containerd/config.toml`), then install `kubeadm kubelet kubectl` and `apt-mark hold` them.

> 💡 **Tip: `--pod-network-cidr` must match your CNI.** Flannel's default is `10.244.0.0/16`, Calico's manifest default is `192.168.0.0/16`. A mismatch gives Pods with broken networking.

> 💡 **Tip: join token problems.** Tokens expire after 24h (`kubeadm token list`); just run `kubeadm token create --print-join-command` again. To recompute the CA hash by hand:
>
> ```bash
> openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null \
>   | openssl dgst -sha256 -hex | sed 's/^.* //'
> ```

> ⚠️ **Trap:** `kubeadm reset` does not remove CNI config (`/etc/cni/net.d`), iptables/IPVS rules, or `$HOME/.kube/config`. Clean them manually before re-initialising.

### Highly available control plane (HA)

```bash
# First control-plane node: point at a load balancer in front of all API servers
sudo kubeadm init --control-plane-endpoint "<LB_DNS>:6443" --upload-certs --pod-network-cidr=<CIDR>

# Additional control-plane nodes
sudo kubeadm join <LB_DNS>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>
```

- **Stacked etcd:** etcd runs on the control-plane nodes (kubeadm default). **External etcd:** separate etcd cluster; fewer failure couplings, more machines.
- 💡 etcd needs a quorum of `⌊n/2⌋ + 1`: 3 members tolerate 1 failure, 5 tolerate 2. Use odd numbers (4 members tolerate no more failures than 3).
- The `--certificate-key` from `--upload-certs` expires after 2 hours; regenerate with `sudo kubeadm init phase upload-certs --upload-certs`.

## 1.2 Upgrading kubeadm version

Must change the repository first:

```bash
nano /etc/apt/sources.list.d/kubernetes.list
```

> 💡 **Tip:** The `pkgs.k8s.io` repository is **per minor version** (`.../core:/stable:/v1.32/deb/`). If you skip the repo edit, `apt` only offers patch releases of the old minor.

**Rules**

- Upgrade **one minor version at a time** (1.31 → 1.32 → 1.33); minors cannot be skipped.
- `kubeadm upgrade apply` runs on the **first** control-plane node only; other control-plane and worker nodes use `kubeadm upgrade node`.
- `kubeadm` does not upgrade the kubelet or kubectl; do those with the package manager.
- Version skew: kubelet may be up to 3 minor versions *older* than the API server (never newer); `kubectl` ±1 minor.
- Back up etcd first ([1.4](#14-backup-and-restore-etcd)).

**Control-plane node**

```bash
# 1. Point the repo at the new minor version (e.g. v1.31 -> v1.32)
sudo nano /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
apt-cache madison kubeadm | head          # find the exact patch version

# 2. Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm='1.32.x-*'
sudo apt-mark hold kubeadm
kubeadm version

# 3. Plan and apply
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.32.x        # add --etcd-upgrade=false if the task says not to touch etcd

# 4. Drain, upgrade kubelet + kubectl, restart, uncordon
kubectl drain <cp-node> --ignore-daemonsets
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet='1.32.x-*' kubectl='1.32.x-*'
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon <cp-node>
```

**Worker node**

```bash
# From the control plane
kubectl drain <worker> --ignore-daemonsets --delete-emptydir-data

# On the worker (ssh): repo edit + kubeadm install as above, then:
sudo kubeadm upgrade node
sudo apt-get install -y kubelet='1.32.x-*' kubectl='1.32.x-*'
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# From the control plane
kubectl uncordon <worker>
kubectl get nodes                          # VERSION column shows the new version
```

> 💡 **Tip:** `kubectl get nodes` shows the *kubelet* version. After `kubeadm upgrade apply` the node still shows the old version until the kubelet package is upgraded and restarted.

Correct commands
On the control-plane node:
```sudo kubeadm upgrade apply v1.37.x```


On the worker node:
```sudo kubeadm upgrade node``` 

WRONG
```sudo kubeadm upgrade apply v1.37.x```


## 1.3 kubeconfig, contexts and admin.conf

### Context

```bash
kubectl config set-context CONTEXT_NAME \
  --cluster=CLUSTER_NAME \
  --user=USER_NAME \
  --namespace=NAMESPACE
```

(See also: [ServiceAccount token context](#serviceaccount-access-by-token) for creating a user + context from a token.)

> 💡 **Tip: everyday kubeconfig commands**
>
> ```bash
> kubectl config get-contexts
> kubectl config current-context
> kubectl config use-context <name>
> kubectl config set-context --current --namespace=<ns>   # change default namespace of current context
> kubectl config view --minify                             # only the active context
> kubectl --kubeconfig=/path/to/file get nodes             # or: export KUBECONFIG=/path/to/file
> ```

### admin.conf

```bash
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

## 1.4 Backup and restore etcd

### Inspect the etcd config

To see the etcd container config (endpoints, cert paths, data dir):

```bash
sudo cat /etc/kubernetes/manifests/etcd.yaml
```

> 💡 **Tip:** grab only the values you need for `etcdctl`:
>
> ```bash
> sudo grep -E 'listen-client-urls|cert-file|key-file|trusted-ca-file|data-dir' /etc/kubernetes/manifests/etcd.yaml
> ```
>
> Use `--cert-file`, `--key-file`, `--trusted-ca-file` (not the `peer-` ones) for `--cert`, `--key`, `--cacert`. Default endpoint: `https://127.0.0.1:2379`.

### Prepare the target directory

Make sure the target directory exists first:

```bash
sudo mkdir -p /opt/backups
```

### Take a snapshot

Run `etcdctl` **on the control-plane node (host)** so the file lands on the host:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot save /opt/backups/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

(Lab example from the original notes: `--endpoints 10.69.0.4:2379`; use whatever `listen-client-urls` shows.)

> ⚠️ **Correction:** the original notes ran `kubectl exec ... etcdctl snapshot save snapshot.db`. That writes the file **inside the etcd container**, not to `/opt/backups`, and it is lost when the Pod is recreated. If you must exec, save under `/var/lib/etcd/` (a hostPath mount) so the file reaches the host. If `etcdctl` is missing on the host: `sudo apt install etcd-client`.

### Verify the snapshot's integrity

```bash
sudo etcdutl snapshot status /opt/backups/etcd-snapshot.db --write-out=table
# older equivalent (deprecated): sudo ETCDCTL_API=3 etcdctl snapshot status /opt/backups/etcd-snapshot.db --write-out=table
```

### Restore

Restore into a **new** data directory, on the host:

```bash
sudo etcdutl --data-dir=/var/lib/etcd-restore snapshot restore /opt/backups/etcd-snapshot.db
# older equivalent: sudo ETCDCTL_API=3 etcdctl snapshot restore /opt/backups/etcd-snapshot.db --data-dir=/var/lib/etcd-restore
```

Then edit the manifest:

```bash
sudo vim /etc/kubernetes/manifests/etcd.yaml
```

Edit **3 locations**: `etcd` → `etcd-restore` (`--data-dir`, the volumeMount `mountPath`, and the hostPath `path`).

> 💡 **Shortcut:** changing **only** the `hostPath.path` (volume `etcd-data`) to `/var/lib/etcd-restore` is enough: the container still sees it as `/var/lib/etcd`. Fewer edits, fewer typos.

Wait 40–50s for the static pod to restart. Check with `sudo crictl ps | grep etcd` (`kubectl` may fail for a minute).

> 💡 **Tip:** `etcdutl snapshot restore` works on the file only, so it needs no endpoints or certs. `etcdctl snapshot restore` is deprecated in favour of `etcdutl`.

> 💡 **Tip: external etcd?** Check `--etcd-servers` in `/etc/kubernetes/manifests/kube-apiserver.yaml`. If it points to another host, `ssh` there: the data dir is in the systemd unit (`systemctl cat etcd`); restore, update `--data-dir`, then `sudo systemctl daemon-reload && sudo systemctl restart etcd`.

> 💡 **Tip: health checks:** `etcdctl endpoint health --write-out=table` and `etcdctl member list --write-out=table` (same cert flags).

## 1.5 Static Pods

Default location:

```
/etc/kubernetes/manifests/
```

- 💡 The real path is `staticPodPath` in `/var/lib/kubelet/config.yaml` (`grep staticPodPath /var/lib/kubelet/config.yaml`). The task may use a custom path.
- 💡 Static Pods are run by the **kubelet**, not the scheduler. They appear in the API as *mirror Pods* named `<pod>-<nodename>` (e.g. `etcd-control-node`), with an `ownerReference` of kind `Node`.
- ⚠️ `kubectl delete pod` on a static Pod only recreates it. To remove it, **delete or move the manifest file**. Editing the file restarts the Pod automatically.
- 💡 Quick generator: `kubectl run web --image=nginx $do | sudo tee /etc/kubernetes/manifests/web.yaml`
- Control-plane components (`etcd`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`) are static Pods on kubeadm clusters.

## 1.6 Node maintenance

```bash
kubectl cordon <node>                 # Stop scheduling new Pods
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>               # Allow scheduling again
```

- `cordon`: marks node unschedulable; existing Pods remain.
- `drain`: evicts eligible Pods for maintenance.
- `uncordon`: makes node schedulable again.
- Drain can be blocked by PodDisruptionBudgets (PDBs). See [PDB](#26-poddisruptionbudget-pdb).

> 💡 **Tip:** `drain` cordons the node automatically. Pods not managed by a controller (bare Pods) block the drain unless you add `--force`, and they are **deleted permanently**. Useful flags: `--timeout=60s`, `--grace-period=<s>`, `--disable-eviction` (bypasses PDBs; last resort).
>
> `uncordon` does **not** move Pods back; the scheduler only places *new* Pods there.
>
> A cordoned node shows `Ready,SchedulingDisabled` in `kubectl get nodes`.

## 1.7 Authentication & RBAC

### ServiceAccount access by token

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

> 💡 **Tip:** `kubectl create token` issues a short-lived token (default 1h). Since v1.24 Kubernetes no longer auto-creates a Secret per ServiceAccount. For a long-lived token create a Secret of `type: kubernetes.io/service-account-token` with annotation `kubernetes.io/service-account.name: <sa>`.
>
> A Pod uses a ServiceAccount via `spec.serviceAccountName`; set `automountServiceAccountToken: false` when the Pod does not need API access.

### ServiceAccount with Role and RoleBinding

```bash
kubectl create sa ci-runner -n dev
kubectl create role pod-reader -n dev --verb=get,list,watch --resource=pods
kubectl create rolebinding ci-runner-binding -n dev --role=pod-reader --serviceaccount=dev:ci-runner
```

Verify:

```bash
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:ci-runner
```

### User certificate via the Kube CSR API

```bash
openssl genrsa -out myuser.key 2048

openssl req -new -key myuser.key -out myuser.csr -subj "/CN=myuser/O=developers"

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

> 💡 **Finish the job** (the step that is easy to forget): extract the signed certificate and build a kubeconfig user.
>
> ```bash
> kubectl get csr myuser -o jsonpath='{.status.certificate}' | base64 -d > myuser.crt
> kubectl config set-credentials myuser --client-certificate=myuser.crt --client-key=myuser.key --embed-certs=true
> kubectl config set-context myuser-ctx --cluster=<CLUSTER_NAME> --user=myuser
> kubectl --context=myuser-ctx get pods     # Forbidden until you bind a Role
> kubectl create rolebinding myuser-view --clusterrole=view --user=myuser -n default
> ```
>
> `CN` becomes the **username** and `O` the **group** in RBAC. `expirationSeconds` has a 10-minute minimum. `kubectl certificate deny <csr>` rejects a request.

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

### ClusterRole / ClusterRoleBinding

```bash
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes
kubectl create clusterrolebinding node-reader-binding --clusterrole=node-reader --user=jane
kubectl create role cm-reader --verb=get --resource=configmaps --resource-name=app-config   # limit to one object
```

> 💡 **RBAC cheat notes**
>
> - **Role / RoleBinding** are namespaced; **ClusterRole / ClusterRoleBinding** are cluster-wide (needed for nodes, PVs, namespaces and other cluster-scoped resources).
> - A **RoleBinding that references a ClusterRole** grants those permissions **only in the RoleBinding's namespace**. This is how the built-in `view`, `edit` and `admin` roles are reused per namespace.
> - RBAC is **additive**: there are no deny rules. `roleRef` cannot be changed after creation (delete and recreate the binding).
> - Subject kinds: `User`, `Group`, `ServiceAccount` (ServiceAccount flag syntax is `--serviceaccount=<namespace>:<name>`).
> - Inspect: `kubectl get role,rolebinding -n <ns>`, `kubectl describe rolebinding <name> -n <ns>`, `kubectl get clusterrolebinding | grep <name>`.
> - Fast YAML: add `$do` to any `kubectl create role|rolebinding|clusterrole...` command.

### kubectl auth can-i

```bash
# Check to see if I can create pods in any namespace
kubectl auth can-i create pods --all-namespaces

# Check to see if I can list deployments in my current namespace
kubectl auth can-i list deployments.apps

# Check to see if service account "foo" of namespace "dev" can list pods in the namespace "prod"
# You must be allowed to use impersonation for the global option "--as"
kubectl auth can-i list pods --as=system:serviceaccount:dev:foo -n prod
```

> 💡 **Tip:** `kubectl auth can-i --list -n dev --as=jane` shows everything a subject can do; `kubectl auth whoami` shows who you are authenticated as.

## 1.8 Pod Security Admission (PSA)

PSA is enforced per **namespace** through labels. It replaced PodSecurityPolicy (removed in v1.25).

```bash
kubectl label ns backend pod-security.kubernetes.io/enforce=restricted
kubectl label ns backend pod-security.kubernetes.io/warn=restricted pod-security.kubernetes.io/audit=restricted
kubectl label ns backend pod-security.kubernetes.io/enforce=baseline --overwrite   # change level
kubectl get ns backend --show-labels
```

| Level | Meaning |
|---|---|
| `privileged` | Unrestricted |
| `baseline` | Blocks known privilege escalations (privileged containers, hostNetwork/hostPID/hostIPC, hostPath volumes, extra capabilities) |
| `restricted` | Hardened best practice (non-root, drop ALL capabilities, seccomp, no privilege escalation) |

| Mode | Effect |
|---|---|
| `enforce` | Rejects violating Pods |
| `warn` | Allows, but shows a warning to the user |
| `audit` | Allows, records an audit annotation |

| Security requirement | Example |
|---|---|
| No privileged containers | `privileged: false` |
| No hostNetwork | `hostNetwork: false` |
| No hostPID | `hostPID: false` |
| No hostPath volumes | Avoid host filesystem access |
| Run as non-root | `runAsNonRoot: true` |
| Drop Linux capabilities | `capabilities.drop: ["ALL"]` |
| No privilege escalation | `allowPrivilegeEscalation: false` |
| Use seccomp | `seccompProfile.type: RuntimeDefault` |

**Example: Pod that violates `restricted`**

This Pod will be rejected:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
  namespace: backend
spec:
  containers:
    - name: nginx
      image: nginx
      securityContext:
        privileged: true
```

Expected error:

```
Error from server (Forbidden):
violates PodSecurity "restricted"
```

**Example: Pod that satisfies `restricted`**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: backend
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

> 💡 **Tips**
>
> - Preview the impact on existing Pods before enforcing: `kubectl label --dry-run=server --overwrite ns backend pod-security.kubernetes.io/enforce=restricted` prints a warning per violating Pod.
> - ⚠️ With a **Deployment**, the Deployment itself is accepted but its Pods are rejected: you see *no Pods*. Look at `kubectl describe rs <rs>` / `kubectl get events` for the `violates PodSecurity` message.
> - PSA only checks Pods at creation; existing Pods keep running.
> - Stock `nginx` runs as root and listens on 80, so it fails `restricted`. Use `nginxinc/nginx-unprivileged` (port 8080) or a non-root busybox/app image.

## 1.9 Helm

| Action | Command |
|---|---|
| Add repo | `helm repo add stable URL` |
| Update repo | `helm repo update` |
| List repos | `helm repo list` |
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
| Show default values | `helm show values <repo>/<chart>` |
| Show values of a release | `helm get values <release>` (add `--all` for computed values) |
| Install or upgrade | `helm upgrade --install <rel> <chart>` |
| Install into namespace | `helm install <rel> <chart> -n <ns> --create-namespace` |
| Override values | `--set key=value` or `-f values.yaml` |

**Important distinction:**

- `--skip-crds`: Skip installing CRDs from a chart's `crds/` directory.
- `--include-crds`: Include CRDs when rendering templates with `helm template`.
- `--debug`: Show additional debugging information.
- `--dry-run`: Simulate an installation or upgrade.

> ⚠️ **Correction:** the original notes equated `--skip-crds` with `--crds.install=false`. Those are not the same. `--skip-crds` is a Helm flag; `crds.install=false` is a **chart-specific value** (used as `--set crds.install=false`) that only some charts define.

> 💡 **Tips**
>
> - `helm rollback <release> <revision>`: the number comes from `helm history`; a rollback creates a *new* revision.
> - `--set` overrides `-f values.yaml`; later `-f` files override earlier ones.
> - Debug a failing install: `helm install ... --dry-run --debug`, or `helm template ... | kubectl apply --dry-run=server -f -`.
> - Stuck releases: `helm list -A --pending` / `--failed`; a release stuck in `pending-install` must be uninstalled first.
> - A chart's `stable/...` name is only a placeholder; the old `stable` repo is archived. Use the repo the task gives you (e.g. `bitnami/nginx`).
> - `helm uninstall` removes the release's resources but not CRDs installed from `crds/`.

## 1.10 Kustomize

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

Base `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

Create the overlay kustomization.yaml (`overlays/production/`):

```bash
cat <<'EOT' > ~/cka/kustomize/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-

replicas:
  - name: app
    count: 3

images:
  - name: nginx
    newTag: "1.25"

labels:
  - pairs:
      env: prod
    includeSelectors: true
EOT
```

Apply the overlay:

```bash
kubectl apply -k ~/cka/kustomize/overlays/production
```

> ⚠️ **Corrections:** the original applied `~/kustomize/overlays/prod`, which does not match the tree (`~/cka/kustomize/overlays/production`). `commonLabels` is deprecated in favour of `labels:` (it still works but prints a warning). `replicas[].name` must match the **Deployment's name in the base** (`app` here), not the prefixed name.

> 💡 **Tips**
>
> - Preview without applying: `kubectl kustomize ~/cka/kustomize/overlays/production` (equivalent to `kustomize build`).
> - Remove: `kubectl delete -k <dir>`; inspect: `kubectl get -k <dir>`.
> - Other common fields: `namespace: <ns>`, `namePrefix`/`nameSuffix`, `commonAnnotations`, `configMapGenerator`/`secretGenerator` (generated names get a hash suffix, which triggers a rollout when content changes).
> - Patches (very common in tasks):
>
> ```yaml
> patches:
>   - target:
>       kind: Deployment
>       name: app
>     patch: |-
>       - op: replace
>         path: /spec/template/spec/containers/0/resources/limits/memory
>         value: 256Mi
> ```
>
> (Inline JSON6902 patch; a strategic-merge patch can be given as a YAML snippet or a `path:` file instead.)

## 1.11 Extension interfaces: CRD, CNI, CSI, CRI

### CRD — CustomResourceDefinition

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

**Matching custom resource (CR):**

```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: orders-db
spec:
  engine: postgres
```

> 💡 **Tips**
>
> - `metadata.name` of a CRD **must be `<plural>.<group>`** (`databases.example.com`).
> - Inspect: `kubectl explain database.spec`, `kubectl api-resources --api-group=example.com`, `kubectl get crd databases.example.com -o yaml`.
> - ⚠️ **Deleting a CRD deletes every custom resource of that type.**
> - An **operator** = CRDs + a controller Deployment (+ RBAC) that reconciles the CRs. Install it with `kubectl apply -f`, Helm, or OLM; afterwards create CRs and check the operator Pod logs.

### CNI — Container Network Interface

- Provides Pod networking and IP addresses.
- Examples: Calico, Flannel, Cilium.

```bash
ls /etc/cni/net.d/     # Configuration
ls /opt/cni/bin/      # Plugin binaries
```

**Troubleshooting:** Pod network failure, `Failed to create pod sandbox`, `CNI plugin not initialized`.

> 💡 **Tips:** without a CNI, nodes stay `NotReady` and CoreDNS stays `Pending`/`ContainerCreating`. If `/etc/cni/net.d/` holds several files, the kubelet uses the first one alphabetically. Not every CNI enforces NetworkPolicy: **Calico and Cilium do, plain Flannel does not** (policies are silently ignored).

### CSI — Container Storage Interface

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

> 💡 `kubectl get volumeattachments` shows which volumes are attached to which nodes; also check the CSI controller/node Pods in `kube-system`.

### CRI — Container Runtime Interface

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

> 💡 **Tips:** `crictl` needs root (`sudo`) and reads its runtime socket from `/etc/crictl.yaml` (`runtime-endpoint: unix:///run/containerd/containerd.sock`); if it complains about endpoints, pass `--runtime-endpoint`. Useful extras: `crictl exec -it <id> sh`, `crictl inspect <id>`, `crictl stats`. Containerd settings live in `/etc/containerd/config.toml`; restart containerd after changing them.

---

## 1.12 - REST

Fetching Kubernetes Secrets via REST API
[Accessing the Kubernetes API from a Pod](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/)

### Option 1: Using kubectl proxy (easiest locally)

`kubectl proxy` runs a local HTTP server that forwards requests to the API server and attaches your kubeconfig credentials and TLS verification automatically:

```bash
kubectl proxy --port=8080 &
curl -s http://localhost:8080/api/v1/namespaces/default/secrets/my-secret
```

### Option 2: Direct HTTPS request with a Bearer token

From inside a Pod, the API server is reachable at `https://kubernetes.default.svc` (or `https://kubernetes.default`), with the ServiceAccount token and CA certificate mounted under `/var/run/secrets/kubernetes.io/serviceaccount/`:

```bash
APISERVER="https://kubernetes.default.svc"
CACERT="/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

curl -s \
  --cacert "$CACERT" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" \
  "$APISERVER/api/v1/namespaces/default/secrets/my-secret"
```

From outside the cluster, replace `$APISERVER` with your API server URL and supply a valid Bearer token or client certificate along with the cluster's CA bundle.

> 💡 **Tips**
>
> - Secret values in the JSON are base64: `... | jq -r '.data.password' | base64 -d`.
> - ⚠️ A `403 Forbidden` from inside a Pod means the Pod's ServiceAccount lacks RBAC permission (the `default` SA can't read Secrets). Test with `kubectl auth can-i get secrets --as=system:serviceaccount:default:default`.
> - Easiest alternative with no proxy: `kubectl get --raw /api/v1/namespaces/default/secrets/my-secret`.
> - **Learn the REST paths from kubectl itself:** `kubectl get pods -v=8` prints every HTTP call it makes.
> - Path pattern: core group `/api/v1/...`; named groups `/apis/<group>/<version>/...` (e.g. `/apis/apps/v1/namespaces/default/deployments`). `kubectl proxy` listens on `127.0.0.1:8001` by default and only on localhost.

## 1.13 Control-plane quick reference

| Component | Port | Notes |
|---|---:|---|
| kube-apiserver | 6443 | Only component that talks to etcd |
| etcd | 2379 (client) / 2380 (peer) | Cluster state |
| kubelet | 10250 | Runs on every node; runs the static Pods |
| kube-scheduler | 10259 | Assigns Pods to nodes |
| kube-controller-manager | 10257 | Runs the controllers (Deployment, Node, ...) |
| NodePort range | 30000–32767 | Service `nodePort` |

| What | Where |
|---|---|
| Static Pod manifests | `/etc/kubernetes/manifests/` |
| PKI / certificates | `/etc/kubernetes/pki/` (etcd certs in `pki/etcd/`) |
| Component kubeconfigs | `/etc/kubernetes/{admin,kubelet,controller-manager,scheduler}.conf` |
| Kubelet config | `/var/lib/kubelet/config.yaml` |
| Kubelet service unit and drop-in | `systemctl cat kubelet` (drop-in: `10-kubeadm.conf`) |

```bash
sudo kubeadm certs check-expiration     # kubeadm certs are valid 1 year by default (CA: 10 years)
sudo kubeadm certs renew all            # then restart the control-plane static Pods
```

> 💡 **Tip:** if the cluster is down and `kubectl` is dead, everything you need is on the control-plane node: `crictl ps -a`, `crictl logs`, `journalctl -u kubelet`, and the manifests above.

---

# Part 2 — Workloads & Scheduling (15%)

## 2.1 Workload types

| Kind | Purpose |
|---|---|
| Deployment | Stateless applications; manages ReplicaSets |
| StatefulSet | Stateful Pods with stable identities and storage |
| DaemonSet | One Pod per eligible node |
| Job | Runs a task to completion |
| CronJob | Runs Jobs on a schedule |
| ReplicaSet | Maintains the desired number of Pod replicas |

> 💡 **Tips**
>
> - **No imperative command for DaemonSet or StatefulSet.** Generate a Deployment, then edit: `kubectl create deployment ds --image=nginx $do > ds.yaml`, change `kind: DaemonSet`, and delete `replicas:` and `strategy:` (DaemonSets have neither).
> - **StatefulSet** Pods are named `web-0`, `web-1`, ... and start/stop in order; it needs `serviceName:` pointing at a **headless Service** (`clusterIP: None`) for stable DNS (`web-0.web.<ns>.svc`); storage comes from `volumeClaimTemplates` (one PVC per Pod, kept when the Pod is deleted).
> - **DaemonSet** Pods ignore the control-plane taint unless you add a toleration for `node-role.kubernetes.io/control-plane`.

## 2.2 Deployments, Jobs and CronJobs: commands

```bash
kubectl create deployment frontend --image=nginx:1.24
kubectl create deployment web --image=nginx --replicas=3     # --replicas works at creation time

kubectl scale deployment web --replicas=3
kubectl set image deployment/nginx nginx=nginx:1.27          # <deployment>/<name> container=image

kubectl rollout status deployment/web                        # "deployment web" also works
kubectl rollout history deployment/web
kubectl rollout undo deployment/web

kubectl create job test-job --image=busybox -- echo hello
kubectl create cronjob backup --image=busybox \
  --schedule="*/5 * * * *" -- echo hello
```

> 💡 **More rollout commands**
>
> ```bash
> kubectl rollout history deployment/web --revision=2         # details of one revision
> kubectl rollout undo deployment/web --to-revision=1         # roll back to a specific revision
> kubectl rollout restart deployment/web                      # rolling restart (same image)
> kubectl rollout pause deployment/web                        # batch several edits...
> kubectl rollout resume deployment/web                       # ...then roll out once
> kubectl annotate deployment/web kubernetes.io/change-cause="upgrade to 1.27"   # fills CHANGE-CAUSE in history (--record is deprecated)
> kubectl create job manual-run --from=cronjob/backup         # trigger a CronJob right now
> ```
>
> - Update strategy: `RollingUpdate` (default, `maxSurge: 25%`, `maxUnavailable: 25%`) or `Recreate` (kill all, then start).
> - Only changes to `spec.template` create a new revision; scaling does not.

**Job and CronJob fields worth knowing**

```yaml
apiVersion: batch/v1
kind: Job
spec:
  completions: 5               # total successful Pods needed
  parallelism: 2               # Pods running at once
  backoffLimit: 4              # retries before the Job is marked Failed
  activeDeadlineSeconds: 120   # hard timeout for the whole Job
  ttlSecondsAfterFinished: 60  # auto-delete finished Job
  template:
    spec:
      restartPolicy: Never     # must be Never or OnFailure (never Always)
      containers:
      - name: task
        image: busybox
        command: ["sh", "-c", "echo hello"]
```

CronJob extras: `schedule` (`min hour day-of-month month day-of-week`), `concurrencyPolicy: Allow|Forbid|Replace`, `successfulJobsHistoryLimit` (default 3), `failedJobsHistoryLimit` (default 1), `startingDeadlineSeconds`, `suspend: true`, `timeZone`.

## 2.3 Scheduling

- **Taint**: prevents a pod from being scheduled on a node. To overcome: add a **toleration**.
- **Node selector**: schedule only onto nodes that carry matching labels (simple equality).
- **Node affinity**: richer label rules. `required...` forces the match; `preferred...` is only a soft preference.

> ⚠️ **A toleration only *permits* scheduling on a tainted node; it does not *attract* the Pod there.** To pin a Pod to a tainted node you need the toleration **and** a nodeSelector/affinity.

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

Soft rule (`preferred` needs a `weight` 1–100 and a `preference`, not `nodeSelectorTerms`):

```yaml
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
```

> 💡 **Tips**
>
> - "IgnoredDuringExecution" means Pods already running are **not** evicted if node labels change later.
> - Within one `nodeSelectorTerms` entry, `matchExpressions` are **ANDed**; multiple `nodeSelectorTerms` entries are **ORed**.
> - Bypass the scheduler entirely with `spec.nodeName: <node>`. This is also the emergency workaround when `kube-scheduler` is down.

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

> 💡 **CLI selectors:** `kubectl get pods -l environment=production`, `-l 'environment in (production,staging)'`, `-l '!canary'`, `--show-labels`. Remove a label: see [6.6.1](#661-remove-a-label).

### d. Taint and toleration

```bash
kubectl taint nodes <node> env=prod:NoSchedule
```

```yaml
nodeSelector:
  env: prod
tolerations:
- key: "env"
  operator: "Equal"
  value: "prod"
  effect: "NoSchedule"
```

> ⚠️ **Fix to the original example:** `nodeSelector: env: prod` matches a **label**, but the command above only adds a **taint**. Also label the node, or the Pod stays `Pending`:
> `kubectl label node <node> env=prod`

> 💡 **Taint reference**
>
> | Effect | Behaviour |
> |---|---|
> | `NoSchedule` | New Pods without a toleration are not scheduled; existing Pods stay |
> | `PreferNoSchedule` | Soft version of NoSchedule |
> | `NoExecute` | Also **evicts** existing Pods without a matching toleration |
>
> - Remove a taint: add a trailing dash: `kubectl taint nodes <node> env=prod:NoSchedule-`
> - Tolerate any value of a key: `operator: Exists` (omit `value`). Tolerate everything: `tolerations: [{operator: Exists}]`.
> - Control-plane nodes carry `node-role.kubernetes.io/control-plane:NoSchedule` (that is why ordinary Pods don't land there). Check taints: `kubectl describe node <node> | grep -i taint`.
> - Scheduler message decoder: `0/3 nodes are available: 1 node(s) had untolerated taint {...}, 2 node(s) didn't match Pod's node affinity/selector` → add a toleration and/or fix the labels.

### e. Pod affinity / anti-affinity and spread (quick reference)

```yaml
affinity:
  podAntiAffinity:                       # keep replicas on different nodes
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: web
      topologyKey: kubernetes.io/hostname
```

`topologyKey` defines the "same place" domain (`kubernetes.io/hostname` = same node, `topology.kubernetes.io/zone` = same zone). `topologySpreadConstraints` (`maxSkew`, `topologyKey`, `whenUnsatisfiable`) spread Pods evenly across domains.

## 2.4 Resource requests and limits

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
- Memory over limit: container is OOMKilled (exit code 137).

> 💡 **Units:** `100m` = 0.1 CPU core (`1` = one core). `Mi`/`Gi` are binary (1024-based), `M`/`G` decimal; `256Mi` ≠ `256M`.
>
> The scheduler places Pods by **requests**, not by live usage. `Insufficient cpu` / `Insufficient memory` in events means requests exceed what is *allocatable* (`kubectl describe node` → *Allocated resources*).
>
> If you set only `limits`, Kubernetes copies them into `requests`.
>
> Change on the fly: `kubectl set resources deployment web --requests=cpu=100m,memory=128Mi --limits=cpu=500m,memory=256Mi`.

### Pod QoS

- `Guaranteed`: every container has CPU and memory requests equal to limits.
- `Burstable`: at least one request/limit is set, but not all meet Guaranteed criteria.
- `BestEffort`: no CPU or memory requests/limits are set.

```bash
kubectl get pods -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,QOS:.status.qosClass"
```

> 💡 Under node memory pressure Pods are evicted in this order: **BestEffort → Burstable → Guaranteed**.

### LimitRange and ResourceQuota

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: dev
spec:
  limits:
  - type: Container
    default:            # default limits
      cpu: 500m
      memory: 256Mi
    defaultRequest:     # default requests
      cpu: 100m
      memory: 128Mi
```

```bash
kubectl create quota dev-quota -n dev --hard=cpu=2,memory=2Gi,pods=10
kubectl describe quota -n dev
```

> ⚠️ **Trap:** if a namespace has a ResourceQuota on CPU/memory, Pods **without** requests/limits are rejected (`must specify limits.cpu...`) unless a LimitRange supplies defaults. A Deployment then shows no Pods; see `kubectl describe rs`.

## 2.5 Autoscaling (HPA)

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

HPA needs the metrics server. See [Metrics server](#55-metrics-server-and-kubectl-top).

**HPA as YAML** (`autoscaling/v2`):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

> 💡 **Tips**
>
> - `kubectl get hpa -w`: TARGETS shows `<unknown>/60%` when the metrics server is missing **or** the Pod has no CPU request.
> - Utilization is measured **relative to the request**, so a missing/too-high request distorts scaling.
> - Defaults: no scaling when the ratio is within ±10% (tolerance); scale-down stabilization window 300s, scale-up reacts immediately.
> - VPA is not built into Kubernetes; it is an add-on with its own CRDs.
> - Newer kubectl versions may expose a `--cpu` flag on `kubectl autoscale` alongside/in place of `--cpu-percent`; check `kubectl autoscale -h` on the exam cluster.

## 2.6 PodDisruptionBudget (PDB)

Protects an application during voluntary disruptions such as node drain.

```bash
kubectl create pdb web-pdb --selector=app=web --min-available=1
kubectl get pdb
```

YAML:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: db-pdb
  namespace: default
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: db
```

> 💡 **Tips**
>
> - Use **either** `minAvailable` **or** `maxUnavailable` (not both). Both accept an integer or a percentage.
> - `kubectl get pdb` → `ALLOWED DISRUPTIONS = 0` means `kubectl drain` will hang on those Pods.
> - ⚠️ A Deployment with `replicas: 1` and `minAvailable: 1` can **never** be drained. Scale up, relax the PDB, or (last resort) `drain --disable-eviction`.
> - PDBs only guard **voluntary** disruptions (drain/eviction), not node crashes.

## 2.7 ConfigMaps and Secrets

### Creation commands

```bash
kubectl create configmap app-config --from-literal=MODE=prod
kubectl create configmap app-config --from-file=config.txt

kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=change-me
```

- ConfigMap: non-sensitive configuration.
- Secret: sensitive configuration (base64 encoding is not encryption).
- `envFrom`: imports all keys as environment variables.
- `valueFrom`: imports a specific key.

> 💡 **More creation forms**
>
> ```bash
> kubectl create configmap app-config --from-env-file=app.env        # KEY=value lines
> kubectl create secret generic app-secret --from-file=ssh-key=./id_rsa
> kubectl create secret docker-registry regcred \
>   --docker-server=<registry> --docker-username=<u> --docker-password=<p>   # used as imagePullSecrets
> kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
> ```
>
> In YAML, use `stringData:` for plain text (the API server encodes it to `data:` for you). Secrets are only *encoded*; protect them with RBAC and enable encryption at rest (`EncryptionConfiguration`) on the API server for real security.

### Consuming a Secret in a Pod

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

**As files (volume mounts)**

```yaml
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: cfg
      mountPath: /etc/app          # each key becomes a file
    - name: creds
      mountPath: /etc/creds
      readOnly: true
  volumes:
  - name: cfg
    configMap:
      name: app-config
  - name: creds
    secret:
      secretName: db-credentials
```

> 💡 **Tips**
>
> - Mounted ConfigMaps/Secrets are **refreshed automatically** (after a short delay); **environment variables are not**: restart the Pods (`kubectl rollout restart`).
> - Mounting over a directory hides its existing files. To drop in a single file, use `subPath` (but `subPath` mounts **don't** update automatically).
> - A missing Secret/ConfigMap/key gives `CreateContainerConfigError` (see [5.2](#52-common-pod-states)).
> - `immutable: true` on a ConfigMap/Secret blocks later edits and reduces API server load.

### TLS Secret

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

The TLS Secret is used by [Ingress](#34-ingress).

> 💡 **Tip:** one-liner for a self-signed cert + key: `openssl req -x509 -nodes -newkey rsa:2048 -days 365 -keyout tls.key -out tls.crt -subj "/CN=app.example.com"`. The TLS Secret must live in the **same namespace** as the Ingress that references it.

## 2.8 Liveness and Readiness probes

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

| Probe | If it fails |
|---|---|
| `livenessProbe` | Container is **restarted** |
| `readinessProbe` | Pod is **removed from Service endpoints** (not restarted) |
| `startupProbe` | Container is restarted; liveness/readiness are **held off** until it succeeds (for slow starters) |

> 💡 **Tips**
>
> - Probe handlers: `httpGet`, `tcpSocket` (`port`), `exec` (`command: ["cat","/tmp/healthy"]`), and `grpc`.
> - Tuning fields: `initialDelaySeconds`, `periodSeconds` (default 10), `timeoutSeconds` (1), `failureThreshold` (3), `successThreshold` (1).
> - A Pod whose readiness probe fails shows `0/1 READY` and an empty Service endpoint list: a classic "Service doesn't work" cause.
> - A liveness probe on the wrong port/path causes endless restarts → `CrashLoopBackOff`.

## 2.9 initContainers and sidecars

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

> 💡 **Tips**
>
> - While init containers run, `kubectl get pod` shows `Init:0/1`. Read their logs with `kubectl logs <pod> -c <init-name>`; if one fails you see `Init:Error` / `Init:CrashLoopBackOff`.
> - Init containers run **one after another, each to completion**, before any app container starts. If one keeps failing the Pod never starts.
> - They share volumes with the app containers (e.g. `emptyDir`) but not the app's running processes.

### b. Sidecar

Kubernetes supports native sidecars using `initContainers` with:

```yaml
restartPolicy: Always
```

**Classic sidecar** (a second entry in `containers`, shared log volume):

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

**Native sidecar** (an init container with `restartPolicy: Always`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: native-sidecar-demo
spec:
  initContainers:
  - name: log-sidecar
    image: busybox
    restartPolicy: Always          # makes it a sidecar: starts first, keeps running
    command: ["sh", "-c", "tail -F /var/log/app.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  containers:
  - name: main-app
    image: busybox
    command: ["sh", "-c", "while true; do echo 'Application log' >> /var/log/app.log; sleep 5; done"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  volumes:
  - name: shared-logs
    emptyDir: {}
```

> ⚠️ **Correction:** the original example was introduced as a native sidecar but used the classic form (both containers under `containers`). Both patterns are shown now. Native sidecars start **before** the main containers, keep running, and are stopped **after** them (so they also work correctly in Jobs). Use `kubectl logs sidecar-demo -c sidecar` to read the sidecar output.

## 2.10 YAML style: lists

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

Both styles are valid YAML. Every list item needs its own `-` dash (all the volume entries), and the items must line up at the same indentation.

> 💡 **YAML survival tips**
>
> - Spaces only, never tabs; keep consistent 2-space indentation.
> - Several objects in one file are separated by a line with `---`.
> - Validate before applying: `kubectl apply -f x.yaml --dry-run=server` (server-side validation) and `kubectl diff -f x.yaml`.
> - Unsure about a field? `kubectl explain pod.spec.containers.<field>` (see [6.2](#62-kubectl-explain)).
> - `kubectl replace --force -f pod.yaml` deletes and recreates, the fix for editing immutable Pod fields.

---

# Part 3 — Services & Networking (20%)

## 3.1 Services

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

> 💡 **Tips**
>
> - `kubectl expose` copies the selector from the Deployment/Pod. `kubectl create service ...` does **not** (see [6.6.2](#662-create-a-service)).
> - `nodePort` range: **30000–32767** (auto-assigned if omitted). Each Service also gets a ClusterIP.
> - `LoadBalancer` stays `<pending>` on clusters with no cloud controller (kubeadm, kind) unless something like MetalLB is installed; use NodePort there.
> - `kubectl get svc -o wide` shows the selector. `kubectl get endpoints web` empty = selector mismatch **or** Pods not Ready.
> - The legacy Endpoints API is deprecated in favour of **EndpointSlices** (v1.33+); both still show data.
> - Quick access test without exposing anything: `kubectl port-forward svc/web 8080:80` then `curl localhost:8080`.

**Headless Service and ExternalName**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  clusterIP: None            # headless: DNS returns the Pod IPs directly (used by StatefulSets)
  selector:
    app: db
  ports:
  - port: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  type: ExternalName
  externalName: api.example.com   # in-cluster name -> CNAME to this DNS name
```

### Service ports

```yaml
spec:
  selector:
    app: web          # must match pod labels
  ports:
  - port: 80          # Service port (what clients call)
    targetPort: 8080  # container port (where the app listens)
```

`wget http://web:80` → pod `:8080`

### Port field cheat sheet

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

> 💡 `containerPort` is **informational**: it does not open or block anything. Traffic reaches the app as long as it listens on that port. `targetPort` may also be a **port name** defined in the Pod (`ports: - name: http, containerPort: 8080`).

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

## 3.2 Service DNS

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

**ERROR:** no Service named `api` in namespace `backend`

```bash
/ # nslookup api.backend.svc.cluster.local
Server:         10.96.0.10
Address:        10.96.0.10:53

** server can't find api.backend.svc.cluster.local: NXDOMAIN
```

**NORMAL**

```bash
/ # nslookup api.backend.svc.cluster.local
Server:         10.96.0.10
Address:        10.96.0.10:53


Name:   api.backend.svc.cluster.local
Address: 10.96.235.141
```

**TEST DNS + HTTP**

```bash
wget -qO- http://api.backend
```

(The flag is `-qO-`: capital letter **O**, not a zero.)

| Result | Meaning |
|---|---|
| HTML or application response | DNS and HTTP connectivity work |
| `bad address` | DNS resolution failed |
| `Connection refused` | Destination reachable, but connection rejected |
| `Connection timed out` | Possible network, routing, or endpoint issue |

> 💡 **Tips**
>
> - The DNS server IP in `nslookup` (`10.96.0.10`) is the **`kube-dns` Service ClusterIP** in `kube-system` (`kubectl get svc kube-dns -n kube-system`). It is also what `cat /etc/resolv.conf` shows in any Pod.
> - A Pod's `/etc/resolv.conf` has `search <ns>.svc.cluster.local svc.cluster.local cluster.local` and `options ndots:5`. That is why short names (`web`, `web.production`) work, and why names with fewer than 5 dots try the search domains first.
> - Pod DNS: `<pod-ip-with-dashes>.<ns>.pod.cluster.local`. StatefulSet Pods behind a headless Service: `web-0.web.<ns>.svc.cluster.local`.
> - Reading the `wget` result: **refused** → Service reached but wrong `targetPort`/app not listening; **timed out** → usually a **NetworkPolicy**, firewall or CNI problem; **bad address** → DNS/CoreDNS problem.
> - `dnsPolicy: ClusterFirst` is the Pod default; custom entries go in `dnsConfig`.

## 3.3 CoreDNS (Corefile)

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
| `errors` | Logs errors to stdout |
| `health` / `ready` | Liveness (`:8080/health`) and readiness (`:8181/ready`) endpoints |
| `prometheus :9153` | Exposes metrics |
| `forward . /etc/resolv.conf` | Sends non-cluster names to the node's upstream resolvers |
| `cache 30` | Caches answers for up to 30 s |
| `loop` | Detects forwarding loops and stops CoreDNS |
| `reload` | Automatically reloads the Corefile when the ConfigMap changes |
| `loadbalance` | Round-robins A/AAAA record order |

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

> 💡 **Tips**
>
> - CoreDNS Pods carry the label `k8s-app=kube-dns` (the label name is kept for historical reasons): `kubectl get pods -n kube-system -l k8s-app=kube-dns`, `kubectl logs -n kube-system -l k8s-app=kube-dns`.
> - Edit with `kubectl edit configmap coredns -n kube-system`; the `reload` plugin picks changes up on its own after a short delay, but `rollout restart` makes it immediate.
> - **Forward one domain to a custom resolver** (common task): add a *separate server block* in the Corefile:
>
> ```
> example.com:53 {
>     errors
>     cache 30
>     forward . 8.8.8.8
> }
> ```
>
> - CoreDNS in `CrashLoopBackOff` with a `loop` message usually means the node's `/etc/resolv.conf` points back at the local stub resolver (e.g. `127.0.0.53`). Point `forward` at real upstream DNS servers instead.
> - Cluster names resolve but `google.com` doesn't → check `forward`/upstream and egress NetworkPolicies; the reverse → check the `kubernetes` block and the Service/endpoints.

## 3.4 Ingress

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

The `secretName` refers to a [TLS Secret](#tls-secret).

> 💡 **Tips**
>
> - Imperative shortcut: `kubectl create ingress web-ingress --class=nginx --rule="app.example.com/*=web-service:80,tls=app-tls-secret"` (add `$do` to preview; a trailing `*` in the path means `Prefix`).
> - `pathType` is **required**: `Prefix`, `Exact`, or `ImplementationSpecific`.
> - An Ingress object does nothing without an **Ingress controller**. Check `kubectl get ingressclass` and the controller Pods; `ingressClassName` must match a real IngressClass.
> - Test without DNS: `curl -H "Host: app.example.com" http://<node-ip>:<ingress-nodePort>/` or `curl --resolve app.example.com:443:<ip> https://app.example.com -k`.
> - `kubectl describe ingress` shows the resolved backends; `<error: endpoints "x" not found>` = wrong Service name/port.
> - Annotations are controller-specific (e.g. `nginx.ingress.kubernetes.io/rewrite-target: /`).
> - ⚠️ **Ingress NGINX was retired in March 2026** (no more releases or security fixes) and the Kubernetes project recommends moving to **Gateway API** or another controller. The Ingress API itself still exists, but Gateway API is the future, so know both ([3.5](#35-gateway-api)).

## 3.5 Gateway API

### 3.5.1 Gateway API basics

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

**GatewayClass and Gateway** (what `web-gateway` could look like):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-gc
spec:
  controllerName: example.com/gateway-controller   # provided by the implementation
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: example-gc
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same            # Same (default) | All | Selector
  - name: https
    protocol: HTTPS
    port: 443
    hostname: app.example.com
    tls:
      mode: Terminate
      certificateRefs:
      - name: app-tls-secret  # TLS Secret in the Gateway's namespace
```

> 💡 **Tips**
>
> - Gateway API objects are **CRDs**; the CRDs and a controller must be installed (`kubectl get crd | grep gateway`, `kubectl get gatewayclass`).
> - Check status: `kubectl get gateway,httproute` and `kubectl describe httproute <name>`; look at conditions `Accepted`, `ResolvedRefs` (backend Service found?) and `Programmed`.
> - A Route attaches to a Gateway via `parentRefs` (optionally `sectionName: <listener>`); the listener's `allowedRoutes` must permit the Route's namespace.
> - A Route referencing a Service or Secret in **another namespace** needs a `ReferenceGrant` in the target namespace.
> - Role split: infrastructure provider → GatewayClass; cluster operator → Gateway; app developer → Routes. That separation is the main design improvement over Ingress.
> - Migration helper: the `ingress2gateway` tool converts existing Ingress objects to Gateway API resources.

### 3.5.2 HTTPRoute: matches and filters

- **`matches`**: determine *which* incoming HTTP requests select a given rule.
- **`filters`**: determine *how* matched requests or responses are modified, redirected, mirrored, or processed.

#### `matches` (request selection)

**Evaluation logic**

- **Within a single match entry (`AND`)**: all specified conditions (`path`, `headers`, `queryParams`, `method`) must match together.
- **Across multiple match entries (`OR`)**: if any entry in `matches[]` evaluates to true, the rule is selected.
- **Default**: omitting `matches` defaults to `PathPrefix: /` (matches all requests).

**Match fields**

- **`path`**
  - `PathPrefix`: segment-aware prefix match (e.g., `/v2` matches `/v2` and `/v2/users`, not `/v20`).
  - `Exact`: exact full path match.
  - `RegularExpression`: regex path match (extended/implementation-dependent).
- **`headers`**: match request header names and values (`Exact` or `RegularExpression`).
- **`queryParams`**: match URL query parameter keys and values (`Exact` or `RegularExpression`).
- **`method`**: match specific HTTP methods (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, etc.).

**Precedence** (when several rules match, the **most specific** wins; only one rule handles a request):

1. `Exact` path match
2. `PathPrefix` with the **longest** prefix
3. Method match
4. The most header matches
5. The most query-parameter matches

Ties between Routes are broken by the **oldest creation time**, then alphabetically by `namespace/name`.

#### Example: AND

All conditions in **one** match entry must be true:

```yaml
matches:
  - path:
      type: PathPrefix
      value: /v2/orders
    method: GET
    headers:
      - type: Exact
        name: X-Channel
        value: mobile
```

#### Example: OR

**Separate** match entries: either one selects the rule:

```yaml
matches:
  - path:
      type: PathPrefix
      value: /v2/orders
  - path:
      type: Exact
      value: /v2/orders/health
```

#### Example: combined AND / OR

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: orders-route
spec:
  parentRefs:
    - name: public-gateway
  rules:
    # Rule 1: Matches if (GET /v2/orders AND X-Channel: mobile) OR (Exact /v2/orders/health)
    - matches:
        - path:
            type: PathPrefix
            value: /v2/orders
          method: GET
          headers:
            - type: Exact
              name: X-Channel
              value: mobile
        - path:
            type: Exact
            value: /v2/orders/health
      backendRefs:
        - name: orders-service
          port: 8080
```

> ⚠️ **Correction:** the original "AND" example actually repeated the two-entry (OR) list. A real AND is a *single* entry with several conditions.

#### `filters` (request / response processing)

| Filter type | Purpose |
|---|---|
| `RequestHeaderModifier` | Add / set / remove request headers |
| `ResponseHeaderModifier` | Add / set / remove response headers |
| `RequestRedirect` | Redirect the client (scheme, host, port, path, status code) |
| `URLRewrite` | Rewrite hostname or path before sending to the backend |
| `RequestMirror` | Copy traffic to another backend (fire and forget) |
| `ExtensionRef` | Implementation-specific custom filter |

```yaml
rules:
- matches:
  - path:
      type: PathPrefix
      value: /api
  filters:
  - type: RequestHeaderModifier
    requestHeaderModifier:
      add:
      - name: X-Env
        value: prod
      remove:
      - X-Debug
  - type: URLRewrite
    urlRewrite:
      path:
        type: ReplacePrefixMatch
        replacePrefixMatch: /
  backendRefs:
  - name: api-service
    port: 8080
# A separate rule: HTTP -> HTTPS redirect (no backendRefs needed)
rules:
- filters:
  - type: RequestRedirect
    requestRedirect:
      scheme: https
      statusCode: 301
```

**Traffic splitting (canary):** give each backend a `weight`:

```yaml
backendRefs:
- name: web-v1
  port: 80
  weight: 90
- name: web-v2
  port: 80
  weight: 10
```

> 💡 `RequestRedirect` and `URLRewrite` can't be combined in the same rule. Weights are relative (90/10 = 9:1); `weight: 0` sends no traffic.

## 3.6 Network Policy

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

### Important: AND vs OR in `from` / `to`

**OR logic** (two separate list items):

```yaml
to:
- namespaceSelector: ...
- podSelector: ...
```

**AND logic** (one list item with both selectors):

```yaml
to:
- namespaceSelector: ...
  podSelector: ...
```

> ⚠️ **The only difference is one `-` dash.** Same item = AND (Pod with these labels *in* namespaces with those labels); separate items = OR (any Pod in those namespaces, **or** matching Pods in the policy's own namespace). This is a favourite exam trick.

DNS and kube-system (egress rule to allow DNS):

```yaml
egress:
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
  ports:
  - protocol: UDP
    port: 53
  - protocol: TCP
    port: 53
```

> ⚠️ **Fixed indentation:** the original snippet's `ports:` was indented differently from `to:`, which is invalid YAML. To restrict this rule to CoreDNS Pods only, add `podSelector: {matchLabels: {k8s-app: kube-dns}}` in the **same** list item as the `namespaceSelector` (AND).

### Critical parameters of a NetworkPolicy

**`podSelector`**

- `{}` (empty): selects all pods in the namespace.
- `matchLabels`: selects specific pods matching the given labels.
- It is a **required** field of `spec`; it cannot simply be left out. (In a `from`/`to` entry, leaving out `podSelector` just means "no Pod filtering".)

**`policyTypes`**

- Specifies whether the policy applies to Ingress, Egress, or both.
- **Critical rule:** a policy only affects the directions listed in `policyTypes`. If omitted, it defaults to `Ingress` (plus `Egress` only when the policy has egress rules). So to deny all egress you must list `Egress` explicitly. Selecting Pods with a policy that lists a direction but allows nothing is what creates default-deny for that direction.

**`ingress` (incoming traffic)**

- `from`: defines allowed sources using:
  - `podSelector`: pods within the same namespace.
  - `namespaceSelector`: all pods in namespaces matching specific labels.
  - `ipBlock`: specific CIDR IP ranges (with optional `except`).
- `ports`: restricts traffic by protocol (TCP, UDP, SCTP) and port number or name.
- Behavior: empty `ingress: []` (or no `ingress:` key with `Ingress` in `policyTypes`) blocks all incoming traffic.

**`egress` (outgoing traffic)**

- `to`: defines allowed destinations using the same selectors as ingress (`podSelector`, `namespaceSelector`, `ipBlock`).
- `ports`: restricts outgoing traffic to specific protocols and ports.
- Behavior: empty `egress: []` blocks all outgoing traffic.

> ⚠️ `ingress: []` **blocks everything**, but `ingress: [{}]` (one empty rule) **allows everything**. One bracket pair is the difference between deny-all and allow-all.

**Default-deny and common patterns**

```yaml
# Deny all ingress to every Pod in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Deny all ingress AND egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow ingress to Pods labelled app=db only from app=web Pods on TCP 5432
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-web
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 5432
```

> 💡 **Tips**
>
> - NetworkPolicies are **additive (OR)**: there are no deny rules; traffic is allowed if **any** policy selecting the Pod allows it.
> - They apply to **Pods**, not Services; Service IPs are translated before policies are evaluated.
> - A connection needs **both ends** to agree: the source's *egress* policy (if any) **and** the destination's *ingress* policy. Replies on an allowed connection are allowed automatically.
> - Restricting egress also blocks **DNS**; always allow UDP/TCP 53 to CoreDNS (see snippet above).
> - `namespaceSelector` matches **namespace labels**. Every namespace automatically has `kubernetes.io/metadata.name=<name>`; for anything else, label it first: `kubectl label ns prod env=prod`.
> - `ipBlock` example: `- ipBlock: {cidr: 10.0.0.0/8, except: [10.0.5.0/24]}`; port ranges via `endPort`.
> - Debug: `kubectl get networkpolicy -A`, `kubectl describe networkpolicy <name>`, then test from a Pod: `wget -qO- --timeout=2 http://svc`. A timeout (not "refused") points to a policy.
> - If policies seem ignored, the CNI may not enforce them (Flannel doesn't).

---

# Part 4 — Storage (10%)

## 4.1 PersistentVolume (PV)

Reference: <https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/>

```yaml
accessModes:
  - ReadWriteOnce
```

A complete static PV (hostPath, lab use only):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

### Access modes and volume modes

| Setting | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Read-write mount by one node |
| `ReadOnlyMany` (ROX) | Read-only mount by many nodes |
| `ReadWriteMany` (RWX) | Read-write mount by many nodes |
| `ReadWriteOncePod` (RWOP) | Read-write mount by one Pod |
| `Filesystem` | Mount volume as a filesystem |
| `Block` | Present volume as a raw block device |

> 💡 RWO is per **node**: several Pods on the *same* node can still mount it. Use RWOP when exactly one Pod may use the volume.

### Reclaim policy

- `Retain`: keep the PV and underlying data after PVC deletion; manual cleanup/reclaim is needed.
- `Delete`: delete the PV and, when supported by the provisioner, the underlying storage.

> 💡 **Tips**
>
> - With `Retain`, after the PVC is deleted the PV goes to **`Released`** and will **not** bind again until you clear its claim: `kubectl patch pv pv-demo -p '{"spec":{"claimRef":null}}'` (it becomes `Available`).
> - Change the policy of an existing PV: `kubectl patch pv pv-demo -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'`.
> - `Recycle` is deprecated; dynamically provisioned PVs inherit the StorageClass `reclaimPolicy` (default `Delete`).
> - PVs are **cluster-scoped** (no namespace); PVCs are namespaced.

## 4.2 PersistentVolumeClaim (PVC)

**Bound vs Pending** depends on the binding mode:

- `Immediate`: Provision/bind PV immediately after PVC creation
- `WaitForFirstConsumer`: Wait until a Pod consumes the PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: manual
```

Use it in a Pod:

```yaml
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-demo
```

> 💡 **How a PVC finds a PV (static binding):** the PV must have `capacity ≥` the request, include the requested `accessModes`, and have the **same `storageClassName`**. Mismatch = PVC stays `Pending`.
>
> - `storageClassName: ""` (empty string) means "no class, bind to a classless PV" and disables dynamic provisioning. **Omitting** the field uses the default StorageClass.
> - With `WaitForFirstConsumer`, a `Pending` PVC with *no Pod yet* is **normal**, not an error.
> - A PVC in use by a Pod is protected (`kubernetes.io/pvc-protection`); deleting it stays `Terminating` until the Pod is gone.
> - **Expand a PVC:** the StorageClass needs `allowVolumeExpansion: true`; then edit the PVC's `spec.resources.requests.storage` to a larger value. PVCs cannot be shrunk.

## 4.3 StorageClass

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

For CSI drivers and PVC troubleshooting, see [CSI](#csi--container-storage-interface).

> 💡 **Tips**
>
> ```yaml
> apiVersion: storage.k8s.io/v1
> kind: StorageClass
> metadata:
>   name: fast
>   annotations:
>     storageclass.kubernetes.io/is-default-class: "true"   # make it the default
> provisioner: kubernetes.io/no-provisioner   # static/local PVs only; or a CSI driver name
> volumeBindingMode: WaitForFirstConsumer
> allowVolumeExpansion: true
> ```
>
> - Only **one** default StorageClass should exist. Switch: `kubectl patch sc old -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'`.
> - `volumeBindingMode`, `provisioner` and `reclaimPolicy` of a StorageClass **cannot be edited** after creation (delete and recreate).
> - **PVC Pending checklist:** no matching PV/StorageClass name → no default class → provisioner/CSI driver not installed or crashing → access mode or size not available → `WaitForFirstConsumer` with no Pod yet. Always read `kubectl describe pvc` → *Events*.
> - Other volume types you should recognise: `emptyDir` (Pod-lifetime scratch space), `hostPath` (node directory; avoid in production), `configMap` / `secret` / `projected`, `persistentVolumeClaim`.

---

# Part 5 — Troubleshooting (30%)

Highest-weight exam domain.

> 💡 **Method:** work **top-down and read the messages**. Most tasks are solved by one of: `kubectl describe` (Events at the bottom), `kubectl logs --previous`, `journalctl -u kubelet`, or fixing a typo in a manifest. Ask in order: *Is the node healthy? → Is the control plane healthy? → Is the Pod scheduled and running? → Does the Service have endpoints? → Does DNS/NetworkPolicy allow it?*

## 5.1 Fast diagnostic sequence

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

> 💡 **Shortcuts**
>
> ```bash
> kubectl get pods -A --field-selector=status.phase!=Running      # only unhealthy Pods
> kubectl get events -A --field-selector type=Warning --sort-by=.lastTimestamp
> kubectl get pod <pod> -o yaml                                   # full spec + status (check containerStatuses, conditions)
> kubectl exec -it <pod> -- sh                                    # inside a running container (add -c <name> if several)
> kubectl debug -it <pod> --image=busybox --target=<container>    # ephemeral debug container (works for distroless images)
> kubectl debug node/<node> -it --image=busybox                   # node shell; host filesystem at /host
> ```
>
> `kubectl describe` prints **Events at the very bottom**: scroll there first. Events are only kept for about an hour.

## 5.2 Common Pod states

| Symptom | First checks |
|---|---|
| `Pending` | Scheduling events, requests, taints, PVC |
| `ImagePullBackOff` | Image name/tag, registry access, imagePullSecrets |
| `CrashLoopBackOff` | `kubectl logs --previous`, command, probes, configuration |
| `ContainerCreating` | Events, CNI, CSI/volume mounts, runtime |
| `OOMKilled` | Memory limits and application usage |
| `CreateContainerConfigError` | Missing Secret, ConfigMap, or referenced key |

> 💡 **More states and what they usually mean**
>
> | Symptom | Usual cause |
> |---|---|
> | `ErrImagePull` | First failure of the pull (becomes `ImagePullBackOff` on retries) |
> | `Init:0/1`, `Init:Error` | An init container is running or failing: `kubectl logs <pod> -c <init>` |
> | `Pending` + empty events | No scheduler running (check `kube-scheduler` Pod/manifest) |
> | `Evicted` | Node pressure (memory/disk); check `kubectl describe node` conditions |
> | `Terminating` forever | Finalizers or an unreachable node: `kubectl delete pod <p> --force --grace-period=0` |
> | `RunContainerError` / `StartError` | Bad command/args or entrypoint, wrong permissions |
> | `Running` but `0/1 READY` | Readiness probe failing → not in Service endpoints |
> | `Completed` | Normal for Jobs / one-shot containers |
>
> **Exit codes** (`kubectl describe pod` → *Last State*): `0` success · `1` application error · `126` command not executable · `127` command not found · `137` SIGKILL (**OOMKilled** or force-killed) · `143` SIGTERM (graceful stop).
>
> A Deployment that makes no Pods: `kubectl describe rs <rs>` / `kubectl get events` show quota, PodSecurity or admission errors.

## 5.3 Node and control-plane checks

```bash
sudo systemctl status kubelet
sudo systemctl status containerd
sudo journalctl -u kubelet -xe
sudo journalctl -u containerd -xe
sudo crictl ps -a
sudo crictl pods
```

Get kubelet logs (follow):

```bash
journalctl -u kubelet -f
```

- Static Pod manifests: `/etc/kubernetes/manifests/`
- Control-plane components commonly run as static Pods.
- If a control-plane component is unhealthy, inspect its Pod, logs, manifest, and kubelet.
- Check node conditions with `kubectl describe node`.

> 💡 **Node `NotReady` playbook** (`ssh` to the node first)
>
> 1. `systemctl status kubelet`: dead? → `sudo systemctl enable --now kubelet` (or `restart`).
> 2. Kubelet crash-looping → `journalctl -u kubelet -xe | tail -50`. Typical causes: **swap enabled**, wrong path in the unit/drop-in (`systemctl cat kubelet`), typo in `/var/lib/kubelet/config.yaml`, bad `--config`/kubeconfig path (`/etc/kubernetes/kubelet.conf`), expired or wrong certificates, the wrong CA/`server:` address in `kubelet.conf`.
> 3. After editing a unit file: `sudo systemctl daemon-reload && sudo systemctl restart kubelet`.
> 4. `systemctl status containerd` and `crictl ps` for the runtime.
> 5. `kubectl describe node` → Conditions: `MemoryPressure`, `DiskPressure`, `PIDPressure`, `NetworkUnavailable` (CNI). Check `df -h` / `free -h`.
> 6. `Ready` condition message **`container runtime network not ready` / `cni plugin not initialized`** → CNI missing or broken.

> 💡 **Control-plane playbook**
>
> - **`kubectl` says `connection refused` / `The connection to the server ... was refused`:** API server (or etcd) is down, or your kubeconfig points at the wrong host/port (API server = **6443**). On the control-plane node: `sudo crictl ps -a | grep -E 'apiserver|etcd'`, then `sudo crictl logs <id>`; kubelet log: `journalctl -u kubelet | tail`.
> - **Manifest typo:** after any edit in `/etc/kubernetes/manifests/*.yaml` the kubelet restarts that static Pod; a wrong flag, wrong certificate path, wrong `--etcd-servers`, or a wrong `hostPath` makes it crash. Compare with a healthy sibling manifest and `crictl logs`.
> - **Scheduler down:** new Pods stay `Pending` with **no events**. **Controller-manager down:** Deployments don't create ReplicaSets/Pods, nodes never go NotReady, Jobs don't run.
> - **etcd down:** API server errors. Check etcd's `--data-dir` and volume paths in its manifest.
> - **Logs when `kubectl logs` can't work:** `/var/log/pods/` and `/var/log/containers/` on the node, or `crictl logs`.
> - Expired certs: `sudo kubeadm certs check-expiration` → `sudo kubeadm certs renew all` → restart control-plane static Pods.

### Debug pod, CSR, services, logs

```bash
kubectl run debug --image=busybox -it --rm --restart=Never -- sh

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

> 💡 `ss -tulpn` shows which process listens on which port (e.g. `6443` kube-apiserver, `10250` kubelet, `2379` etcd). Pending kubelet-serving or client CSRs after a certificate problem show up in `kubectl get csr`.

## 5.4 API server health

```bash
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/livez?verbose'
```

- `readyz`: whether the API server is ready to serve requests.
- `livez`: whether the API server is alive.

> 💡 `/healthz` is deprecated in favour of `/livez` and `/readyz`. `kubectl get --raw='/readyz?verbose'` lists each check with `[+]` / `[-]`, so a failing etcd shows up immediately.

## 5.5 Metrics server and `kubectl top`

Installation:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Disable TLS verification:

```bash
kubectl edit deployment metrics-server -n kube-system
```

Add to args:

```yaml
  - --kubelet-insecure-tls
```

Then restart:

```bash
kubectl rollout restart deployment/metrics-server -n kube-system
```

Get top pods by CPU, by memory, alphabetically:

```bash
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
kubectl top pods --sort-by=.metadata.name
```

> 💡 **Tips**
>
> - Saving the edit already triggers a new rollout, so the explicit `rollout restart` is only needed if you changed something outside the Deployment spec.
> - `--kubelet-insecure-tls` is acceptable for labs only (kubelet serving certs not signed by the cluster CA).
> - `error: Metrics API not available` → check `kubectl get apiservice v1beta1.metrics.k8s.io` (must be `Available=True`), the metrics-server Pod logs, and wait ~1 minute after installing.
> - Handy variants: `kubectl top pods -A --sort-by=memory`, `kubectl top pod <p> --containers`, `kubectl top nodes`.

## 5.6 Network troubleshooting

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

See also: [Service DNS test results](#32-service-dns), [CoreDNS](#33-coredns-corefile), [CNI](#cni--container-network-interface).

> 💡 **"Service not reachable" checklist (in this order)**
>
> 1. `kubectl get endpoints <svc>`: **empty?** → selector doesn't match Pod labels (`kubectl get pods --show-labels`, `kubectl get svc <svc> -o wide`) or Pods aren't Ready.
> 2. `targetPort` equals the port the app really listens on?
> 3. Pod-to-Pod by IP works, but by name fails → **DNS**: CoreDNS Pods running? `kubectl logs -n kube-system -l k8s-app=kube-dns`.
> 4. Direct by IP also fails → **NetworkPolicy** or **CNI** (`kubectl get networkpolicy -A`, CNI Pods in `kube-system`).
> 5. ClusterIP works only from some nodes → **kube-proxy**: `kubectl get ds -n kube-system kube-proxy`, `kubectl logs -n kube-system -l k8s-app=kube-proxy`.
> 6. NodePort/Ingress/Gateway: check the controller Pods, IngressClass/GatewayClass, and node firewalls.
>
> Bypass the Service to isolate the layer: `kubectl port-forward pod/<pod> 8080:<containerPort>`. Same-name Service in the wrong namespace is a classic mistake.

## 5.7 Logs

```bash
kubectl logs -l app=payments <options>
```

| Option | Meaning |
|---|---|
| `--all-pods` | Get logs from all Pods of the resource (e.g. `deployment/x`); implies `--prefix` |
| `--all-containers` | Include every container in the Pod (incl. init/sidecars) |
| `-c`, `--container string` | Logs of one named container |
| `-f`, `--follow` | Stream logs live |
| `--ignore-errors` | While following, keep going if a Pod's stream errors out |
| `--insecure-skip-tls-verify-backend` | Skip kubelet serving-cert verification (e.g. expired cert) |
| `--limit-bytes int` | Maximum bytes to return |
| `--prefix` | Prefix each line with `[pod/container]` |
| `-p`, `--previous` | Logs of the previous (crashed) container instance |
| `-l`, `--selector string` | Label selector to choose Pods |
| `--since duration` | Only logs newer than e.g. `5m`, `1h` |
| `--tail int` | Last N lines. Default `-1` (all); **with `-l`, the default is 10** |

> 💡 **More options:** `--timestamps`, `--since-time=<RFC3339>`, `--max-log-requests` (default **5**: with `-l` and more than 5 Pods you get an error; raise it, e.g. `--max-log-requests=20`), `--pod-running-timeout`.
> You can target controllers directly: `kubectl logs deploy/web`, `kubectl logs job/backup`, `kubectl logs ds/agent -c <container>`.

Log tricks:

```bash
# Get logs from first pod
kubectl logs $(kubectl get pods --no-headers | head -n 1 | awk '{print $1}')

# Follow logs
kubectl logs -f POD_NAME

# Previous container logs
kubectl logs POD_NAME --previous

# Get container names in first pod
kubectl get pod POD_NAME -o jsonpath='{.spec.containers[*].name}'

# Get images in all containers
kubectl get pod POD_NAME -o jsonpath='{.spec.containers[*].image}'
```

> 💡 Redirect to a file when the task asks for it: `kubectl logs <pod> --since=1h --timestamps > /opt/logs/pod.log`. Write to the **exact path** requested and check it with `cat`/`wc -l`.

---

# Part 6 — Tools, Tricks & Reference

## 6.1 kubectl tricks

Get top 1 in a list:

```bash
kubectl get pods -o jsonpath='{.items[0].metadata.name}'
```

Or use `head` (and `tail` for the last one):

```bash
kubectl get pods --no-headers | head -n 1
```

Extract specific columns with awk (`$1` = first column, `$2` = second column):

```bash
kubectl get pods --no-headers | awk '{print $1}'
```

Count resources:

```bash
kubectl get pods --no-headers | wc -l
```

For log-related tricks, see [Logs](#57-logs).

> 💡 **More output and query tricks**
>
> ```bash
> kubectl get pods -o name                                   # pod/xyz lines, easy to loop over
> kubectl get pods -o wide --show-labels
> kubectl get pods --sort-by=.metadata.creationTimestamp
> kubectl get pods -A --field-selector spec.nodeName=<node>  # all Pods on one node
> kubectl get pods -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[*].image,NODE:.spec.nodeName'
> kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
> kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
> kubectl get deploy web -o json | jq '.spec.template.spec.containers[].image'
> kubectl api-resources                                      # kinds, short names (po, svc, deploy, pv, pvc, sc, ns), scope
> kubectl api-resources --namespaced=false                   # cluster-scoped kinds
> kubectl get all -n <ns>                                    # core workload kinds only (not ConfigMaps, PVCs, Ingress...)
> ```
>
> **Generate, edit, apply (the exam workflow)**
>
> ```bash
> kubectl run web --image=nginx $do > pod.yaml               # export do="--dry-run=client -o yaml"
> kubectl create deployment web --image=nginx --replicas=3 $do > deploy.yaml
> kubectl expose deployment web --port=80 $do
> kubectl edit deployment web                                # edit live object
> kubectl patch deployment web -p '{"spec":{"replicas":5}}'
> kubectl replace --force -f pod.yaml                        # delete + recreate (immutable fields)
> kubectl wait --for=condition=Ready pod/web --timeout=60s
> kubectl cp <pod>:/path/file ./file                         # copy files out of a Pod
> kubectl delete pod web $now                                # export now="--force --grace-period=0"
> ```

## 6.2 `kubectl explain`

CKA exam tip: when unsure about YAML syntax, use:

```bash
kubectl explain <resource>.<field> --recursive
```

Basic commands:

```bash
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
```

Practical example: Pod securityContext

```bash
kubectl explain pod.spec.securityContext
kubectl explain pod.spec.containers.securityContext
```

Find fields such as `runAsUser`, `runAsGroup`, `privileged`, and `readOnlyRootFilesystem`.

Practical example: Probes

```bash
kubectl explain pod.spec.containers.livenessProbe
kubectl explain pod.spec.containers.readinessProbe
kubectl explain pod.spec.containers.startupProbe
```

Useful options:

```bash
# Specify API version
kubectl explain deployment --api-version=apps/v1
```

> 💡 `kubectl explain ... --recursive | less` and then `/fieldName` to search is faster than browsing. `kubectl explain` also works for CRDs (`kubectl explain database.spec`), and the `--help` text of `kubectl create <thing> -h` contains ready-to-copy examples.

## 6.3 Linux commands

### Keep containers running / loops

| Command | Purpose |
|---|---|
| `sleep infinity` | Keep container running indefinitely |
| `sleep 3600` | Sleep for 1 hour |
| `tail -f /dev/null` | Keep container running indefinitely |
| `while true; do sleep 3600; done` | Infinite loop |
| `sh -c 'while true; do echo hello; sleep 5; done'` | Repeatedly print output |

> 💡 **`command` vs `args`:** `command` replaces the image's `ENTRYPOINT`; `args` replaces its `CMD`. With `kubectl run`, words after `--` become **args**; add `--command` to make them the **command**:
>
> ```bash
> kubectl run tmp --image=busybox -- sleep 3600              # args: ["sleep","3600"]
> kubectl run tmp --image=busybox --command -- sleep 3600    # command: ["sleep","3600"]
> ```
>
> `sleep infinity` isn't supported by very old busybox builds; `tail -f /dev/null` works everywhere.

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

> 💡 `kubectl run curl --image=curlimages/curl -it --rm --restart=Never -- sh` gives you a Pod with `curl`. Handy flags: `-s` (silent), `-m 3` (3 s timeout), `-o /dev/null -w '%{http_code}\n'` (status code only), `-H 'Host: app.example.com'`.

### Ubuntu/Alpine (if utilities are installed)

```bash
ip addr
ip route
cat /etc/resolv.conf
cat /etc/hosts
printenv
```

### Useful tools

- `jq`
- `dnsutils`
- `net-tools`
- `tmux`

### How to run a sh script

```bash
sh /script.sh
# or
chmod +x /script.sh
/script.sh
```

> 💡 **Process / network quick checks on a node:** `ss -tulpn | grep 6443`, `ps aux | grep kubelet`, `systemctl cat <unit>`, `journalctl -u <unit> --since "10 min ago"`, `grep -R "pattern" /etc/kubernetes/`.

## 6.4 Nginx basic config

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

> 💡 Reload without downtime: `nginx -s reload`. To ship a config via Kubernetes, put it in a ConfigMap and mount it with `subPath` over `/etc/nginx/nginx.conf` (or mount a directory at `/etc/nginx/conf.d/`).

## 6.5 OpenSSL & certificate checks

**Local certificate file (`.pem`, `.crt`, `.cer`)**

Show only the expiration date:

```bash
openssl x509 -in cert.pem -noout -enddate
```

Show both start (`notBefore`) and expiration (`notAfter`) dates:

```bash
openssl x509 -in cert.pem -noout -dates
```

> 💡 **More certificate commands**
>
> ```bash
> openssl x509 -in cert.pem -noout -subject -issuer                 # who it is / who signed it
> openssl x509 -in cert.pem -noout -text | grep -A1 "Subject Alternative Name"   # SANs
> openssl x509 -in cert.pem -noout -checkend 86400 && echo ok       # exits 1 if it expires within 24h
> openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates  # real cluster example
> echo | openssl s_client -connect <host>:443 2>/dev/null | openssl x509 -noout -dates   # a live server's cert
> sudo kubeadm certs check-expiration                                # all kubeadm-managed certs at once
> kubectl get secret app-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates   # cert inside a TLS Secret
> ```
>
> The dot in `tls.crt` must be escaped (`tls\.crt`) in jsonpath.

## 6.6 Miscellaneous

### 6.6.1 Remove a label

```bash
kubectl label pod <pod-name> <label-key>-
kubectl label node <node-name> <label-key>-
```

(The trailing `-` removes the label. The same trick works for annotations: `kubectl annotate pod <pod> <key>-`, and taints: `kubectl taint nodes <node> <key>:<effect>-`.)

### 6.6.2 Create a service

```bash
kubectl create service clusterip svc-ip --tcp=80:8080
```

It creates a ClusterIP Service named `svc-ip`.

Understand `--tcp=80:8080`: `--tcp=<service-port>:<target-port>`

| Parameter | Value | Meaning |
|---|---|---|
| Service name | `svc-ip` | Service name |
| Type | `ClusterIP` | Internal cluster access |
| Protocol | `TCP` | TCP protocol |
| Port | `80` | Service port |
| TargetPort | `8080` | Backend container port |

> ⚠️ **Trap:** `kubectl create service` sets the selector to `app=<service-name>` (here `app=svc-ip`), which will **not** match your Pods unless they carry that label. `kubectl expose deployment/pod ...` copies the real selector instead. Fix a mismatch with `kubectl set selector svc svc-ip app=web` or edit the Service. Always verify with `kubectl get endpoints svc-ip`.
>
> Other types: `kubectl create service nodeport web --tcp=80:8080 --node-port=30080`, `kubectl create service externalname ext --external-name=api.example.com`.

### 6.6.3 Vim essentials (exam editor)

| Keys | Action |
|---|---|
| `i` / `Esc` | Insert mode / back to normal mode |
| `:wq` · `:q!` | Save and quit · quit without saving |
| `dd` · `yy` · `p` · `u` | Delete line · copy line · paste · undo |
| `Shift+V`, move, then `>` | Select lines and indent them (uses `shiftwidth`) |
| `5>>` | Indent 5 lines |
| `:set paste` | Stop auto-indent mangling when pasting YAML (then `:set nopaste`) |
| `/text` · `n` | Search · next match |
| `:%s/old/new/g` | Replace all |
| `:set list` | Show hidden characters (spot tabs in YAML) |

---

*End of notes.*
