# glasskube-argocd-demo

In this guide we explore some options of using Glasskube in a GitOps powered environment with ArgoCD and renovate.
Feel free to try out all the steps yourself in your own test cluster.
We are also eager to get feedback on Glasskube and the guide and are happy to discuss your inputs.

This guide is structured as follows:
* [Prerequisites](#prerequisites)
* [General Setup](#general-setup)
* [Bootstrap Glasskube](#bootstrap-glasskube)
* [Installing packages](#installing-packages)
* [Updating packages](#updating-packages)
* [Updating Glasskube](#updating-glasskube)
* [Custom Package Repository](#custom-package-repository)
* [Known Issues](#known-issues)
* [Summary](#summary)

## Prerequisites

Please note that this guide is aimed at people that already have some experience with ArgoCD/GitOps and are looking
for a reference of how Glasskube and its packages can play a role in such environments. Renovate experience is nice to
have but not required. 

Therefore, we don't start from the very beginning, but instead ask you to make sure you:

* Have ArgoCD installed in your cluster
* Connect an empty repo or your fork of [TBA](https://github.com/glasskube/TBA) to your ArgoCD installation, such that it can serve as the source of truth

In the following we make use of a local minikube cluster. 

This repo already contains the completed solution, so if you aim to follow the guid step by step from scratch, you should start with an empty repo. 

## General Setup

First, let's discuss how we are going to do it: We will follow the [Apps of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps),
and therefore have one Application for every installed Glasskube package, and one Application for Glasskube itself. 

TBA

## Bootstrap Glasskube

Glasskube relies on in-cluster components and CRDs, making it necessary to first install these in your cluster.

The `glasskube bootstrap` command (additional docs [here](https://glasskube.dev/docs/getting-started/bootstrap/)), 
as many other `glasskube` commands, supports a `--dry-run` flag, which in combination with the `-o yaml` argument,
returns the list of manifests that it would apply to cluster without actually applying them. We will use the produced `yaml` output and
commit it into our repo.

```
glasskube bootstrap --dry-run -o yaml > apps/glasskube.yaml
```

Following the [ArgoCD documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern), we create the parent application `glasskube` manually via the argo CLI or UI. 
Of course you can also define it as an `Application` custom resource and apply it with `kubectl` 

(**TODO** I would expect that it's possible to also have the parent application in the gitops repo – 
but I don't know how it would initially get picked up by argo? – also in the Argo docs it's done manually via CLI). 

Most notably, we indirectly refer to the previously generated `yaml` in `path`. This is what the custom resource could look like
after the application has been created:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: glasskube
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
  project: default
  source:
    path: apps
    repoURL: https://github.com/glasskube/TBA
    targetRevision: HEAD
```

**TODO** not sure about all the paths/directory structure yet, but I think every package must be in its own directory, so that we can refer to it explicitly in the wrapping Application

Depending on the selected sync policy, ArgoCD will automatically apply the Glasskube resources or you will have to sync manually.
Either way, you should be able to observe the `glasskube` application like this in the UI:

TODO screenshot of only glasskube being there after the first step. 

As soon as the sync has completed, Glasskube will be bootstrapped in your cluster, which you can check by using `glasskube version`, for example:

```
$ glasskube version
  ____ _        _    ____ ____  _  ___   _ ____  _____ 
 / ___| |      / \  / ___/ ___|| |/ / | | | __ )| ____|
| |  _| |     / _ \ \___ \___ \| ' /| | | |  _ \|  _|  
| |_| | |___ / ___ \ ___) |__) | . \| |_| | |_) | |___ 
 \____|_____/_/   \_\____/____/|_|\_\\___/|____/|_____|									  
	
glasskube: v0.13.0
package-operator: v0.13.0
```

We can also have a look at the available clusterpackages and packages:

```
$ glasskube ls
PACKAGENAME  NAMESPACE  NAME  VERSION  AUTO-UPDATE  REPOSITORY  STATUS
quickwit                                            glasskube   Not installed  

NAME                      VERSION  AUTO-UPDATE  REPOSITORY  STATUS
argo-cd                                         glasskube   Not installed  
caddy-ingress-controller                        glasskube   Not installed  
cert-manager                                    glasskube   Not installed  
cloudnative-pg                                  glasskube   Not installed  
cyclops                                         glasskube   Not installed  
gateway-api                                     glasskube   Not installed  
headlamp                                        glasskube   Not installed  
ingress-nginx                                   glasskube   Not installed  
k8sgpt-operator                                 glasskube   Not installed  
keptn                                           glasskube   Not installed  
kube-prometheus-stack                           glasskube   Not installed  
kubernetes-dashboard                            glasskube   Not installed  
kubetail                                        glasskube   Not installed  
rabbitmq-operator                               glasskube   Not installed  
sealed-secrets                                  glasskube   Not installed  
```

## Installing packages

Just as the `bootstrap` command the `install` command supports `--dry-run` and `-o yaml`. We can therefore simulate installing a package
in our cluster and thereby get the `Package` or `ClusterPackage` custom resource that we can put into our git repo. 

Please note: Even though the command does not apply anything, it still requires Glasskube to already be bootstrapped in the cluster, since the CRDs
need to be present, amongst other reasons. If you want to apply all applications in one step/commit (i.e. glasskube + installed packages),
you are of course free to write the `Package` or `ClusterPackage` custom resources yourself – the `--dry-run` + `-o yaml` approach is
just for your convenience. 

### Installing cert-manager

The following command will simulate installing the `cert-manager` clusterpackage in its latest version.
Make sure to disable auto updates by Glasskube, as we will instead use renovate (see below).

```
$ glasskube install cert-manager --dry-run -o yaml > apps/cert-manager/clusterpackage.yaml

Version not specified. The latest version v1.15.1+1 of cert-manager will be installed.
Would you like to enable automatic updates? (y/N) N
Summary:
 * The following packages will be installed in your cluster (glasskube-argocd-demo):
    1. cert-manager (version v1.15.1+1)
 * Automatic updates will be not enabled
Continue? (Y/n) 
✅ cert-manager is now installed in glasskube-argocd-demo.
```

Note that the created resource has a flaw in the current version (`v0.13.0`), which is that too much `metadata` is being set.
Of the created metadata, only the `name` should be included – for now you have to manually remove the rest to avoid weird out-of-sync
issues with ArgoCD. You can find the corresponding issue [here](https://github.com/glasskube/glasskube/issues/1008). 

Following the apps of apps pattern again, we want to wrap our clusterpackage in an argo `Application` custom resource
in `apps/cert-manager.yaml`, which holds the reference to the clusterpackage resource:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    server: https://kubernetes.default.svc
  project: default
  source:
    path: apps/cert-manager
    repoURL: https://github.com/glasskube/TBA
    targetRevision: HEAD
```

After commiting and pushing these two files, you can sync the new application via ArgoCD. After the sync, the `cert-manager` application
will appear in ArgoCD. You can observe the clusterpackages status via the Glasskube CLI:

```
$ glasskube describe cert-manager
Package: cert-manager — X.509 certificate management for Kubernetes and OpenShift
Version:     v1.15.1+1
Status:      Ready
Message:     flux: Helm install succeeded for release cert-manager/cert-manager-cert-manager.v1 with chart cert-manager@v1.15.1
Auto-Update: Disabled

Package repositories:
 * glasskube (installed)

References: 
 * Glasskube Package Manifest: https://packages.dl.glasskube.dev/packages/cert-manager/v1.15.1+1/package.yaml
 * ArtifactHub: https://artifacthub.io/packages/helm/cert-manager/cert-manager

Long Description:
A Helm chart for cert-manager
```

**TODO** follow up issue or argo integration question: how could we sync the clusterpackage status to be reflected in the argo application?

### Installing kube-prometheus-stack

The [ArgoCD documentation's example](https://github.com/argoproj/argocd-example-apps/tree/master/apps) makes use of helm to template out the dynamic parts of this `Application` resource.

## Updating packages

## Updating Glasskube

## Custom Package Repository

## Known Issues

## Summary

---

old:

### Parent App

Create the parent glasskube app manually via the UI or CLI. 

Ref: https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#helm-example

### Bootstrap Glasskube

```
glasskube bootstrap --dry-run -o yaml > apps/glasskube.yaml
```

Afterwards, push it into your repo and sync the glasskube application. 
It is necessary that bootstrap is completed, in order for the following install commands to work:

### Install a package

```
glasskube install cert-manager --dry-run -o yaml --yes > apps/packages/cert-manager/clusterpackage.yaml
```

As we are following the [apps of apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps),
you should also create an argocd `Application` for `cert-manager` (see `apps/cert-manager.yaml`).


