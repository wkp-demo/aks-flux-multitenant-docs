# Multitenant AKS GitOps Operations with Flux v2

Welcome to _Hack Hours_, a joint initiative by Weaveworks and Microsoft targeted at expert engineers looking to dive deep into containerized and kubernetes native technologies and patterns.

## The scenario

In this iteration we will be walking through a common enterprise scenario. One _platform team_ owns and manages the Kubernetes platform on top of which multiple _application teams_ (aka tenants) deploy cloud native applications.

**The Platform Team** is interested in guaranteeing compliance, security and governance across all applications deployed by numerous application teams on top of multiple platform environments.

**The Application Teams (Tenants)** are interested in autonomy and velocity, they want to be able to be onboarded onto the platform quickly and with little effort, while transparently applying to the necessary guidelines as defined by the platform team.

We will be using Flux2 to manage our platform and applications using the GitOps operating model, and Kubernetes namespaces and policies to isolate and secure multiple applications (tenants) running on our cluster.

## What you'll need

- One Azure Subscription with the permissions and credits as required to start an AKS service
- One Resource Group with one AKS Cluster
- The Flux CLI
- A GitHub Account for forking the necessary repositories and a configured Personal Access Token
- A git client and basic git understanding

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

### 2. Onboard each application team (aka. tenant)

#### Create a base directory for all tenants

```
|-- clusters
    |-- production
        |-- tenants
            |-- base
            |   |- namespace.yaml
            |   |- rbac.yaml
            |   |- repository.yaml
            |-- app1
            |-- app2
   
```

#### Declare the common objects required across all tenants

Each tenant will need a namespace, a service account, a cluster role, a Git repository and a Kustomization, we will define by convention the path in which Tenants will have to add their kustomization.yaml files and other manifests for them to be synced up against their namespace.

We will validate via policy that any resource being created by a tenant's kustomizations is created inside their namespace. We will also define network policies that deny traffic between tenant namespaces.

#### Create a directory for each tenant, create a kustomize overlay for each tenant patching base

Our base defines the common structure that will be applied across all tenants, but for each tenant we will need to define some specific attributes, such as name and repository URI. We will use kustomize overlays to override and/or define these custom tenant values without having to duplicate full manifests.


## Repository Content Walkthrough

### Platform Repo

### Application Repos