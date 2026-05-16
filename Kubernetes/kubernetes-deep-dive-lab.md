# Kubernetes Deep-Dive Lab — Built Around Your `fcms` App

This is a hands-on lab that uses **your own running app** (the `fcms`
namespace: frontend, backend, db, with a PVC, ConfigMap, Secret, and
Ingress) to take you from "it works and I know some kubectl" to "I
understand why every piece behaves the way it does."

## How to use this guide

1. Do experiments **in order**. Each one builds a mental model the next
   one depends on.
2. Before every command that changes something, **write your prediction
   down** (a, b, c answers) on paper or in a notes file. This is not
   optional — being wrong on purpose and then understanding why is the
   entire learning mechanism. Recognition ("oh yeah that makes sense")
   is not understanding; prediction is.
3. After each experiment, answer the **"Explain it back"** questions in
   your own words. If you go vague, that's a gap — go read the linked
   docs and come back.
4. One experiment per sitting is fine. Depth beats speed.

Throughout, replace placeholders like `<backend-pod>` with real names
from `kubectl get pods -n fcms`.

A note on your starting state: your backend pods showed
`RESTARTS 2 (157m ago)` while frontend and db showed `0`. Keep that
mystery in mind — you'll be able to fully explain it by Experiment 7.

---

## Pre-flight: Back everything up

You will deliberately break things. Make this safe and reversible.

```bash
mkdir -p ~/k8s-backup
# Export every object in your namespace so you can always restore:
kubectl get all,configmap,secret,ingress,pvc -n fcms -o yaml > ~/k8s-backup/fcms-full.yaml

# If you have the original manifest files, copy them too:
# cp *.yaml ~/k8s-backup/
```

To restore anything later:

```bash
kubectl apply -f ~/k8s-backup/fcms-full.yaml
```

---

## Experiment 0 — Read what's already there (no changes)

The first skill is reading the system, not changing it.

```bash
kubectl get all -n fcms
kubectl describe deployment fcms-backend -n fcms
kubectl describe pod <backend-pod> -n fcms
kubectl get events -n fcms --sort-by=.metadata.creationTimestamp
```

**Explain it back:**

- In `describe deployment`, find the section that lists the
  ReplicaSet. Why does a Deployment have a ReplicaSet underneath it
  instead of managing Pods directly?
- In `describe pod`, find `Restart Count` and the `Last State` /
  `Reason`. What does it say for the backend pods? Form a hypothesis
  about why backend restarted but db didn't. Write it down — you'll
  confirm or kill this hypothesis in Experiment 7.
- In the events list, what's the earliest thing that happened? Read the
  story top to bottom: scheduled → pulled image → created → started.

**Concept:** Kubernetes is a set of controllers running reconciliation
loops. You declare desired state; controllers continuously observe
actual state and act to close the gap. Everything below is a
consequence of this one idea.
Docs: https://kubernetes.io/docs/concepts/architecture/controller/

---

## Experiment 1 — The reconciliation loop (delete a pod)

**Predict first (write it down):**
- (a) How many seconds until something replaces a deleted backend pod?
- (b) Which object creates the replacement — the Deployment, the
  ReplicaSet, the scheduler, or the kubelet?
- (c) Does your app go down during this? Why or why not?

**Run it.** Terminal 1 (leave running):

```bash
kubectl get pods -n fcms -w
```

Terminal 2:

```bash
kubectl delete pod <backend-pod> -n fcms
```

Watch Terminal 1: the pod goes `Terminating`, and a **new pod with a
different random suffix** appears and goes `Running`. Now find out who
did it:

```bash
kubectl describe replicaset <backend-replicaset> -n fcms
```

Read the `Events:` at the bottom — you'll see the ReplicaSet itself
created a pod because it observed 1 pod but its spec wants 2.

**Explain it back:**
- You never ran a "create pod" command. So who created it, and how did
  it know to? Trace: Deployment owns ReplicaSet, ReplicaSet owns Pods.
- Why does the new pod have a different name?
- Reconcile your prediction with reality. Where were you wrong, and
  what does the gap teach you?

Docs: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/

---

## Experiment 2 — Service discovery reacts in real time

A Service IP is stable; the pods behind it are disposable. This
experiment makes that visible.

**Predict:** When you kill a backend pod, what happens to the
`fcms-backend` Endpoints list, and for roughly how long is the dead
pod's IP still listed?

**Run it.** Terminal 1:

```bash
kubectl get endpoints fcms-backend -n fcms -w
# (newer clusters: kubectl get endpointslices -n fcms -w)
```

