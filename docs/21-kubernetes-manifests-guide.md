# Writing the Kubernetes manifests yourself

A guide, not an answer key. It tells you **what** the two services need from
Kubernetes and **why**, where the traps are, and how to prove each step
works. The YAML is yours to write.

Everything in the "contract" and "traps" sections was verified: both images
were run under the restrictions below, and deployed into minikube, before
this guide was written. The manifests used for that were deleted on purpose.

The team's target cluster is **K3s on Elad's server**, per the
[Phase 3 plan](20-phase-3-plan.md) (item 2: hand-written manifests first,
Helm later). The guide notes where K3s differs from minikube — see §10.

---

## 1. What you are building

Two independent services in one namespace:

```
  your browser
      │  kubectl port-forward
      ▼
  review-console  ──/api/──▶  review-api  ──▶  the model (LM Studio)
  (nginx, :8080)              (Flask, :8000)    outside the cluster
                                   │
                                   └──▶  GitHub (clones repositories)
```

- **review-api** is the product. The final stage deploys **only this**.
- **review-console** is a debugging add-on. You apply it to check, from
  inside the cluster, that the API is up and can reach the model — then you
  can delete it again without touching the API.

That split should be visible in how you organise your files: deploying the
API must not require the console, and removing the console must not touch
the API. Kustomize is a good fit (see §8), but plain `kubectl apply -f` on
two folders works too.

---

## 2. Before you write anything: get the images into the cluster

Kubernetes pulls images; it does not see your local Docker. Two options:

| Option | How | Good for |
|---|---|---|
| Load into minikube | build with `backend/build.sh` and `frontend/build.sh`, then `minikube image load <image>:<tag>` | local practice |
| Push to a registry | `build.sh --registry docker.io/<dockerhub-user> --tag <commit-sha> --push` (after `docker login`) | K3s, anything shared, CI/CD; use a Docker Hub namespace, or use the AWS ECR image URI for EKS |

Check it landed: `minikube image ls | grep review-`.

> **Trap — the `:latest` tag.** When an image tag is `latest` (or missing),
> Kubernetes defaults `imagePullPolicy` to `Always`, so it tries to pull from
> Docker Hub even though the image is already in minikube — and
> `review-api:latest` on Docker Hub, if it exists, is **somebody else's
> image**. Either use a real tag or set the pull policy yourself. Know
> which one you chose and why. The plan's answer is to tag with the **commit
> SHA** (`--tag "$(git rev-parse --short HEAD)"`): a tag that moves makes a
> rollback impossible to describe and a running Pod impossible to identify.

---

## 3. The contract each image expects

These are facts about the images. Your manifests must fit around them.

### review-api

| Property | Value |
|---|---|
| Image | `review-api:<tag>` (built from `backend/`) |
| Listens on | `8000/TCP` |
| Runs as | uid **10001**, gid **10001** — numeric, non-root |
| Needs writable | `/reports` (keep it: reports and crash checkpoints) and `/tmp` (clones, deleted after each review) |
| Root filesystem | can be read-only once those two are writable |
| Liveness | `GET /healthz` — no API key needed, checks nothing external |
| Readiness | `GET /readyz` — `503` if `/reports` or the temp directory is not writable |
| Model reachability | `GET /api/v1/health?probe=1` — **not** a probe (see §4.6) |
| Settings | environment variables — see `backend/README.md` |
| Secrets it can use | `GOOGLE_API_KEY`, `GITHUB_TOKEN`, `REVIEW_API_KEY`, … |
| Must reach | the model's base URL, and `github.com:443` |
| Replicas | **exactly one** — see §4.2 |

### review-console

| Property | Value |
|---|---|
| Image | `review-console:<tag>` (built from `frontend/`) |
| Listens on | `8080/TCP` |
| Runs as | uid **101**, gid **101** (the nginx user), non-root |
| Needs writable | `/etc/nginx/conf.d` (config is rendered there at start) and `/tmp` (pid and temp files) |
| Root filesystem | can be read-only once those two are writable |
| Liveness / readiness | `GET /healthz` — the console's own; it does not depend on the API |
| Setting | `REVIEW_API_URL` — `scheme://host:port`, **no path**. Default `http://review-api:8000` |
| Must reach | the review-api Service |

If `REVIEW_API_URL` has a path in it the container refuses to start and
says why in its log. That is deliberate: with a path, nginx would strip
`/api/` from every request and the console would 404 with no explanation.

