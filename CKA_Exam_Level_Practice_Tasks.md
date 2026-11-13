# CKA Practice Exam — Hands-on Tasks

**Format:** Practical, task-based Kubernetes administration  
**Level:** CKA exam difficulty  
**Instructions:** Complete each task using `kubectl`, `kubeadm`, and YAML as appropriate. Use imperative commands where practical. Verify every change before moving on.

> These are practice scenarios, not a timed or officially scored CKA exam. Unless a task says otherwise, use the namespace and resource names specified. Do not delete or recreate existing resources when the task asks you to modify them.

---

## 1. etcd Snapshot Backup and Restore

**Control-plane node:** `control-node`  
**etcd certificates:** `/etc/kubernetes/pki/etcd/`

On `control-node`:

1. Create an etcd snapshot at `/opt/backup/etcd-snapshot.db`.
2. Restore that snapshot into `/var/lib/etcd-restore`.
3. Update the etcd static Pod manifest so etcd uses the restored data directory.
4. Verify that etcd is healthy and using the new data directory.

**Completion criteria**
- The snapshot file exists and passes `etcdutl snapshot status`.
- The etcd static Pod manifest references the restored data directory consistently.
- The etcd Pod becomes healthy.

## 2. RBAC — Role and RoleBinding for a ServiceAccount

In namespace `dev`, configure access for an automation identity.

1. Create ServiceAccount `ci-runner`.
2. Create Role `pod-reader` allowing `get`, `list`, and `watch` on Pods only.
3. Create RoleBinding `ci-runner-binding` that binds the Role to ServiceAccount `ci-runner`.
4. Verify the permissions using `kubectl auth can-i`.

**Required verification**
- The ServiceAccount can `get`, `list`, and `watch` Pods in `dev`.
- It cannot create or delete Pods.

## 3. Cluster Upgrade with kubeadm

The cluster is running Kubernetes `v1.35`. Upgrade the control-plane node to `v1.36`.

1. Upgrade the control-plane node using the supported kubeadm upgrade sequence.
2. Drain the control-plane node before upgrading its kubelet.
3. Upgrade kubelet and kubectl to the target version.
4. Restart kubelet.
5. Uncordon the control-plane node.
6. Verify the node and cluster component versions.

**Constraints**
- Follow the supported minor-version upgrade path.
- Do not skip the `kubeadm upgrade` phase.
- Preserve the existing cluster configuration.

## 4. Deployment Scheduling — Taint and Toleration

Node `node02` has this taint:

```text
env=prod:NoSchedule
```

Create Deployment `prod-web` in namespace `production` with:

- Image: `nginx:1.25`
- Replicas: `2`
- Both Pods must be schedulable on `node02`.

Add the required toleration. Verify that both Pods are running on `node02`.

## 5. Deployment Rolling Update and Rollback

Deployment `frontend` in namespace `default` currently runs `nginx:1.24`.

1. Update the image to `nginx:1.25-broken`.
2. Check the rollout status and identify that the rollout is unhealthy.
3. Roll back to the previous working revision.
4. Verify the Deployment is healthy and uses `nginx:1.24`.

## 6. ConfigMap as a Container Environment Variable

Create ConfigMap `app-config` in namespace `default` with:

```text
APP_ENV=production
```

Create Pod `app-pod` with:

- Image: `busybox:1.36`
- Command: `sleep 3600`
- Environment variable `APP_ENV` sourced from ConfigMap `app-config`.

Verify the value from inside the running container.

## 7. Expose a Deployment with a NodePort Service

Deployment `web` exists in namespace `default` and its Pods listen on port `80`.

Create Service `web-svc` with:

- Type: `NodePort`
- Service port: `80`
- Target port: `80`
- NodePort: `30080`

Verify the Service selector and endpoints, and confirm the assigned NodePort.

## 8. NetworkPolicy — Default Deny and Targeted Allow

In namespace `restricted`, create two NetworkPolicies.

### Policy A — Default deny ingress