Note the current pod IPs. Terminal 2:

```bash
kubectl delete pod <backend-pod> -n fcms
```

Watch the IP set in Terminal 1: the dead pod's IP drops out, and the
new pod's IP appears **only once the new pod is Ready**.

**Explain it back:**
- Your Service `fcms-backend` has ClusterIP `10.105.61.238` and that
  never changed. The pod IPs behind it did. So what is a Service,
  really — a process, or a stable name plus a controller-maintained
  list of backends?
- Your frontend talks to `fcms-backend` and never noticed a pod died.
  Explain exactly why it didn't break.
- What is the role of pod **readiness** here? What would happen if a
  pod were added to Endpoints before it could actually serve traffic?

Docs: https://kubernetes.io/docs/concepts/services-networking/service/

---

## Experiment 3 — Prove DNS is how your app finds its parts

Your frontend reaches the backend by a *name*. Prove the mechanism end
to end.

```bash
kubectl exec -it <frontend-pod> -n fcms -- sh
```

Inside the pod:

```sh
cat /etc/resolv.conf
nslookup fcms-backend          # if nslookup missing, see fallback below
nslookup fcms-db
# Fallback if no nslookup/dig:
#   wget -qO- http://fcms-backend:5000   (proves reachability by name)
exit
```

`fcms-backend` resolves to `10.105.61.238` — the exact ClusterIP from
your `kubectl get all` output. Look at the `search` line in
`resolv.conf`: it contains something like
`fcms.svc.cluster.local svc.cluster.local cluster.local`. That's why
the short name `fcms-backend` works from inside the `fcms` namespace.

**Explain it back:**
- Write the full chain for one request, no hand-waving:
  name → ? → ClusterIP → ? → a live pod.
  (Answer: name → CoreDNS → Service ClusterIP → kube-proxy
  iptables/IPVS rules → one pod IP from the Endpoints list.)
- What is the fully-qualified name of your backend Service? Why does
  the short form only work inside the same namespace?
- Which component answers the DNS query? (Look in `kube-system`:
  `kubectl get pods -n kube-system | grep -i dns`)

Docs: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/

---

## Experiment 4 — Load balancing across replicas

You have 2 backend replicas. Are requests actually spread across them?

**Predict:** If you send 10 requests to the backend Service, do they
hit one pod or both? How would you prove it rather than assume it?

**Run it.** Easiest proof is to watch logs from both pods while you
generate traffic.

```bash
# Terminal 1 — stream logs from ALL backend pods at once:
kubectl logs -n fcms -l app=fcms-backend -f --prefix=true
# (if your label key isn't `app`, check:
#  kubectl get pods -n fcms --show-labels )
```

```bash
# Terminal 2 — hit the service repeatedly from inside the cluster:
kubectl run tmp-curl --rm -it --image=curlimages/curl -n fcms --restart=Never -- \
  sh -c 'for i in $(seq 1 10); do curl -s -o /dev/null -w "%{http_code}\n" http://fcms-backend:5000/; done'
```

Watch which pod prefixes show log lines in Terminal 1.

**Explain it back:**
- Did both pods serve traffic? What component is doing the spreading,
  and at what layer (DNS round-robin? virtual IP + kube-proxy rules)?
- If one backend pod were not Ready, would it still receive traffic?
  Tie this back to Endpoints from Experiment 2.

Docs: https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies

---

## Experiment 5 — Scaling and the desired-state model

**Predict:** After `--replicas=4`, how many pods, and who creates the
new ones? After `--replicas=0`, what happens to your app, and is the
Deployment "gone"?

```bash
kubectl scale deployment fcms-backend --replicas=4 -n fcms
kubectl get pods -n fcms -w        # watch them appear
kubectl get rs -n fcms             # ReplicaSet DESIRED/CURRENT/READY

kubectl scale deployment fcms-backend --replicas=0 -n fcms
kubectl get deployment fcms-backend -n fcms   # still exists, 0/0
# Try the app now — backend is unreachable but the Deployment object lives.

kubectl scale deployment fcms-backend --replicas=2 -n fcms   # restore
```

**Explain it back:**
- "Desired state" is a number you set; the controller makes reality
  match. Where is that number stored, and who acts on it?
- At `replicas=0` the Deployment still exists. What does that tell you
  about the difference between a *workload object* and the *pods* it
  manages?

---

## Experiment 6 — Rollouts, history, and rollback

This is how real deployments and real outages happen.

**Predict:** If you set the backend image to a tag that doesn't exist,
what happens to (a) the old pods, (b) the new pods, (c) live traffic?