---

## 4. review-api, step by step

### 4.1 Namespace and Pod Security

The architecture doc (`docs/03-high-level-design/12-kubernetes-architecture.md`,
12.1) says the namespace enforces the **restricted** Pod Security profile.
That is three labels on the Namespace object — look up *Pod Security
Admission* and the `pod-security.kubernetes.io/enforce` label.

Why bother: with `enforce: restricted`, a pod that breaks the rules is
**rejected at admission** with a message listing every violation. That turns
the namespace into a test for everything in §4.2.

**Prove it works** before deploying anything else: try to start a plain pod
in that namespace (`kubectl run test --image=nginx -n <ns>`). It should be
refused, and the error tells you exactly which fields the restricted profile
requires. Read it — it is your checklist for the next step.

### 4.2 The Deployment

Things to decide, and the reasoning behind each:

- **Replicas: 1.** The API keeps its job list in memory and runs one review
  at a time against one model. A second replica would accept a second review
  the first knows nothing about, and answer status requests for jobs it never
  started with 404.
- **Update strategy.** The reports volume in §4.4 can only be mounted by one
  node at a time. With the default rolling update, the new pod waits for a
  volume the old pod still holds. Look at the two strategies a Deployment
  supports and pick the one that stops the old pod first.
- **Security context** — pod level and container level. The restricted
  profile will tell you what is mandatory; the contract in §3 tells you the
  values. Things to look up: `runAsNonRoot`, `runAsUser`, `runAsGroup`,
  `fsGroup`, `seccompProfile`, `allowPrivilegeEscalation`,
  `readOnlyRootFilesystem`, `capabilities`.
  *Think:* which of these makes `/reports` writable by uid 10001?
- **Service account token.** The API never talks to the Kubernetes API, so
  it should not have a token mounted. Find the field that turns that off.
- **Resources.** Set requests and limits. A review loads the whole
  repository's source into memory; 1 GiB is a reasonable ceiling to start
  from — then watch it with `kubectl top pod` (metrics-server is enabled in
  your minikube).

### 4.3 Configuration and secrets