Create `default-deny-ingress` to deny ingress traffic to all Pods in the namespace.

### Policy B — Allow frontend to backend

Create `allow-frontend` with these rules:

- Select destination Pods labelled `app=backend`.
- Allow ingress only from Pods labelled `app=frontend`.
- Allow traffic on TCP port `8080`.

Verify both policies and inspect their selectors and rules.

## 9. PersistentVolume and PersistentVolumeClaim

Create PersistentVolume `pv-logs` with:

- Capacity: `500Mi`
- Access mode: `ReadWriteOnce`
- Storage class: `manual`
- Host path: `/mnt/logs`

Create PVC `pvc-logs` in namespace `default` with:

- Requested capacity: `200Mi`
- Access mode: `ReadWriteOnce`
- Storage class: `manual`

Verify the PV and PVC binding.

## 10. Mount a PVC in a Pod

Create Pod `logger` in namespace `default` with:

- Image: `busybox:1.36`
- Command: `sleep 3600`
- PVC: `pvc-logs`
- Mount path: `/var/log/app`

Verify the Pod is running and the volume is mounted at the required path.

## 11. Ingress — Route Two Paths

Services already exist in namespace `default`:

- `api-svc`, Service port `8080`
- `web-svc`, Service port `80`

Create Ingress `app-ingress` with:

- `/api` → `api-svc:8080`
- `/` → `web-svc:80`
- Both paths use `pathType: Prefix`.

Verify the Ingress rules and backend references.

## 12. Pod with an Init Container

Create Pod `init-web` with:

**Application container**
- Name: `nginx`
- Image: `nginx`

**Init container**
- Name: `init-check`
- Image: `busybox`
- Command: `sh -c "sleep 10; echo initialization-complete"`

The init container must complete successfully before the Nginx container starts.

Verify the init container completed and the application container is running.

## 13. Pod with a Sidecar and Shared Volume

Create Pod `webapp` with two containers and a shared volume.

**Main container**
- Name: `main-app`
- Image: `nginx`
- Mount the shared volume at `/var/log/app`.

**Sidecar**
- Name: `sidecar`
- Image: `busybox`
- Command: `sh -c "tail -f /var/log/app/access.log"`
- Mount the same shared volume at `/var/log/app`.

**Volume**
- Name: `logs`
- Type: `emptyDir`

Verify both containers are running and both mount the same volume.

## 14. Modify an Existing Deployment to Add a Logging Sidecar

Deployment `webapp` already exists in namespace `default`.

Modify the existing Deployment; do not create a separate Pod or replace the application container.

1. Keep the existing application container.
2. Add an `emptyDir` volume named `logs`.
3. Mount `logs` at `/var/log/app` in both containers.
4. Configure the application container to write its log file to `/var/log/app/app.log`.
5. Add sidecar container `sidecar` using an appropriate image and command to run:

   ```sh
   tail -f /var/log/app/app.log
   ```

6. Apply the update and verify the sidecar can read the log file.

**Constraint:** The sidecar may start before the log file exists. Do not treat that initial `tail` error alone as task failure if the Pod becomes healthy and the sidecar subsequently reads the file.

## 15. PVC Binding and MariaDB Deployment

An existing PersistentVolume and a MariaDB Deployment manifest are provided.

**Given**
- PV: `mariadb-pv`
- Storage class: `manual`
- PVC to create: `mariadb-pvc`
- Namespace: `default`
- Requested storage: `1Gi`
- Access mode: `ReadWriteOnce`
- Deployment: `mariadb`

Tasks:

1. Inspect `mariadb-pv` and identify its capacity, access modes, and StorageClass.
2. Create PVC `mariadb-pvc` with compatible settings.
3. Confirm the PVC reaches `Bound`.
4. Update the provided Deployment manifest to use `mariadb-pvc`.
5. Apply the Deployment.
6. Verify the MariaDB Pod is running and the PVC is mounted at `/var/lib/mysql`.

**Constraint:** Do not modify or recreate the supplied PV unless required by the task environment.