```bash
kubectl rollout history deployment/fcms-backend -n fcms

# Break it on purpose with a bogus image tag:
kubectl set image deployment/fcms-backend fcms-backend=nginx:doesnotexist-999 -n fcms

kubectl get pods -n fcms -w
kubectl describe pod <new-failing-pod> -n fcms   # read the Events
kubectl rollout status deployment/fcms-backend -n fcms   # will hang/stall
```

You should see `ImagePullBackOff` / `ErrImagePull` on the *new* pod.
Crucially, check whether your app is still up — the old pods are kept
running until new ones are healthy.

```bash
# Undo:
kubectl rollout undo deployment/fcms-backend -n fcms
kubectl rollout status deployment/fcms-backend -n fcms   # now completes
```

**Explain it back:**
- Why did your app stay up even though the new version was broken?
  (Look up `RollingUpdate`, `maxSurge`, `maxUnavailable`.)
- What does a new ReplicaSet have to do with a rollout? Run
  `kubectl get rs -n fcms` and watch how many ReplicaSets exist for
  the backend now and which one has the pods.
- What exactly did `rollout undo` change back?

Docs: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment

---

## Experiment 7 — Probes, CrashLoopBackOff, and your restart mystery

Now solve the puzzle from the start: backend restarted twice, db did
not.

**Investigate first (no changes):**

```bash
kubectl describe pod <backend-pod> -n fcms
# Look at: Last State, Reason, Exit Code, Restart Count, and whether
# livenessProbe / readinessProbe are configured.
kubectl get deployment fcms-backend -n fcms -o yaml | grep -A15 -i probe
```

The likely story: at startup the backend came up *before* the database
was accepting connections, the process exited, and Kubernetes
restarted the container (backoff) until the DB was ready. The db pod
had nothing to wait on, so it never crashed. Kubernetes does **not**
guarantee start order between pods — apps must tolerate dependencies
not being ready yet.

**Now make it visible deliberately.** Add (or tighten) a liveness probe
that you know will fail, observe `CrashLoopBackOff`, then revert.

```bash
# Inspect what a failing probe does to RESTARTS, then revert with:
kubectl rollout undo deployment/fcms-backend -n fcms
# (or re-apply your backup)
```

**Explain it back:**
- Difference between a **liveness** probe (restart the container) and a
  **readiness** probe (remove from Service Endpoints but don't kill).
  Tie readiness back to Experiments 2 and 4.
- Explain your original `RESTARTS 2 (157m ago)` line in one or two
  sentences using the words *start order*, *exit code*, and *backoff*.
- What's the correct production-grade fix for "backend needs DB
  first" — an init container, a readiness probe, ret/reconnect logic
  in the app, or some combination? Argue it.

Docs: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

---

## Experiment 8 — Resource limits and OOMKills

**Predict:** If you give the backend a memory *limit* far below what it
needs, what status/reason will the pod show, and what will the restart
count do?

```bash
# See current requests/limits (may be empty — that itself is a finding):
kubectl get deployment fcms-backend -n fcms -o yaml | grep -A8 resources

# Set an absurdly low memory limit on purpose:
kubectl set resources deployment/fcms-backend -n fcms \
  --limits=memory=16Mi --requests=memory=8Mi

kubectl get pods -n fcms -w
kubectl describe pod <backend-pod> -n fcms   # look for OOMKilled
```

You should see the container killed with reason **OOMKilled** and the
restart count climbing. Revert:

```bash
kubectl rollout undo deployment/fcms-backend -n fcms
```

**Explain it back:**
- Difference between a resource **request** (scheduling guarantee) and
  a **limit** (hard ceiling). Which one caused the kill?
- What are QoS classes (Guaranteed / Burstable / BestEffort) and which
  one is your backend in? (`kubectl describe pod` shows `QoS Class`.)
- Why is "no limits set at all" also a risky production state?

Docs: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

---

## Experiment 9 — Storage: the db-as-Deployment footgun

Your db is a **Deployment** with **one PVC**. This experiment teaches
why that's a known trap.

### Part A — PVC outlives the pod (data survives)

**Predict:** Create a record in your app. If you delete the db pod,
does the data survive when the new pod comes up? Why?

```bash
kubectl get pvc -n fcms
kubectl describe pvc <pvc-name> -n fcms   # note STATUS, ACCESS MODES, STORAGECLASS

# Create a record in your app UI, then:
kubectl delete pod <db-pod> -n fcms
kubectl get pods -n fcms -w
# Reload your app — the record should still be there.
```

It survives because the **PVC's lifecycle is decoupled from the pod's**.
The new pod re-mounts the same volume.

### Part B — Two db pods, one volume (the footgun)

**Predict carefully before running:** You scale the db Deployment to 2.
There is one PVC. What happens to the second pod? Two possibilities —
reason about which: (i) it can't mount and stays Pending/ContainerCreating,
or (ii) it mounts the same files and you now have two database engines
fighting over one data directory (corruption risk).

```bash
kubectl scale deployment fcms-db --replicas=2 -n fcms
kubectl get pods -n fcms
kubectl describe pod <new-db-pod> -n fcms   # read Events / volume mount

# ALWAYS scale back:
kubectl scale deployment fcms-db --replicas=1 -n fcms
```

**Explain it back:**
- What is the access mode of your PVC (`RWO` vs `RWX`) and how does
  that determine what happened in Part B?
- Why do stateful workloads like databases use a **StatefulSet** with
  `volumeClaimTemplates` (one PVC *per* pod, stable identity) instead
  of a Deployment with a shared PVC?
- What does your PVC's **StorageClass** and **reclaim policy** mean for
  what happens to the data if the PVC itself is deleted?

Docs:
https://kubernetes.io/docs/concepts/storage/persistent-volumes/ and
https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

---

## Experiment 10 — ConfigMap and Secret: where config actually lives

**Predict:** If you change a value in the ConfigMap, does a running pod
see the new value immediately, on its own, or only after a restart?

```bash
kubectl get configmap -n fcms
kubectl describe configmap <cm-name> -n fcms
kubectl get secret -n fcms
kubectl get secret <secret-name> -n fcms -o jsonpath='{.data}' ; echo
# Secret values are base64, not encrypted. Decode one key to prove it:
kubectl get secret <secret-name> -n fcms \
  -o jsonpath='{.data.<KEY>}' | base64 -d ; echo

# See how the deployment consumes them (envFrom / valueFrom / volume):
kubectl get deployment fcms-backend -n fcms -o yaml | grep -A12 -iE 'env|configMap|secret|volume'
```

Change a ConfigMap value, then check a running pod's env (it will
**not** have changed if injected as env vars — env is set at container
start only):

