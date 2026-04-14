# Lab 9 — ArgoCD App of Apps with Sealed Secrets and ingress-nginx

Everything in this lab is installed by ArgoCD from git. A single root
Application (`myapp-root`) bootstraps the whole stack:

```
myapp-root  (app-of-apps)
├── sealed-secrets        [sync-wave -10]  Helm, bitnami-labs
├── ingress-nginx         [sync-wave -10]  Helm, kubernetes.github.io
├── myapp-secrets         [sync-wave  -5]  directory, this repo
│     ├── postgres-credentials  (SealedSecret → myapp namespace)
│     └── postgres-credentials  (SealedSecret → production namespace)
├── myapp-helm            [sync-wave   0]  Helm, devops2-lab8
└── myapp-kustomize       [sync-wave   0]  Kustomize, devops2-lab8
```

Sync waves guarantee the controllers and SealedSecrets land before the
workloads that depend on them, so the Postgres Deployments never start
without their `postgres-credentials` Secret.

## Layout

| Path                                        | What it is                                      |
|---------------------------------------------|--------------------------------------------------|
| `argocd/root-app.yaml`                      | App of Apps root — apply this one file to bootstrap everything |
| `argocd/00-sealed-secrets.yaml`             | ArgoCD Application installing the Sealed Secrets controller via Helm |
| `argocd/01-ingress-nginx.yaml`              | ArgoCD Application installing ingress-nginx via Helm |
| `argocd/02-secrets.yaml`                    | ArgoCD Application that delivers the SealedSecret manifests in `secrets/` |
| `argocd/myapp-helm.yaml`                    | ArgoCD Application for the lab8 Helm umbrella chart |
| `argocd/myapp-kustomize.yaml`               | ArgoCD Application for the lab8 Kustomize production overlay |
| `secrets/postgres-myapp.sealedsecret.yaml`  | Encrypted Postgres credentials for the `myapp` namespace |
| `secrets/postgres-production.sealedsecret.yaml` | Encrypted Postgres credentials for the `production` namespace |

## Prerequisites

- A Kubernetes cluster (kind / minikube / k3d / real cluster).
- `kubectl` and the `kubeseal` CLI:
  ```sh
  brew install kubeseal              # macOS
  # or download from github.com/bitnami-labs/sealed-secrets/releases
  ```
- ArgoCD installed into the `argocd` namespace:
  ```sh
  kubectl create namespace argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  kubectl -n argocd rollout status deploy/argocd-server
  ```

## Before you bootstrap: seal real credentials

The two files in `secrets/` ship with `AgA_REPLACE_ME_*` placeholder
ciphertext. They will fail to decrypt. Replace them with real encrypted
values first.

1. Bootstrap just the Sealed Secrets controller so `kubeseal` has a key to
   encrypt against. Easiest path: apply the controller Application by hand:
   ```sh
   kubectl apply -n argocd -f lab9/argocd/00-sealed-secrets.yaml
   kubectl -n kube-system rollout status deploy/sealed-secrets-controller
   ```

2. Produce the two SealedSecret manifests:
   ```sh
   # myapp namespace
   kubectl -n myapp create secret generic postgres-credentials \
     --from-literal=POSTGRES_USER=admin \
     --from-literal=POSTGRES_PASSWORD='change-me' \
     --from-literal=POSTGRES_DB=appdb \
     --dry-run=client -o yaml |
   kubeseal --controller-namespace kube-system \
            --controller-name sealed-secrets-controller \
            --format yaml \
            > lab9/secrets/postgres-myapp.sealedsecret.yaml

   # production namespace
   kubectl -n production create secret generic postgres-credentials \
     --from-literal=POSTGRES_USER=admin \
     --from-literal=POSTGRES_PASSWORD='change-me-too' \
     --from-literal=POSTGRES_DB=appdb \
     --dry-run=client -o yaml |
   kubeseal --controller-namespace kube-system \
            --controller-name sealed-secrets-controller \
            --format yaml \
            > lab9/secrets/postgres-production.sealedsecret.yaml
   ```

3. Commit and push the regenerated files. The ciphertext is safe to store
   in git — only the controller in this specific cluster can decrypt it.

## Bootstrap the whole stack

Push this `lab9` directory to a git remote (the `repoURL` fields currently
assume `github.com/mkalybek/devops2-lab9`, branch `main` — tweak to match
your actual remote). Then apply the root:

```sh
kubectl apply -n argocd -f lab9/argocd/root-app.yaml
```

ArgoCD will create `sealed-secrets`, `ingress-nginx`, `myapp-secrets`,
`myapp-helm`, and `myapp-kustomize` as child Applications and reconcile
them in sync-wave order.

Watch it land:

```sh
kubectl -n argocd get applications
kubectl -n kube-system get pods -l app.kubernetes.io/name=sealed-secrets
kubectl -n ingress-nginx get pods
kubectl -n myapp get pods,svc,ingress,secret
kubectl -n production get pods,svc,ingress,secret
```

You should see a real `postgres-credentials` Secret in both `myapp` and
`production` — created by the controller, never by git — and the Postgres
Deployments picking it up via `envFrom`.

## How the secrets flow into the workloads

- **Helm (lab8 `helm/myapp`)**: the db subchart now reads
  `.Values.db.auth.existingSecret` (default: `postgres-credentials`) and
  switches the container to `envFrom: secretRef` when it is set. The
  plaintext `username`/`password`/`database` values are only used if you
  explicitly unset `existingSecret`.
- **Kustomize (lab8 `kustomize/base`)**: the db Deployment uses
  `envFrom: secretRef: postgres-credentials` directly. No plaintext creds
  left in the manifest.

Both paths depend on the SealedSecret for their namespace being applied
first — which is exactly what sync-wave `-5` on `myapp-secrets` guarantees.

## Tearing it all down

```sh
kubectl -n argocd delete application myapp-root
```

The `resources-finalizer.argocd.argoproj.io` finalizer on each child
Application cascades the delete down to the managed workloads, Secrets,
and the sealed-secrets / ingress-nginx controllers.