- Plain settings (which provider, which model, the model's URL) belong in a
  **ConfigMap**. There is a way to turn every key of a ConfigMap into
  environment variables in one go — find it, it saves listing each variable.
- Keys and tokens belong in a **Secret**, and **never in a file you commit**.
  Create it from the command line instead:
  `kubectl create secret generic <name> --from-literal=GOOGLE_API_KEY=...`
  (and consider how that command ends up in your shell history).
- Make the Secret reference **optional**, so the API still deploys on a
  cluster where nobody has created the Secret yet. Look for the `optional`
  field.

### 4.4 Storage

- `/reports` needs a **PersistentVolumeClaim**. Reports and checkpoints must
  survive a pod restart — surviving a crash is the whole point of a
  checkpoint. Your minikube has a default StorageClass (`standard`), so a
  claim with just a size and an access mode will bind.
- `/tmp` needs an **emptyDir**. It holds clones that are deleted after every
  review, so it must not persist — but it has to be writable, because the
  root filesystem is not. Consider a size limit.

**Prove it:** after deploying, `kubectl exec` into the pod and write a file
to `/reports` and to `/` — the first should work, the second should fail.

### 4.5 The Service

- Type **ClusterIP**. The API clones repositories and accepts tokens; do not
  expose it outside the cluster by accident.
- **Name it `review-api`.** The console defaults to that name, and the
  gateway will look for it too.

> **Trap — service links. This one crashed the first deployment.**
> For every Service in a namespace, Kubernetes injects environment variables
> into every pod: a Service named `review-api` produces
> `REVIEW_API_PORT=tcp://10.103.82.121:8000`. `REVIEW_API_PORT` is also the
> variable this API reads for its own port number, so the pod crash-looped
> on `int("tcp://10.103.82.121:8000")`.
>
> The API now ignores values in that form (with a warning in its log), so it
> will not crash again. But you should still stop Kubernetes injecting them:
> there is a pod-spec field for exactly this — search for *enableServiceLinks*.
> The same trap waits for any application whose settings end in `_PORT`.
>
> **See it for yourself:** `kubectl exec deploy/review-api -- printenv | grep REVIEW_API`
> before and after you set that field.

### 4.6 Probes

| Probe | Path | Why this one |
|---|---|---|
| startup | `/healthz` | gives the process time to start before liveness judges it |
| liveness | `/healthz` | "is the process alive?" — nothing more |
| readiness | `/readyz` | "can it do its job?" — fails if `/reports` or `/tmp` is not writable |

**Do not** use `/api/v1/health?probe=1` for any of them. It calls the
model. If the model is down, a liveness probe on it makes Kubernetes restart
a perfectly healthy API in a loop — and every restart wipes the in-memory
job list. It also returns `401` once `REVIEW_API_KEY` is set, because a
kubelet sends no key.

### 4.7 Reaching the model from a pod

`localhost` inside a pod is the pod. The model is somewhere else:

| Where the model runs | URL the pod should use |
|---|---|
| LM Studio on the machine running minikube | `http://host.minikube.internal:1234/v1` |
| LM Studio on the K3s server itself | the server's own address — `host.minikube.internal` does **not** exist on K3s (§10) |
| LM Studio on a teammate's machine, nodes on the tailnet | `http://100.x.y.z:1234/v1` |
| Google AI Studio | the provider's default URL — needs `GOOGLE_API_KEY` in the Secret |

This is `LMSTUDIO_BASE_URL` in your ConfigMap. LM Studio must listen on all
interfaces, not only `127.0.0.1` (check `netstat -ano | findstr :1234`).

---

## 5. review-console, step by step

A smaller version of §4, with its own differences:

- **Security context:** uid/gid 101 (§3). No volume to own, so think about
  whether `fsGroup` is needed at all.
- **Writable paths:** two emptyDirs — `/etc/nginx/conf.d` and `/tmp`. Leave
  out the first and the container exits at start, unable to render its
  config; leave out the second and nginx cannot create its temp directories.
  Both messages are in §7.
- **`REVIEW_API_URL`:** `http://review-api:8000` — the Service from §4.5.
  Using the default? Then you may not need to set it at all. Decide.
- **Service:** ClusterIP on 8080.
- **No Ingress.** The console has no authentication and proxies straight to
  the API. Reach it with a port-forward:

  ```bash
  kubectl -n <ns> port-forward svc/review-console 8090:8080
  ```

  then open `http://localhost:8090`.
- **Start order.** nginx resolves `review-api` when it starts. If the
  review-api **Service** does not exist yet, the console crash-loops with
  `host not found in upstream`. The Service object is enough — the API pod
  does not have to be ready.

---

## 6. How you know it works

Run these in order. Each has an expected result.

| # | Command | Expect |
|---|---|---|
| 1 | `kubectl -n <ns> get pods` | both `Running`, `READY 1/1`, restarts `0` |
| 2 | `kubectl -n <ns> exec deploy/review-api -- id` | `uid=10001` |
| 3 | `kubectl -n <ns> exec deploy/review-api -- printenv REVIEW_API_PORT` | nothing, once service links are off |
| 4 | port-forward the console, open `http://localhost:8090` | green dot; header reads **"API via console → http://review-api:8000 · lmstudio reachable (N models)"** |
| 5 | `curl http://localhost:8090/api/v1/health?probe=1` | `"reachable": true` — the model is reachable **from inside the cluster** |
| 6 | start a `quick` review from the console | completes; the report appears under *Saved reports* |
| 7 | `kubectl -n <ns> delete pod -l <api selector>` | new pod comes up; the report from #6 is still listed |
| 8 | `kubectl run bad --image=nginx -n <ns>` | **rejected** by Pod Security |
| 9 | remove the console (delete its folder's objects) | API unaffected: `kubectl -n <ns> get deploy review-api` still ready |

Step 5 is the connection test this whole console exists for.

Step 7 is the one that proves the storage is right — if the report is gone,
`/reports` was not on the claim.

---

## 7. When it goes wrong

Real failures, what they look like, and where to look.

| Symptom | Likely cause | Look at |
|---|---|---|
| `CrashLoopBackOff`, log ends in `int('tcp://…')` | service-link variable (§4.5), on an image older than the fix | `kubectl logs --previous` |
| `ErrImagePull` / `ImagePullBackOff` | image not in the cluster, or `:latest` forcing a pull (§2) | `kubectl describe pod` → Events |
| Pod rejected: `violates PodSecurity "restricted:latest"` | a required security field missing | the rejection message lists them all |
| review-api `Running` but never `Ready`, `/readyz` says `reports directory is not writable` | the volume is not writable by uid 10001 | `fsGroup`; the volume mount path |
| review-api never `Ready`, `/readyz` says `no writable temp directory for clones` | read-only root and no volume on `/tmp`. Without the readiness check every review fails with `No usable temporary directory found in ['/tmp', …]` | `volumeMounts` |
| console exits at start: `can't create /etc/nginx/conf.d/default.conf: Read-only file system` | no emptyDir on `/etc/nginx/conf.d` | `kubectl logs --previous` |
| console exits at start: `mkdir() "/tmp/proxy_temp" failed (30: Read-only file system)` | no emptyDir on `/tmp` | `kubectl logs --previous` |
| console exits: `[emerg] host not found in upstream "review-api"` | Service missing, wrong name, or in another namespace | `kubectl get svc` |
| console loads, header says `cannot reach API` | proxy upstream wrong | open `/console.json` — it shows the upstream in use |
| header says `lmstudio unreachable` | pod cannot reach the model URL | §4.7; `kubectl exec … -- python -c "import urllib.request,sys; print(urllib.request.urlopen(sys.argv[1]).status)" <url>` |
| new pod stuck `ContainerCreating` during an update | rolling update + a volume only one pod may mount | §4.2 strategy |
| PVC stays `Pending` | no default StorageClass | `kubectl get storageclass` |
| probe failures after setting `REVIEW_API_KEY` | a probe pointed at `/api/v1/...` | §4.6 |

---

## 8. Organising the files

A layout that matches "each microservice in its own folder":

```
backend/k8s/        the API — namespace, config, storage, workload, service
frontend/k8s/       the console — workload, service
```

Kustomize (`kubectl apply -k <folder>`) lets each folder deploy on its own,
and a third kustomization can list both folders to deploy them together.
Things worth learning while you do it:

- the `namespace:` field, so the files themselves need not repeat it;
- the `images:` field, to point at a registry and tag without editing YAML;
- `kubectl kustomize <folder>` to see exactly what will be applied.

The `.dockerignore` in both folders already excludes `k8s/`, so manifests
there never end up inside an image.

---

## 9. Check yourself

Answer these before calling it done:

1. Why is replicas 1 correct here, when it is usually a smell?
2. What would break if liveness pointed at `/api/v1/health?probe=1`?
3. Which field makes a volume writable by a non-root user, and at which
   level of the spec does it go?
4. Why does the console need a writable `/etc/nginx/conf.d` but not a
   writable `/usr/share/nginx/html`?
5. What exactly did Kubernetes inject that crashed the first deployment,
   and which pod-spec field stops it?
6. Why is the console not behind an Ingress?
7. How would you point both images at a registry without editing the
   Deployment files?

---

## 10. On K3s instead of minikube

K3s is conformant Kubernetes: every object in this guide behaves the same.
What surrounds it differs, and these are the places it shows:

| | minikube | K3s |
|---|---|---|
| getting an image in | `minikube image load` | a registry (GHCR), or `k3s ctr images import <file.tar>` after `docker save` |
| default StorageClass | `standard` | `local-path` |
| reaching the host | `host.minikube.internal` | not provided — use the host's real IP, or its tailnet address |
| Pod Security Admission | built in | built in |

Two consequences worth thinking through before you write anything:

- **`local-path` volumes are node-local and ReadWriteOnce.** On one node a
  shared volume works by accident; on more it does not. The plan says to
  write PVCs as if the cluster had several nodes — which is one more reason
  for the single replica and the update strategy in §4.2.
- **Pulling from a private GHCR package needs credentials in the cluster.**
  Look up `imagePullSecrets`, and create that Secret from the command line,
  like the one in §4.3 — never from a committed file. The plan's item on
  secrets (Sealed Secrets) is the longer-term answer.

---

## References

- Pod Security Admission — kubernetes.io/docs/concepts/security/pod-security-admission/
- Pod Security Standards (restricted) — kubernetes.io/docs/concepts/security/pod-security-standards/
- Configure a Security Context — kubernetes.io/docs/tasks/configure-pod-container/security-context/
- Liveness, Readiness and Startup Probes — kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Environment variables for Services — kubernetes.io/docs/concepts/services-networking/service/#environment-variables
- Persistent Volumes — kubernetes.io/docs/concepts/storage/persistent-volumes/
- Kustomize — kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
- minikube: pushing images — minikube.sigs.k8s.io/docs/handbook/pushing/
- K3s: private registries and air-gapped images — docs.k3s.io/installation/private-registry
- K3s: local-path storage — docs.k3s.io/storage
- Pulling from a private registry — kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
