# CKA Practice Tasks v2: Missing Topics (T1–T41)

**Format:** Practical, task-based Kubernetes administration  
**Level:** CKA exam difficulty  
**Companion to:** CKA_Exam_Level_Practice_Tasks.md (tasks 1–20)

> These are practice scenarios, not an officially scored exam. Task numbers (T1–T41) match the gap list from the review. Unless a task says otherwise, use the names given. Do not delete and recreate existing resources when the task asks you to modify them. Troubleshooting tasks assume a lab where the fault has been injected for you; if you build your own lab, break the component first, then fix it.

**Start every task with:**

```bash
kubectl config use-context <context-name>
kubectl config get-contexts
```

---

# Part A: Troubleshooting (≈30% of the exam)

## T1. Node NotReady: Diagnose and Fix

Node `node01` shows `NotReady`.

1. Confirm the status with `kubectl get nodes` and `kubectl describe node node01` (check Conditions).
2. SSH to `node01` and inspect `systemctl status kubelet` and `journalctl -u kubelet --no-pager | tail -50`.
3. Identify the root cause (stopped service, disabled service, wrong runtime socket, or bad config).
4. Fix it, and make sure kubelet survives a reboot (`systemctl enable`).
5. Verify the node returns to `Ready`.

**Completion criteria**
- `node01` is `Ready`.
- `systemctl is-enabled kubelet` returns `enabled` and `is-active` returns `active`.
- You can state the root cause in one sentence.

## T2. Broken kubelet Configuration

The kubelet on `node02` is crash-looping.

1. Read the kubelet logs with `journalctl -u kubelet`.
2. Inspect `/var/lib/kubelet/config.yaml` and the kubelet drop-in under `/usr/lib/systemd/system/kubelet.service.d/` (or `/etc/systemd/system/kubelet.service.d/`).
3. Find the faulty value (for example a wrong `clientCAFile` path, bad `staticPodPath`, or wrong `--container-runtime-endpoint`).
4. Fix it, run `systemctl daemon-reload`, restart kubelet.
5. Verify with `crictl ps` and `kubectl get nodes`.

**Completion criteria**
- kubelet is `active (running)` and stable for at least 60 seconds.
- `node02` is `Ready`.
- Existing Pods on the node are not deleted by you.

## T3. kube-apiserver Down

`kubectl` returns `connection refused` or times out.

1. On `control-node`, run `crictl ps -a | grep kube-apiserver` and read the failing container's logs with `crictl logs <id>`.
2. Inspect `/etc/kubernetes/manifests/kube-apiserver.yaml` for a bad flag, wrong certificate path, wrong `--etcd-servers` URL, or YAML error.
3. Correct the manifest in place. Keep a backup copy outside the manifests directory (never leave `.bak` files inside it).
4. Wait for kubelet to recreate the static Pod.
5. Verify with `kubectl get pods -n kube-system`.

**Completion criteria**
- `kubectl get nodes` works again.
- The kube-apiserver static Pod is `Running` and not restarting.
- No stray backup files remain in `/etc/kubernetes/manifests/`.

## T4. kube-scheduler Down

New Pods stay `Pending` with no scheduling events.

1. Create a test Pod `sched-test` (`nginx`) and confirm it is `Pending` with an empty Events section.
2. Check `kubectl get pods -n kube-system | grep scheduler`, then `crictl logs` on the control-plane node if the API is reachable only partly.
3. Inspect `/etc/kubernetes/manifests/kube-scheduler.yaml` (typical faults: wrong image tag, bad `--kubeconfig` path, typo in a flag, or a wrong command).
4. Fix the manifest.
5. Confirm `sched-test` gets scheduled.

**Completion criteria**
- The scheduler static Pod is `Running`.
- `sched-test` is `Running` and shows a `Scheduled` event.
- Delete `sched-test` afterwards.

## T5. Pending Pod Diagnosis

Pod `report-gen` in namespace `ops` is `Pending`.