```bash
kubectl exec -n fcms <backend-pod> -- env | grep <YOUR_KEY>
kubectl rollout restart deployment/fcms-backend -n fcms   # picks up new value
```

**Explain it back:**
- Are Secrets encrypted by default? What did the base64 decode prove?
- Why didn't the env var change in the running pod when you edited the
  ConfigMap? What's the difference between env-injected config and
  config mounted as a volume (which *can* update live)?
- What does `rollout restart` actually do, and why did that make the
  new value take effect?

Docs:
https://kubernetes.io/docs/concepts/configuration/configmap/ and
https://kubernetes.io/docs/concepts/configuration/secret/

---

## Experiment 11 — Ingress: how traffic gets in from outside

You have an Ingress and the `ingress-nginx` namespace. This is the
front door.

```bash
kubectl get ingress -n fcms
kubectl describe ingress <ingress-name> -n fcms
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Trace the full external path and write it out:

```
your browser
  → minikube node IP / `minikube tunnel`
  → ingress-nginx controller Service
  → ingress-nginx controller pod (reads Ingress rules)
  → matches host/path rule → fcms-frontend Service (ClusterIP)
  → kube-proxy rules → a frontend pod
  → (frontend calls fcms-backend Service → a backend pod
       → fcms-db Service → db pod → PVC)
```

**Explain it back:**
- An Ingress *object* is just rules. What actually enforces them?
  (The ingress-nginx **controller** — without a controller, an Ingress
  object does nothing.)
- In your Ingress rules, which host/paths map to which Services? Read
  them off `describe ingress`.
- Compare Ingress vs a `NodePort`/`LoadBalancer` Service. Why use an
  Ingress for an HTTP app with multiple services?

Docs: https://kubernetes.io/docs/concepts/services-networking/ingress/

---

## Experiment 12 — Break the wiring, watch the failure shape

For each: predict the *exact* symptom, break it, observe, then heal.

```bash
# A) Delete the backend Service. How does the app fail — slow, instant,
#    which page? Then recreate from backup.
kubectl delete svc fcms-backend -n fcms
#   ...test app, observe, then:
kubectl apply -f ~/k8s-backup/fcms-full.yaml

# B) Point the frontend at a wrong backend service name (edit env /
#    ConfigMap) and observe the failure mode vs case A. Revert.

