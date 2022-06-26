## FluxCD - GitOps

Read the documentation for FluxCD and how to use it [using this URL](https://fluxcd.io/docs/get-started/). In this project everything is meant to be running on ARM64/AMD64.


The repository follows the following structure, some manifests have been omitted for simplicity:
```
├── .github                         # Workflows 
├── alerts                          # Alerts set up for GitOps deployed resources status
├── applications                    # Folder to manage Helm Releases
│   ├── base                        # Folder to hold Helm Releases with default values
│   │   ├── simple-backend-service  # Python microservice default values for release
│   │   └── simple-frontend-service # ReactJS microservice default values for release
│   └── dev                         # Folder with Kustomization to manage DEV values overrides
├── automation                
├── clusters/my-cluster
│   ├── flux-system                 # Flux system CRDs
│   ├── infrastructure.yaml         # Kustomization that enables/disables all infrastructure tools
│   ├── apps.yaml                   # Kustomization that enables/disables all application releases
│   └── alerts.yaml                 # Kustomization that enables/disables all Flux alerts
├── infrastructure
│   ├── chart-museum-registry       # ChartMuseum Helm Registry
│   ├── github-runners              # Github Runners
│   ├── secrets                     # Directory for YAML manifest with encryption
│   ├── sources                     # Sources for Helm Registries
│   └── kustomization.yaml          # Kustomization to manage the infrastructure/ directory 
└── README.md
```

## GitOps Workflow - Mozilla SOPS
```bash
# Import secret key to encode/decode secrets
$ gpg --import private_key.pgp

# Export public/private key
$ gpg --output private_key.pgp --armor --export-secret-key email@email.com

# Encode/decode YAML manifest, working when the PGP key exists on the machine
$ sops --encrypt --in-place manifest.yaml
$ sops --decrypt manifest.yaml
```

## Bootstraping the cluster

```bash

# The bootstrap command does following:
#  - Creates a git repository fleet-infra on your GitHub account
#  - Adds Flux component manifests to the repository
#  - Deploys Flux Components to your Kubernetes Cluster
#  - Configures Flux components to track the path /clusters/my-cluster/ in the repository

$ export GITHUB_USER=<your-username>
$ export GITHUB_REPO=<your-repository>
$ export GITHUB_TOKEN=<your-pat-token>

# Verify you meet conditions
$ flux check --pre

# Can be bootstrapped to any other branch other than 'main'
$ flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=clusters/my-cluster \
  --read-write-key \
  --personal

# Clone the repo on another computer
$ git clone https://github.com/$GITHUB_USER/$GITHUB_REPO

# Create the sops-gpg PGP key after bootstraping Flux so it can decrypt the encoded YAML manifests
$ gpg --export-secret-keys \
      --armor $PUB_KEY |
  kubectl create secret generic sops-gpg \
      --namespace=flux-system \
      --from-file=sops.asc=/dev/stdin
```

## Upgrading the cluster
```bash
# Be sure to set all arguments needed, such as --extra-components if you have any extra controllers enabled, here is an upgrade to 0.29.5
$ flux install --version="0.29.5" --components-extra=image-reflector-controller,image-automation-controller  --export > ./cluster/base/flux-system/gotk-components.yaml
```

### Useful examples
```bash
# Example creating a git source
$ flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval=3m \
  --export > ./clusters/my-cluster/podinfo-source.yaml
```

```bash
# Example creating a Kustomization
$ flux create kustomization podinfo \
  --target-namespace=development \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --interval=5m \
  --export > ./clusters/my-cluster/podinfo-kustomization.yaml
```

### Introducing reconciliation
```bash
# Reconciliation. This process takes changes defined by sources and ensures the cluster reaches the desired state. Flux won't apply the changes pulled in from the sources until there is an associated Kustomization resource telling it to reconcile to the source. 

# Reconcile stuck HelmRelease / stuck flux-system Kustomization
$ flux reconcile helmrelease nginx -n nginx
$ flux reconcile kustomization flux-system -n flux-system
```

### Debugging
```bash
# Debug stuck HelmRelease
NAMESPACE       NAME                    READY   MESSAGE                         REVISION        SUSPENDED
network         helmrelease/traefik     Unknown Reconciliation in progress                      False
$ kubectl describe helmrelease traefik -n network
```