1. Run `kubectl describe pod report-gen -n ops` and read the Events.
2. Determine the cause. Possible causes: insufficient CPU/memory requests, an untolerated taint, a `nodeSelector` that matches no node, or an unbound PVC.
3. Fix the cause **without recreating the namespace**. You may edit the Pod's controller or related objects (node label, PVC, requests).
4. Verify the Pod becomes `Running`.

**Completion criteria**
- `report-gen` is `Running`.
- You identified the cause from the Events message, not by guessing.
- Other workloads in `ops` were not disrupted.

## T6. CrashLoopBackOff Investigation

Deployment `orders` in namespace `shop` has Pods in `CrashLoopBackOff`.

1. Use `kubectl logs <pod> --previous` and `kubectl describe pod <pod>` (check Last State, Exit Code, Events).
2. Find the cause: bad `command`/`args`, a missing ConfigMap key, a wrong environment variable, or an aggressive liveness probe.
3. Fix the **Deployment** (not the individual Pod) using `kubectl edit` or `kubectl set`.
4. Verify with `kubectl rollout status`.

**Completion criteria**
- All replicas are `Running` and `READY`.
- Restart count stops increasing.
- The fix is in the Deployment spec.

## T7. Service with No Endpoints

Service `catalog` in namespace `shop` returns connection errors. `kubectl get ep catalog -n shop` shows `<none>`.

1. Compare the Service `selector` with the Pod labels (`kubectl get pods --show-labels`).
2. Check `port` versus `targetPort` against the container's real listening port.
3. Check that the Pods are `Ready` (a failing readiness probe also empties endpoints).
4. Fix the mismatch on the **Service** if the selector or `targetPort` is wrong. Do not relabel Pods unless the Pods are the faulty side.
5. Verify endpoints are populated and test with a temporary Pod using `curl`.

**Completion criteria**
- `kubectl get ep catalog -n shop` lists Pod IPs.
- `curl catalog.shop.svc.cluster.local` from a test Pod returns a response.

## T8. kube-proxy Repair

Service ClusterIPs are unreachable from Pods on `node02`, but Pod-to-Pod traffic works.

1. Check `kubectl get ds kube-proxy -n kube-system` and the Pod on `node02`.
2. Read the logs: `kubectl logs -n kube-system <kube-proxy-pod>`.
3. Inspect the ConfigMap `kube-proxy` in `kube-system` (check `mode`, `clusterCIDR`, and the kubeconfig reference).
4. Fix the faulty setting, then restart the DaemonSet with `kubectl rollout restart`.
5. Verify Service access from a Pod on `node02`.

**Completion criteria**
- kube-proxy is `Running` on every node.
- A Pod pinned to `node02` can reach a ClusterIP Service.

## T9. Certificate Expiry and Renewal

The cluster certificates are close to expiring.

1. Run `kubeadm certs check-expiration` and record the nearest expiry.
2. Renew all certificates with `kubeadm certs renew all`.
3. Restart the control-plane static Pods (move the manifests out of the manifests directory for a short time and back, or restart kubelet, as appropriate to the lab).
4. Update your admin kubeconfig if required (`/etc/kubernetes/admin.conf`).
5. Verify new expiry dates and API health.

**Completion criteria**
- `check-expiration` shows renewed dates (about one year out).
- `kubectl get nodes` and `kubectl get pods -n kube-system` work.

## T10. Resource Usage with kubectl top

metrics-server is installed.

1. In namespace `monitoring`, identify the Pod with the highest CPU usage.
2. Write **only its name** to `/opt/answers/top-cpu-pod.txt`.
3. Also write the name of the node with the highest memory usage to `/opt/answers/top-mem-node.txt`.

Useful: `kubectl top pods -n monitoring --sort-by=cpu`, `kubectl top nodes --sort-by=memory`.

**Completion criteria**
- Both files exist and each contains one name and nothing else.
- The names match live `kubectl top` output.

## T11. Collect Component Logs and Events

A Deployment `payments` in namespace `shop` is intermittently failing.

