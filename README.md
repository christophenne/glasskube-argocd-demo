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

(TODO I would expect that it's possible to also have the parent application in the gitops repo – 
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

TODO not sure about all the paths/directory structure yet, but I think every package must be in its own directory, so that we can refer to it explicitly in the wrapping Application

Depending on the selected sync policy, ArgoCD will automatically apply the Glasskube resources or you will have to sync manually.
Either way, you should be able to observe the `glasskube` application, e.g. like this in the UI:

TODO screenshot of only glasskube being there after the first step. 

As soon as the sync has completed, Glasskube will be bootstrapped in your cluster, which you can check by using `glasskube ls`, for example:

```shell

```

## Installing packages

Just as the `bootstrap` command the `install` command supports `--dry-run` and `-o yaml`. We can therefore simulate installing a package
in our cluster and thereby get the `Package` or `ClusterPackage` custom resource that we can put into our git repo. 

Please note: Even though the command does not apply anything, it still requires Glasskube to already be bootstrapped in the cluster, since the CRDs
need to be present, amongst other reasons. If you want to apply all applications in one step/commit (i.e. glasskube + installed packages),
you are of course free to write the `Package` or `ClusterPackage` custom resources yourself – the `--dry-run` + `-o yaml` approach is
just for your convenience. 

### Installing cert-manager

### Installing kube-prometheus-stack

## Updating packages

## Updating Glasskube

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


