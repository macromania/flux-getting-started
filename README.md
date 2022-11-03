# Setup

You need a cluster to run `flux` operations. A simple and lightweight cluster management option can be [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

## kind

Install kind

```sh
brew install kind
```

Create a new cluster called `moonshot`

```sh
kind create cluster --name moonshot
```

This will create a new cluster using latest Kind Node Image.
When you list containers using `docker container ls`, you will see the Kind Control Plane.

```sh
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                       NAMES
13ebb8a488b4   kindest/node:v1.25.3   "/usr/local/bin/entr…"   2 minutes ago   Up 2 minutes   127.0.0.1:65063->6443/tcp   moonshot-control-plane
```

By default, the cluster access configuration is stored in `${HOME}/.kube/config` if `$KUBECONFIG` environment variable is not set.  
You can use the new directly from terminal as Kind must have already set your `kubectl` context. You can check current context with:

```sh
kubectl cluster-info
cat ~/.kube/config
```

To see all the clusters you have created, you can use the `get clusters` command.

## Github

You need a repository to push configurations of your cluster so that Flux can pull and apply.
You can create a personal access token to use in place of a password with the command line or with the API.
For that, Flux needs `GITHUB_TOKEN` and `GITHUB_USER` available in the environment.

There are multiple ways to create a `GITHUB_TOKEN`. A simpler way could be using [Github Settings Page](https://github.com/settings/tokens?type=beta).  
More details can be found on this page:
<https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token>.

After creating your token, add it to your environment

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

Check you have everything needed to run Flux by running the following command:

```sh
flux check --pre
```

## Flux

Run the bootstrap command:

```sh
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=<YOUR REPOSITORY> \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

The output will be similar to:

```sh
► connecting to github.com
✔ repository created
✔ repository cloned
✚ generating manifests
✔ components manifests pushed
► installing components in flux-system namespace
deployment "source-controller" successfully rolled out
deployment "kustomize-controller" successfully rolled out
deployment "helm-controller" successfully rolled out
deployment "notification-controller" successfully rolled out
✔ install completed
► configuring deploy key
✔ deploy key configured
► generating sync manifests
✔ sync manifests pushed
► applying sync manifests
◎ waiting for cluster sync
✔ bootstrap finished
```

The bootstrap command above does following:

- Creates a git repository `<YOUR REPOSITORY>` on your GitHub account
- Adds Flux component manifests to the repository
- Deploys Flux Components to your Kubernetes Cluster
- Configures Flux components to track the path /clusters/my-cluster/ in the repository

Your repository will have following directories and files:

```sh
.
└── clusters
    └── my-cluster
        └── flux-system
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml
```

### Pods

This example uses a public repository [stefanprodan/podinfo](github.com/stefanprodan/podinfo).  
Podinfo is a tiny web application made with Go that showcases best practices of running microservices in Kubernetes.  
Podinfo is used by CNCF projects like Flux and Flagger for end-to-end testing and workshops.

Create a GitRepository manifest pointing to podinfo repository’s master branch.
The GitRepository API defines a Source to produce an Artifact for a Git repository revision.

```sh
flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval=30s \
  --export > ./clusters/my-cluster/podinfo-source.yaml
  ```

  The output will be similar to:

  ```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 30s
  ref:
    branch: master
  url: https://github.com/stefanprodan/podinfo

  ```

In the above example:

- A GitRepository named podinfo is created, indicated by the .metadata.name field.
- The source-controller checks the Git repository every five minutes, indicated by the .spec.interval field.
- It clones the master branch of the [stefanprodan/podinfo](https://github.com/stefanprodan/podinfo) repository, indicated by the .spec.ref.branch and .spec.url fields.
- The specified branch and resolved HEAD revision are used as the Artifact revision, reported in-cluster in the .status.artifact.revision field.
- When the current GitRepository revision differs from the latest fetched revision, a new Artifact is archived.
- The new Artifact is reported in the .status.artifact field.

Commit and push the podinfo-source.yaml file to the repository:

```sh
git add . && git commit -m "added podinfo GitRepository"
```

### Deployment

Configure Flux to build and apply the kustomize directory located in the [stefanprodan/podinfo](https://github.com/stefanprodan/podinfo) repository.  

The Kustomization API defines a pipeline for fetching, decrypting, building, validating and applying Kustomize overlays or plain Kubernetes manifests.  
The Kustomization Custom Resource Definition is the counterpart of Kustomize’ `kustomization.yaml` config file.

Use the flux create command to create a Kustomization that applies the podinfo deployment.

```sh
flux create kustomization podinfo \
  --target-namespace=default \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --interval=5m \
  --export > ./clusters/my-cluster/podinfo-kustomization.yaml
```

The output will be similar to:

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 5m0s
  path: ./kustomize
  prune: true
  sourceRef:
    kind: GitRepository
    name: podinfo
  targetNamespace: default
```

In the above example:

- A Flux Kustomization named podinfo is created that watches the GitRepository for artifact changes.
- The Kustomization builds the YAML manifests located at the specified `spec.path`, sets the namespace of all objects to the `spec.targetNamespace`, validates the objects against the Kubernetes API, and finally applies them on the cluster.
- Every 5 minutes, the Kustomization runs a server-side apply dry-run to detect and correct drift inside the cluster.
- When the Git revision changes, the manifests are reconciled automatically. If previously applied objects are missing from the current revision, these objects are deleted from the cluster when spec.prune is enabled.