1. Save logs of all its Pods (all containers) to `/opt/answers/payments.log`.
2. Save the namespace events, sorted by time, to `/opt/answers/shop-events.txt`.
3. Save the kubelet log of the node running the most `payments` Pods to `/opt/answers/kubelet.log` (last 200 lines).

Useful: `kubectl logs -l app=payments --all-containers --prefix`, `kubectl get events -n shop --sort-by=.metadata.creationTimestamp`, `journalctl -u kubelet -n 200`.

**Completion criteria**
- All three files exist and are non-empty.
- The log file contains output from every replica.

---

# Part B: Cluster Architecture, Installation and Configuration

## T12. Join a Worker Node

Add `node03` to the cluster.

1. On `control-node`, generate a join command with `kubeadm token create --print-join-command`.
2. Confirm containerd and kubeadm/kubelet are already installed on `node03`; install them if the lab requires it.
3. Run the join command on `node03`.
4. Verify the node registers and becomes `Ready` (the CNI must already be installed).
5. Label the node `role=worker`.

**Completion criteria**
- `node03` appears in `kubectl get nodes` as `Ready`.
- The label `role=worker` is set.
- You can explain the token's default lifetime (24 hours).

## T13. ClusterRole and ClusterRoleBinding

In namespace `ops`, ServiceAccount `monitor` needs read access to Nodes cluster-wide.

1. Create ServiceAccount `monitor` in `ops`.
2. Create ClusterRole `node-reader` allowing `get`, `list`, `watch` on `nodes`.
3. Create ClusterRoleBinding `monitor-node-reader` binding the role to the ServiceAccount.
4. Verify with `kubectl auth can-i list nodes --as=system:serviceaccount:ops:monitor`.

**Required verification**
- `get`, `list`, `watch` on nodes: `yes`.
- `delete nodes` and `list pods -n ops`: `no`.

## T14. Kubeconfig Contexts

Using the client certificate from the existing task 17 (user `jane`):

1. Add credentials `jane` to your kubeconfig with `kubectl config set-credentials` using `--client-certificate` and `--client-key` (use `--embed-certs=true`).
2. Create context `dev-ctx` for cluster `kubernetes`, user `jane`, namespace `dev`.
3. List all contexts.
4. Write the current context name to `/opt/answers/current-context.txt`.
5. Switch to `dev-ctx` and confirm `kubectl get pods` targets namespace `dev` (expect a Forbidden error until RBAC is granted).

**Completion criteria**
- `dev-ctx` exists with the correct user and namespace.
- The answer file contains the correct context name.
- Switch back to your admin context afterwards.

## T15. Helm Lifecycle

1. Add the `bitnami` repo: `helm repo add bitnami https://charts.bitnami.com/bitnami` and run `helm repo update`.
2. Install chart `bitnami/nginx` as release `web` in a **new** namespace `helm-demo` (`--create-namespace`) with `replicaCount=2` via `--set` or a values file.
3. Upgrade the release to `replicaCount=3`.
4. View the history and roll back to revision 1.
5. Uninstall the release.

**Completion criteria**
- `helm history web -n helm-demo` shows install, upgrade and rollback entries before uninstall.
- After rollback the Deployment has 2 replicas.
- After uninstall no release remains in `helm list -n helm-demo`.

## T16. Kustomize Overlay

A base exists in `~/kustomize/base` with Deployment `app` (`nginx:1.24`, 1 replica).

1. Create an overlay in `~/kustomize/overlays/prod`.
2. The overlay must: set replicas to `3`, add `namePrefix: prod-`, change the image to `nginx:1.25` (use `images:`), and add a label `env=prod`.
3. Preview with `kubectl kustomize ~/kustomize/overlays/prod`.
4. Apply with `kubectl apply -k`.

**Completion criteria**
- Deployment `prod-app` exists with 3 replicas and image `nginx:1.25`.
- The base directory is unchanged.

## T17. Install a CNI Plugin

A fresh kubeadm cluster has nodes in `NotReady` and CoreDNS Pods in `Pending`.

