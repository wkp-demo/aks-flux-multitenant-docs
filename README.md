# Multitenant AKS GitOps Operations with Flux v2

Welcome to _Hack Hours_, a joint initiative by Weaveworks and Microsoft targeted at expert engineers looking to dive deep into containerized and kubernetes native technologies and patterns.

## The scenario

In this iteration we will be walking through a common enterprise scenario. One _platform team_ owns and manages the Kubernetes platform on top of which multiple _application teams_ (aka tenants) deploy cloud native applications.

**The Platform Team** is interested in guaranteeing compliance, security and governance across all applications deployed by numerous application teams on top of multiple platform environments.

**The Application Teams (Tenants)** are interested in autonomy and velocity, they want to be able to be onboarded onto the platform quickly and with little effort, while transparently applying to the necessary guidelines as defined by the platform team.

The original version of Flux did not allow for syncing of multiple repositories.  We will be using Flux2 to manage our platform and applications using the GitOps operating model, and Kubernetes namespaces and policies to isolate and secure multiple applications (tenants) running on our cluster. 

## What you'll need

- One Azure Subscription with the permissions and credits as required to start an AKS service
- One Resource Group with one AKS Cluster
- The Flux CLI
- A GitHub Account for forking the necessary repositories and a configured Personal Access Token.
- A git client and basic git understanding

> Note: The only required PAT scope is `repo`

## The architecture

[Include architecture diagram]

## Repositories

