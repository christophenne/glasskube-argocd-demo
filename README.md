# glasskube-argocd-demo

## Prerequisites

* Have argocd installed in your cluster
* Have a gitops repo connected to argocd

## Steps

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
glasskube install cert-manager --dry-run -o yaml --yes > apps/packages/cert-manager/package.yaml
```

As we are following the [apps of apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps),
you should also create an argocd `Application` for `cert-manager` (see `apps/cert-manager.yaml`).