1. Confirm the cause (`kubectl describe node`: network plugin not ready).
2. Check the Pod CIDR used at `kubeadm init` (`kubectl cluster-info dump | grep -m1 cluster-cidr`).
3. Install Calico or Flannel using the manifest provided in the lab, ensuring its CIDR matches the cluster's.
4. Wait for the CNI Pods to become `Running`.
5. Verify nodes become `Ready` and CoreDNS starts.

**Completion criteria**
- All nodes `Ready`.
- CoreDNS Pods `Running`.
- Two Pods on different nodes can ping each other by Pod IP.

## T18. CustomResourceDefinitions

A CRD `backups.stable.example.com` already exists.

1. List all CRDs in the cluster.
2. Use `kubectl explain backups.spec` to read its schema.
3. Create a `Backup` custom resource named `nightly` in `default` with fields that satisfy the schema.
4. List Backups and describe `nightly`.
5. Write the CRD's API group and kind to `/opt/answers/crd.txt`.

**Completion criteria**
- `kubectl get backups` lists `nightly`.
- The resource passes schema validation (no apply errors).

## T19. Pod Security Admission

1. Create namespace `secure`.
2. Label it to enforce the `restricted` Pod Security Standard: `pod-security.kubernetes.io/enforce=restricted`.
3. Try to create a privileged Pod there (`securityContext.privileged: true`) and capture the rejection message.
4. Create a compliant Pod (non-root, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `seccompProfile.type: RuntimeDefault`).

**Completion criteria**
- The privileged Pod is rejected with a PodSecurity violation.
- The compliant Pod is `Running`.

---

# Part C: Workloads and Scheduling

## T20. Horizontal Pod Autoscaler

Deployment `web` exists in namespace `default` with CPU requests set.

1. Confirm metrics-server works with `kubectl top pods`.
2. Create an HPA for `web`: CPU target `50%`, min `2`, max `6` (`kubectl autoscale deployment web --cpu-percent=50 --min=2 --max=6`).
3. Generate load from a temporary busybox Pod (a `wget` loop against the Service).
4. Watch `kubectl get hpa -w` and observe scaling.

**Completion criteria**
- `kubectl get hpa web` shows real utilisation (not `<unknown>`).
- Replicas stay within 2 to 6.
- If the target shows `<unknown>`, you can explain why (missing requests or metrics-server).

## T21. Resource Requests, ResourceQuota and LimitRange

In a new namespace `limited`:

1. Create ResourceQuota `team-quota`: `requests.cpu=1`, `requests.memory=1Gi`, `limits.cpu=2`, `limits.memory=2Gi`, `pods=5`.
2. Create LimitRange `defaults`: default request `100m` CPU / `64Mi` memory, default limit `200m` CPU / `128Mi` memory.
3. Create a Pod with **no** resources set and confirm the defaults were injected.
4. Create a Pod requesting `3` CPUs and show it is denied.

**Completion criteria**
- The first Pod shows the LimitRange defaults in `kubectl get pod -o yaml`.
- The over-quota Pod is rejected with a `forbidden: exceeded quota` message.
- `kubectl describe quota -n limited` shows current usage.

## T22. Node Affinity

1. Label `node01` with `disk=ssd` and `node02` with `zone=a`.
2. Create Pod `affinity-pod` (`nginx`) with **required** node affinity for `disk=ssd` and **preferred** node affinity (weight `50`) for `zone=a`.
3. Verify placement with `kubectl get pod -o wide`.

**Completion criteria**
- The Pod runs on a node labelled `disk=ssd`.
- The YAML uses `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`.
- Explain why the preferred rule did not override the required one.

## T23. Pod Anti-Affinity and Topology Spread

Create Deployment `spread-web` (`nginx`, 3 replicas) in namespace `default`.

1. Use `podAntiAffinity` (`requiredDuringSchedulingIgnoredDuringExecution`, `topologyKey: kubernetes.io/hostname`) so no two replicas share a node.
2. Verify each Pod is on a different node.
3. Replace the anti-affinity with a `topologySpreadConstraints` entry (`maxSkew: 1`, `whenUnsatisfiable: DoNotSchedule`) and compare behavior.

