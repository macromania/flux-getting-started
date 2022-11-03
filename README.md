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