## 16. Migrate Ingress to Gateway API

The following resources already exist in namespace `default`:

- Service: `web-service`, port `80`
- TLS Secret: `app-tls-secret`

### Part A — Create the existing Ingress

Create Ingress `web-ingress` with:

- Ingress class: `nginx`
- Host: `app.example.com`
- TLS Secret: `app-tls-secret`
- Path: `/`
- Path type: `Prefix`
- Backend: `web-service:80`

### Part B — Migrate to Gateway API

Create the equivalent Gateway API configuration using:

- A `Gateway` with an HTTPS listener for `app.example.com`.
- TLS termination using `app-tls-secret`.
- An `HTTPRoute` that attaches to the Gateway and routes `/` to `web-service:80`.

Verify the Gateway listener and HTTPRoute status conditions.

**Note:** The GatewayClass and Gateway API CRDs/controller must be available in the cluster. Use the installed API versions and controller configuration.

## 17. Certificate Signing Request for a Developer

A developer needs Kubernetes client-certificate authentication.

Create a CSR with:

- Username: `jane`
- Group: `developers`
- CSR object name: `jane-csr`
- Private key: RSA, 2048 bits
- Signer: `kubernetes.io/kube-apiserver-client`
- Requested expiration: `365` days

Tasks:

1. Generate the private key and certificate signing request.
2. Base64-encode the CSR request data as required by the Kubernetes CSR object.
3. Create the Kubernetes CSR object.
4. Inspect the CSR and approve it.
5. Retrieve the issued certificate and verify the CSR status.

**Security:** Keep the private key secure. Do not submit the private key to the Kubernetes CSR object.

## 18. CoreDNS — Restore Kubernetes Service DNS

External DNS resolution works, but Kubernetes Service names do not resolve. CoreDNS Pods are `Running`.

The CoreDNS ConfigMap is missing the Kubernetes DNS plugin configuration.

Tasks:

1. Inspect the CoreDNS ConfigMap in `kube-system`.
2. Edit the `Corefile` to restore the `kubernetes` plugin configuration for the cluster DNS domain.
3. Preserve the existing forwarding and other required Corefile directives.
4. Restart or allow CoreDNS to reload the configuration as appropriate.
5. Verify that a Pod can resolve a Kubernetes Service by DNS name.

**Important:** Use the cluster's actual DNS domain and preserve the existing CoreDNS configuration. Do not replace the entire Corefile with an unrelated default.

## 19. Job Failure and Retry Verification

Create Job `failure-job` with:

- Image: `busybox:1.36`
- Command: `sh -c "echo Starting; exit 1"`
- Completions: `1`
- Backoff limit: `3`

Tasks:

1. Create the Job.
2. Inspect the Job and its Pods while it retries.
3. Verify that the Job reaches the failed condition after exhausting retries.
4. Use Job status and Pod logs/events as evidence.

## 20. Job — Parallel Successful Completions

Create Job `batch-job` with:

- Image: `busybox:1.36`
- Command: `sh -c "echo CKA Job completed; sleep 5"`
- Completions: `3`
- Parallelism: `2`
- Backoff limit: `2`

Verify that the Job completes successfully with three successful completions.

---

## Final Verification Checklist

Before finishing a task, verify the actual cluster state rather than relying only on successful command output.

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get deploy,rs,sts,ds,job,cronjob -A
kubectl get svc,ep,endpointslices -A
kubectl get pv,pvc,sc
kubectl get ingress -A
kubectl get networkpolicy -A
kubectl get sa,role,rolebinding -A
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

### Exam habits

- Read the namespace, resource name, image, port, and replica count carefully.
- Check the current context and namespace before changing resources.
- Use `kubectl explain` when unsure about a YAML field.
- Prefer imperative commands for quick resource creation when they support the required configuration.
- Use YAML for complex resources and edits to existing workloads.
- Verify with `get`, `describe`, `logs`, `rollout status`, and `auth can-i`.
- Do not assume a resource is correct just because `kubectl apply` succeeded.