**Completion criteria**
- 3 replicas on 3 different nodes.
- If the lab has fewer than 3 schedulable nodes, explain the `Pending` Pod using `describe`.

## T24. Static Pod

1. On `node01`, find the kubelet `staticPodPath` (`grep staticPodPath /var/lib/kubelet/config.yaml`).
2. Create a manifest there for Pod `static-web` (`nginx:1.25`).
3. Verify the mirror Pod on the API server (it will be named `static-web-node01`).
4. Try to delete the mirror Pod with `kubectl delete` and observe that it returns.
5. Remove the Pod properly by deleting the manifest file.

**Completion criteria**
- `kubectl get pod static-web-node01` shows `Running` on `node01`.
- After deleting the manifest, the Pod disappears.

## T25. DaemonSet on Control-Plane Nodes

Create DaemonSet `node-agent` in namespace `kube-system` with image `busybox:1.36` and command `sleep 3600`.

1. Add a toleration so Pods also run on control-plane nodes (taint `node-role.kubernetes.io/control-plane:NoSchedule`).
2. Verify one Pod per node, including `control-node`.

**Completion criteria**
- `DESIRED` equals the node count in `kubectl get ds node-agent -n kube-system`.
- A Pod runs on `control-node`.

## T26. Cordon, Drain and Uncordon

1. Cordon `node02` and confirm `SchedulingDisabled`.
2. Create Deployment `drain-test` (`nginx`, 4 replicas) and show no new Pods land on `node02`.
3. Drain `node02` safely: `kubectl drain node02 --ignore-daemonsets --delete-emptydir-data`.
4. Record which nodes the evicted Pods moved to.
5. Uncordon `node02`.

**Completion criteria**
- No non-DaemonSet Pods remain on `node02` after the drain.
- All 4 replicas stay `Running`.
- `node02` is schedulable again.

## T27. Secrets: Env Var and Volume

1. Create Secret `db-creds` in `default` with `username=admin` and `password=S3cr3t!`.
2. Create Pod `secret-pod` (`busybox:1.36`, `sleep 3600`) that consumes `password` as env var `DB_PASS` **and** mounts the whole Secret read-only at `/etc/db-creds`.
3. Verify both inside the container.
4. Decode the stored value from the API with `kubectl get secret db-creds -o jsonpath` and `base64 -d`.

**Completion criteria**
- `echo $DB_PASS` prints `S3cr3t!`.
- `/etc/db-creds/username` contains `admin`.
- Explain why base64 is encoding, not encryption.

## T28. Liveness and Readiness Probes

Create Deployment `probe-web` (`nginx:1.25`, 2 replicas).

1. Add an HTTP **readiness** probe (`GET /`, port 80, `initialDelaySeconds: 5`, `periodSeconds: 5`).
2. Add an exec **liveness** probe running `cat /usr/share/nginx/html/index.html`.
3. Break the readiness path (change it to `/missing`) and observe: Pods run but are not `Ready`, and they are removed from Service endpoints.
4. Fix the path.

**Completion criteria**
- You can explain the difference in effect: readiness removes the Pod from endpoints; liveness restarts the container.
- After the fix, all replicas are `Ready`.

## T29. CronJob

1. Create CronJob `reporter`: schedule `*/5 * * * *`, image `busybox:1.36`, command `sh -c "date; echo report"`, `successfulJobsHistoryLimit: 2`, `failedJobsHistoryLimit: 1`, `concurrencyPolicy: Forbid`.
2. Trigger it manually: `kubectl create job reporter-manual --from=cronjob/reporter`.
3. Check the Job's logs.
4. Suspend the CronJob.

**Completion criteria**
- `reporter-manual` completes and its logs contain `report`.
- `spec.suspend` is `true`.

## T30. Scaling and PodDisruptionBudget

1. Scale Deployment `web` to `3` replicas.
2. Create a PDB `web-pdb` with `minAvailable: 2` for the Pods of `web`.
3. Drain a node that hosts two `web` Pods and observe that evictions respect the PDB (eviction blocked or staggered).
4. Check `kubectl get pdb` for `ALLOWED DISRUPTIONS`.

