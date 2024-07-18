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

## General Setup

First, let's discuss how we are going to do it: We will follow the [Apps of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps),
and therefore have one Application for the Glasskube installation itself, and one Application for every installed Glasskube package. 

TBA

## Bootstrap Glasskube

Glasskube relies on in-cluster components and CRDs, making it necessary to first install these in your cluster.

The `glasskube bootstrap` commands, as many other `glasskube` commands, supports a `--dry-run` flag, which in combination with the `-o yaml` argument,
returns the list of manifests that it would apply to cluster without actually applying them. We will use the produced `yaml` output and
commit it into our repo, alongside with an argo `Application` wrapping it.

**Bootstrap**
```
glasskube bootstrap --dry-run -o yaml > apps/glasskube/glasskube/glasskube.yaml
```

TODO not sure about all the paths/directory structure yet, but I think
* every package must be in its own directory, so that we can refer to it explicitly in the wrapping Application
* 

**Argo Application**


More information on bootstrapping can be found [here](https://glasskube.dev/docs/getting-started/bootstrap/).

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


