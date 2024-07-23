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

Note that currently we do not provide an explicit way to sync the clusterpackage's status to the argo application. If you know a good way
how to integrate that with ArgoCD, please let us know. Otherwise, the application will just appear as OK/in-sync, even though it is 
actually still installing in the background (**TODO maybe this is good enough / figure this out?**).

### Installing kube-prometheus-stack

TBA

The [ArgoCD documentation's example](https://github.com/argoproj/argocd-example-apps/tree/master/apps) makes use of helm to template out the dynamic parts of this `Application` resource.
(**TODO** I didn't want to do this since it might feel like helm is required to use Glasskube – I guess advanced users will figure this part out anyway. 
On the other hand, this could be something the CLI command could support? a flag like `--with-argo-application <path-to-application-yaml>` and then you get the additional yaml written to that file. 
In that case, we should also support `--file <path-to-clusterpackage-yaml>` to be consistent)

## Updating packages

There are two options handling package version updates: 
* Using the `glasskube update --dry-run -o yaml` command (WIP, see [glasskube/glasskube#660](https://github.com/glasskube/glasskube/issues/660))
* Integrating [renovate](https://github.com/renovatebot/renovate) into the cluster. 

### Integrating Renovate

Renovate Glasskube Support is still work in progress (see [renovatebot/renovate#29322](https://github.com/renovatebot/renovate/issues/29322)), 
but we will show a proof of concept in the following, since the datasource/versioning part is already available.

Assuming you are working with a Github repository, you can use the Renovate Github App and enable it for that particular repo. 
We can add the following `regexMatcher` to `renovate.json`:

```json
{
  "regexManagers": [
    {
        "fileMatch": [
          ".+\\.(yaml)|(yml)$"
        ],
        "matchStrings": [
          "packageInfo:.*\n.*name: (?<depName>.*)\n.*\n.*version: (?<currentValue>.*)"
        ],
        "datasourceTemplate": "glasskube-packages",
        "versioningTemplate": "glasskube"
    }
  ]
}
```

The manager simply looks for all appearances of 

```yaml
packageInfo:
  name: <depName>
  version: <currentValue>
```

in all the repo's yaml files, where the `depName` and `currentValue` will be used by renovate to extract the current version of this (cluster-)package.

The regex-based approach has some limitations (see below), and they will be resolved with the custom Glasskube Renovate manager. 
However, we can still show that on a general level, Glasskube packages can be updated successfully with this:

TBA downgrade – sync, rerun renovate, see that a PR appears – sync

One issue with this regex-based approach is, that `name` and `version` have to appear in that order, even though schematically it would also be correct the other way around.

Another limitation of the current renovate integration is, that it works only with packages of the [public Glasskube package repo](https://github.com/glasskube/packages).
**TODO** this is a followup task inside the renovate datasource I think? Making the repo configurable, also with auth. 

Last but not least, without a dedicated glasskube manager inside renovate, renovate will not be aware of dependencies. That means, it will simply always try
to update to the latest version, instead of checking whether an update to that version is actually allowed in the used cluster. As a consequence, this could 
lead to somebody installing a version that is not allowed because of dependency restrictions, however, the package operator will not actually install it. The
package status would be set to "Failed" with an error message indicating the dependency conflict, but the previously installed version of the package would
not be touched/destroyed. 

**TODO** making the manager dependency aware will be very interesting and most likely only be possible in the selfhosted renovate, because it actually
needs to somehow interact with the cluster, to see which packages are installed in which versions to do the dependency resolution. 
For exmaple there could be an in-cluster component with an endpoint /latest-available-update?package={package}, that does the dependency check and
returns the latest possible installable version, taking dependencies into account. Maybe we could call this endpoint from the renovate manager. 

## Updating Glasskube

**TODO** probably also with renovate but how?

## Custom Package Repository

TBA

## Known Issues

### Renovate Limitations

TBA but see above

### Dependency Resolution

Installing packages with dependencies is not 100% GitOps-compatible yet, as the dependencies will be created by the operator. 
Consider this: To install a clusterpackage `P` that has a dependency on `D`, one would do `glasskube install P --dry-run -o yaml`, which
would output the clusterpackage custom resource for `P`. However, the dependency `D` will only be resolved at reconciliation time by
the package operator, and will therefore not be represented in the git repository at all. A temporary workaround would be to have a closer look
at the output of the `install` command, which also shows the dependencies which will be installed and in which version. One could then 
manually add the required packages custom resources to the git repo as well. However, this will be tackled in a future version to make the
user experience better, see [glasskube/glasskube#430](https://github.com/glasskube/glasskube/issues/430).

## Summary

TBA