**Completion criteria**
- `web-pdb` selects the correct Pods.
- You can explain why a drain may hang when `minAvailable` equals the replica count.
- Uncordon the node afterwards.

---

# Part D: Services and Networking

## T31. ClusterIP and DNS Across Namespaces

Service `api` exists in namespace `backend`.

1. Start a temporary Pod in namespace `default`: `kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh`.
2. Resolve `api.backend.svc.cluster.local` with `nslookup`.
3. Fetch it with `wget -qO- http://api.backend`.
4. Explain why the short name `api` fails from `default`.

**Completion criteria**
- The FQDN resolves to the Service ClusterIP.
- The HTTP request succeeds.
- Check `/etc/resolv.conf` in the Pod and explain the search domains.

## T32. NetworkPolicy with namespaceSelector and Egress

In namespace `data`, Pods labelled `app=db` must be protected.

1. Label namespace `payments` with `team=payments`.
2. Create NetworkPolicy `db-ingress`: allow ingress to `app=db` **only** from Pods in namespaces labelled `team=payments`, on TCP `5432`.
3. Create NetworkPolicy `db-egress`: for `app=db` Pods allow egress **only** to DNS (UDP/TCP `53` to the `kube-system` namespace) and to Pods labelled `app=backup` on TCP `9000`.
4. Test allowed and denied connections with temporary Pods.

**Completion criteria**
- A Pod in `payments` reaches `db:5432`; a Pod in `default` does not.
- `db` Pods can resolve DNS but cannot reach arbitrary destinations.
- Note: enforcement requires a CNI that supports NetworkPolicy.

## T33. Ingress with Hosts and TLS

Services `shop-svc:80` and `blog-svc:80` exist in `default`. TLS Secrets `shop-tls` and `blog-tls` exist.

1. Create Ingress `multi-host` with `ingressClassName: nginx`.
2. Route `shop.example.com` to `shop-svc:80` and `blog.example.com` to `blog-svc:80`, both `pathType: Prefix` on `/`.
3. Add a `tls` section for each host with its Secret.
4. Inspect with `kubectl describe ingress multi-host`.

**Completion criteria**
- Both host rules and TLS entries appear in `describe`.
- Backends resolve to existing Services (no `<error: services not found>`).

## T34. Headless Service and Per-Pod DNS

1. Create headless Service `db-hl` (`clusterIP: None`, port `80`, selector `app=db-sts`).
2. Create StatefulSet `db-sts` (`nginx`, 3 replicas, `serviceName: db-hl`).
3. From a temporary Pod, resolve `db-hl.default.svc.cluster.local` (expect 3 A records) and `db-sts-0.db-hl.default.svc.cluster.local`.

**Completion criteria**
- The headless lookup returns all Pod IPs.
- Each Pod has a stable DNS name.
- Explain the difference from a normal ClusterIP Service.

## T35. Gateway API: Header Match and Traffic Split

A Gateway `main-gw` exists in `default`. Services `app-v1` and `app-v2` listen on port `80`.

1. Create HTTPRoute `canary` attached to `main-gw` for host `app.example.com`.
2. Send traffic with a 90/10 split via `backendRefs` weights (`app-v1` 90, `app-v2` 10).
3. Add a second rule: requests with header `x-canary: true` go 100% to `app-v2`.
4. Verify `status.parents` conditions show `Accepted=True` and `ResolvedRefs=True`.

**Completion criteria**
- Both rules are present in the route.
- Status conditions are healthy.
- Use the installed Gateway API version (`kubectl api-resources | grep gateway`).

---

# Part E: Storage

## T36. StorageClass: Retain, Expansion, Default

1. Create StorageClass `fast-retain` using the cluster's provisioner (for example `rancher.io/local-path` or the lab-provided one) with `reclaimPolicy: Retain` and `allowVolumeExpansion: true`.
2. Mark it as the default class with the annotation `storageclass.kubernetes.io/is-default-class="true"`.
3. Make sure no other StorageClass is still marked default.