# C) Delete the PVC's backing pod AND cordon nothing — already done in
#    Exp 9; here instead delete a frontend pod during active use and
#    watch zero downtime (2 replicas) vs deleting the db pod (brief
#    blip, single replica).
```

**Explain it back:**
- "Service deleted" vs "wrong service name" produce *different* failure
  shapes (DNS NXDOMAIN vs connection refused vs timeout). Describe each
  and what that tells you about where in the chain the break is. This
  is the core production debugging skill.

---

## Experiment 13 — RBAC: who is allowed to do what

Every pod runs as a ServiceAccount with permissions.

```bash
kubectl get serviceaccount -n fcms
kubectl get pod <backend-pod> -n fcms -o jsonpath='{.spec.serviceAccountName}' ; echo

# Can the default SA in fcms list pods? Ask the API server directly:
kubectl auth can-i list pods -n fcms \
  --as=system:serviceaccount:fcms:default
kubectl auth can-i delete deployments -n fcms \
  --as=system:serviceaccount:fcms:default
```

**Explain it back:**
- What's the difference between a Role and a ClusterRole, and a
  RoleBinding vs ClusterRoleBinding?
- Your app pods almost certainly don't need Kubernetes API access at
  all. Why is a tightly-scoped (or no-permission) ServiceAccount a
  security best practice?

Docs: https://kubernetes.io/docs/reference/access-authn-authz/rbac/

---

## Experiment 14 — The control plane itself

Finally, look under the floor.

```bash
kubectl get pods -n kube-system
# Identify: kube-apiserver, etcd, kube-scheduler,
# kube-controller-manager, coredns, kube-proxy, the CNI pods.

kubectl get componentstatuses              # (may be deprecated; try anyway)
kubectl describe pod -n kube-system <kube-scheduler-pod>
```

**Explain it back, in your own words, one sentence each:**
- **kube-apiserver:** the only thing anything talks to; the front door
  to cluster state.
- **etcd:** the database of record — all desired/actual state lives
  here.
- **kube-scheduler:** decides *which node* a new pod runs on.
- **kube-controller-manager:** runs the reconciliation loops
  (Deployment, ReplicaSet, endpoints, etc.) you watched all lab.
- **kubelet:** node agent that actually starts/stops containers and
  reports status.
- **kube-proxy:** programs the node networking rules that make Service
  ClusterIPs work (Experiments 2–4).
- **CoreDNS:** answers the name lookups from Experiment 3.

Now retell **one CRUD request in your app** as a single narrative that
touches: Ingress → Service → kube-proxy → pod → DNS → backend Service →
backend pod → DB Service → DB pod → PVC, *and* explain how the system
would self-heal if any one pod in that path died. If you can write that
clearly with no hand-waving, you understand your app's Kubernetes setup
at depth.

---

## Capstone — Write it up

Create a file `HOW-MY-APP-RUNS-ON-K8S.md` and write, from memory:

1. The full request path for one create operation.
2. What each object in `fcms` does and why it exists.
3. How the pieces find each other (labels, selectors, Services, DNS).
4. How the system self-heals (which controller does what).
5. The two known weak spots in this setup (db-as-Deployment with a
   shared PVC; backend startup ordering vs the DB) and how you'd fix
   each properly.

Anywhere your writing goes vague is exactly the spot to revisit. That
document is the proof you crossed from "it works" to "I understand it."

---

## Quick command reference

```bash
# Observe
kubectl get all -n fcms
kubectl get pods -n fcms -o wide --show-labels
kubectl describe pod <pod> -n fcms
kubectl get events -n fcms --sort-by=.metadata.creationTimestamp
kubectl logs -n fcms -l app=fcms-backend -f --prefix=true

# Inspect wiring
kubectl get endpoints -n fcms
kubectl get deployment <d> -n fcms -o yaml
kubectl exec -it <pod> -n fcms -- sh

# Change desired state
kubectl scale deployment <d> --replicas=N -n fcms
kubectl set image deployment/<d> <container>=<image> -n fcms
kubectl set resources deployment/<d> --limits=memory=… -n fcms
kubectl rollout status deployment/<d> -n fcms
kubectl rollout history deployment/<d> -n fcms
kubectl rollout undo deployment/<d> -n fcms
kubectl rollout restart deployment/<d> -n fcms

# Safety net
kubectl get all,configmap,secret,ingress,pvc -n fcms -o yaml > ~/k8s-backup/fcms-full.yaml
kubectl apply -f ~/k8s-backup/fcms-full.yaml
```