* **[aks-flux-multitenant-docs](https://github.com/wkp-demo/aks-flux-multitenant-docs)**: This repository, which includes documentation, architecture diagrams and some utilities to assist in setting up the initial required resources.
* **[aks-flux-multitenant-platform](https://github.com/wkp-demo/aks-flux-multitenant-platform)**: The repository managed by the platform team. This repository contains the manifests necessary to configure the Kubernetes cluster with the required namespaces, roles, policies and other resources to guarantee application isolation and compliance.
* **[aks-flux-multitenant-app1](https://github.com/wkp-demo/aks-flux-multitenant-app1) and [aks-flux-multitenant-app2](https://github.com/wkp-demo/aks-flux-multitenant-app2)**: Each repo holds a sample application, and would be managed by an independent _application team_. The application in these repos will be deployed to the platform managed by the _platform team_.

For details in the contents of each repository see [repository content walkthrough](#repository-content-walkthroguh)

## Preparation (Optional)

In this repository inside `helpers\terraform` you will find a Terraform template which will create a randomly named resource group and AKS cluster using your current Azure identity.

You can run `terraform apply` for the automated creation of your cluster. Once completed, the following command with configure your local `kubectl` with the proper AKS context:

> Note: Replace the name and resource group names in the command with the ones shown in your terraform apply output!

```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

kubernetes_cluster_name = "meet-jaybird-aks"
resource_group_name = "meet-jaybird-rg"

❯ az aks get-credentials --name meet-jaybird-aks --resource-group meet-jaybird-rg
Merged "meet-jaybird-aks" as current context in /Users/lmurillo/.kube/config
```

## Step by step

### 1. Configure your platform

#### Fork and clone the platform repository

Head over to GitHub and Fork the sample platform repo provided at [https://github.com/wkp-demo/aks-flux-multitenant-platform](https://github.com/wkp-demo/aks-flux-multitenant-platform) by hitting the `Fork` button and choosing your personal account.

Clone your forked copy locally.

```
❯ git clone git@github.com:murillodigital/aks-flux-multitenant-platform.git
Cloning into 'aks-flux-multitenant-platform'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (3/3), done.
```

#### Get the Flux2 Installation CLI

Flux2 includes a convenient installation CLI that will bootstrap all required components in your cluster, store the resulting state in a repository and path of your choosing, and automatically configure flux to track changes and maintain the cluster synchronized following a GitOps operating model.

You can directly run an officially provided script to download the right flux cli binary for your architecture.

```
❯ curl -s https://fluxcd.io/install.sh | sudo bash
[INFO]  Downloading metadata https://api.github.com/repos/fluxcd/flux2/releases/latest
[INFO]  Using 0.13.3 as release
[INFO]  Downloading hash https://github.com/fluxcd/flux2/releases/download/v0.13.3/flux_0.13.3_checksums.txt
[INFO]  Downloading binary https://github.com/fluxcd/flux2/releases/download/v0.13.3/flux_0.13.3_darwin_amd64.tar.gz
[INFO]  Verifying binary download
[INFO]  Installing flux to /usr/local/bin/flux
```

Confirm that the cli tool is installed and available in your `PATH` by running `flux -v`, you should see an output indicating the version installed (it may differ from the version shown above).

#### Check your cluster readiness

Now, you can confirm that your AKS cluster satisfies the requirements that must be met for Flux2 to be installed on it:

```
❯ flux check --pre
► checking prerequisites
✔ kubectl 1.21.0 >=1.18.0-0
✔ Kubernetes 1.19.9 >=1.16.0-0
✔ prerequisites checks passed
```

#### Install Flux2 and have it track your platform repository

Flux's CLI includes a handy feature to deploy all the components required by Flux in a specific cluster, and have it sync up and track a repository of your choosing automatically. In this exercise, you will install your Flux v2 and have it track your for of the platform repository (the one you created [here](#fork-and-clone-the-platform-repository)).

You will need to provide a personal access token so that the CLI can identify against GitHub. See [this link](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) for instructions on how to create your personal access token.

Set your GitHub username and token as environment variables.

```
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

Now run the Flux v2 CLI bootstrap command, using your forked platform repo as `--repository` and `./clusters/production` for path.

```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=aks-flux-multitenant-platform \
  --branch=master \
  --path=./clusters/production \
  --personal
```

#### Inspect Flux2 resources

Flux will install all resources in the `flux-system` namespace. You can validate that flux got installed correctly by inspect the resources in the namespace:

```
❯ kubectl get all -n flux-system
NAME                                           READY   STATUS    RESTARTS   AGE
pod/helm-controller-69667f94bc-dnnw4           1/1     Running   0          2m
pod/kustomize-controller-6977b8cdd4-tcr6f      1/1     Running   0          2m
pod/notification-controller-5c4d48f476-kj6qw   1/1     Running   0          119s
pod/source-controller-b4b88948f-967j2          1/1     Running   0          119s

NAME                              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/notification-controller   ClusterIP   10.0.104.125   <none>        80/TCP    2m3s
service/source-controller         ClusterIP   10.0.112.18    <none>        80/TCP    2m2s
service/webhook-receiver          ClusterIP   10.0.240.150   <none>        80/TCP    2m2s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/helm-controller           1/1     1            1           2m1s
deployment.apps/kustomize-controller      1/1     1            1           2m1s
deployment.apps/notification-controller   1/1     1            1           2m
deployment.apps/source-controller         1/1     1            1           2m

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/helm-controller-69667f94bc           1         1         1       2m1s
replicaset.apps/kustomize-controller-6977b8cdd4      1         1         1       2m1s
replicaset.apps/notification-controller-5c4d48f476   1         1         1       2m
replicaset.apps/source-controller-b4b88948f          1         1         1       2m
```

### 2. Installing global platform components

The `production/platform` path in the repo includes components required for platform-wide operation, such as an ingress controller and [Kyverno](https://kyverno.io/) for runtime policy enforcement.

Kyverno and [kyverno policies](https://kyverno.io/policies/) can be used across multiple clusters, therefore, they are referenced by a `Kustomization` pointing to manifests that live outside the `clusters/production` path.

Kyverno policies will require the use of a service account for each tenant, and 

### 3. On-boarding tenants

All tenants are placed inside a `tenants/` subdirectory inside the applicable cluster, in our case, all tenants can be found inside `clusters/production/tenants`.

Each tenant will get a specific `Namespace` and a `ServiceAccount` with permissions inside that namespace only. One `GitRepository` will be added for each tenant, by convention, tenants will need to place inside `deployment/production` anything they want to have deployed onto their specific namespace.

You will need to fork both application team repositories, and update the URL to match your fork in the `GitRepository` object declared in `tenant-repository.yaml` for each tenant.

```
.
├── README.md
├── clusters
│   └── production
│       ├── platform
│       │   ├── ingress-release.yaml
│       │   ├── ingress-repository.yaml
│       │   ├── kyverno
│       │   │   └── kyverno.yaml
│       │   └── namespace.yaml
│       └── tenants
│           ├── app1
│           │   ├── kustomization.yaml
│           │   ├── namespace.yaml
│           │   ├── rbac.yaml
│           │   ├── tenant-application.yaml
│           │   └── tenant-repository.yaml
│           ├── app2
│           │   ├── kustomization.yaml
│           │   ├── namespace.yaml
│           │   ├── rbac.yaml
│           │   ├── tenant-application.yaml
│           │   └── tenant-repository.yaml
│           └── common
└── kyverno
    ├── core
    │   └── install.yaml
    └── policies
        └── policies.yaml
```

## Repository Content Walk-through

### Platform Repo

#### Platform Components

The `aks-flux-multitenant-platform` repository is owned by the Platform team and holds the state of the core platform components as well as the declaration of the tenants allowed to deploy workloads in any given cluster.

Multiple clusters can be declared inside a single repo, as long as they are stored in different paths, as well as multiple clusters can point to the same state given they're configured to track identical path.

> Only the platform team requires any type of access rights to the platform repo, application teams require no access whatsoever.

In our example, we are storing our cluster state inside the `clusters/production` path.

In that path you will find the following manifests:

* **ingress-release.yaml** and **ingress-repository.yaml**: For our example we are deploying the Nginx Ingress Controller using Helm. In these two manifests you will find a `HelmRepository` and a `HelmRelease`, which effectively make the `nginx-stable` Helm repo available to Flux and installs the `nginx-ingress` chart.
  
* **namespace.yaml**: We will install all platform components inside a specific namespace, aptly called `platform`, which is declared in this manifest.

There is also a `clusters/production/kyverno` directory which installs Kyverno and various policies by using a `Kustomization`. The manifests required to install Kyverno and applicable policies are placed at the root of the repository inside a subdirectory, outside any one specific `clusters` path, so that they can be referenced and used across multiple clusters.

#### Tenants

Each cluster should have a `tenants` directory, and inside it each tenant should get a subdirectory of their own. It would be ideal to follow a convention in which there is consistency across directory names, team names and namespaces. In our scenario as an example, we are using `app1` and `app2` as sample team names, which are consistent with the suffix used in each application specific repository, used as namespace and for tenants directory name.

Each tenant holds pretty much identical resources:

* **kustomization.yaml**: This file wraps all other manifests required for every specific tenant.
* **namespace.yaml**: Creates a namespace for the tenant.
* **rbac.yaml**: Creates a `ServiceAccount`, `Role` and `RoleBinding` for the application team. These resources provide access to the tenant to perform actions only within their own namespace.
* **tenant-application.yaml** and **tenant-repository.yaml**: These resources assign a `GitRepository` for each tenant, and by using a `Kustomization` define the convention to sync up anything in `deployment/production` inside the tenant's application repo with the cluster. Note that the `ServiceAccount` created above is used for `serviceAccountName` in the `Kustomization` so that Flux will isolate that which is synced through that resource to the namespace assigned to the tenant.

### Application Repos

In our scenario each tenant uses a single repository. In practice tenants may have multiple repos, which would require multiple `GitRepository` and `Kustomization` objects to exist in the tenant subdirectory in the platform repo.

All application team repos will be different, depending on the stack they're working with and the architecture of their application, what is most important is that, any manifest placed inside `deployment/production` will be synced up by Flux inside the namespace assigned to the tenant.

In our example, you can find two types of application installs, `app1` uses a `HelmRepository` and `HelmRelease` to install `podinfo`, whereas `app2` includes standard Kubernetes manifests by themselves.