**Completion criteria**
- `kubectl get sc` shows exactly one `(default)`.
- `kubectl get sc fast-retain -o yaml` shows both settings.

## T37. Dynamic Provisioning

1. Create PVC `dyn-pvc` (`300Mi`, `ReadWriteOnce`) with **no** `storageClassName` field, so the default class is used.
2. Confirm a PV was provisioned and the PVC is `Bound` (if the class uses `WaitForFirstConsumer`, it binds after a Pod uses it).
3. Create Pod `dyn-pod` (`busybox:1.36`, `sleep 3600`) mounting it at `/data`, and write a file.

**Completion criteria**
- PVC `Bound` with an automatically created PV.
- The file is present inside the Pod at `/data`.

## T38. Expand a PVC

PVC `grow-pvc` (`1Gi`) is bound and used by Pod `grow-pod`.

1. Confirm the StorageClass has `allowVolumeExpansion: true`.
2. Patch the PVC request to `2Gi`.
3. Watch the PVC conditions (`FileSystemResizePending` may appear; some drivers need a Pod restart).
4. Verify the new size with `kubectl get pvc` and `df -h` inside the Pod.

**Completion criteria**
- `status.capacity.storage` is `2Gi`.
- The mounted filesystem reflects the new size.

## T39. Reclaim Policy and Reusing a Released PV

PV `pv-retain` has `persistentVolumeReclaimPolicy: Retain` and is bound to PVC `old-pvc`.

1. Delete `old-pvc`.
2. Observe that the PV becomes `Released`.
3. Make it available again by clearing the stale claim reference (`kubectl patch pv pv-retain -p '{"spec":{"claimRef":null}}'`).
4. Create a new PVC that binds to it.

**Completion criteria**
- PV status moves `Bound` to `Released` to `Available` to `Bound`.
- You can state what `Delete` would have done instead.

## T40. StatefulSet with volumeClaimTemplates

1. Create StatefulSet `store` (`nginx`, 3 replicas, `serviceName: store-hl`) with a `volumeClaimTemplate` named `data` (`100Mi`, `ReadWriteOnce`), mounted at `/usr/share/nginx/html`. Create the matching headless Service.
2. Write a unique file in each Pod's volume.
3. Delete `store-1` and wait for it to be recreated.
4. Confirm the file survived and the PVC name is the same.

**Completion criteria**
- PVCs `data-store-0`, `data-store-1`, `data-store-2` exist.
- The recreated Pod reattaches to its original PVC with data intact.
- Explain why deleting the StatefulSet does not delete its PVCs.

## T41. Projected Volume

ConfigMap `app-settings` (`LOG_LEVEL=debug`) and Secret `app-token` (`token=abc123`) exist.

1. Create Pod `projected-pod` (`busybox:1.36`, `sleep 3600`) with a single `projected` volume combining both sources, mounted at `/etc/app`.
2. Map them to files `settings/log_level` and `secrets/token`.
3. Verify both files inside the container.

**Completion criteria**
- `/etc/app/settings/log_level` contains `debug`.
- `/etc/app/secrets/token` contains `abc123`.
- Only one volume entry appears in the Pod spec.

---

# Suggested Study Order

1. **Highest value:** T1–T9 (troubleshooting).
2. **Next:** T12, T13, T20, T24, T26, T31, T32, T36, T37.
3. **Then:** everything else, ideally repeating tasks you could not finish in under 8 minutes.

# Final Verification Checklist

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide | grep -v Running
kubectl get events -A --sort-by=.metadata.creationTimestamp | tail -30
kubectl get sc,pv,pvc -A
kubectl get svc,ep -A
kubectl auth can-i --list --as=<user-or-sa>
```

### Exam habits

- Switch context first, every task.
- Use `kubectl explain <resource>.<field> --recursive` instead of searching docs for field names.
- Generate YAML with `--dry-run=client -o yaml`, then edit.
- For control-plane faults, work with `crictl` and `/etc/kubernetes/manifests/`; kubelet recreates static Pods automatically.
- Always verify the end state, not just the command output.
