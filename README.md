## Flux operator and controllers

A simple demo stack for the Flux Kubernetes Operator

Ref: [Flux operator site](https://fluxcd.control-plane.io/operator/)    
Ref: [Flux Operator installation](https://fluxcd.control-plane.io/operator/install/)    
Ref: [Flux Controller configuration](https://fluxcd.control-plane.io/operator/flux-config/)    

This first section is relevant if you want to spin up your own cluster and Flux operator; if you already have access to a Kubernetes cluster with the operator installed, this section can be skipped.

I install the operator and controllers in the project cluster using a pipeline with the **Terraform** Helm operator, but simplest for a lab cluster is just to use Helm from the command line. Default settings are mostly enough for lab work; for this demo multi-tenancy needs to be enabled (is disabled by default) and the type of cluster needs to be set to `aws` if deploying on **EKS**. My initial cluster was a Kind cluster with three nodes and for that the  `instance.cluster.type` value is set to `kubernetes`; otherwise EKS clusters hosted by AWS are what we use, in which case the value needs to be set to `aws`.

The operator:
```
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system \
  --create-namespace \
  --set multitenancy.enabled=true \
  --wait
```

The operator pod can sometimes take a few minutes to fully deploy; hence the `--wait`

The controllers:
```
helm -n flux-system install flux oci://ghcr.io/controlplaneio-fluxcd/charts/flux-instance \
  --set instance.cluster.multitenant=true \
  --set instance.cluster.type=kubernetes # replace with 'aws' for eks clusters
```

## Demo/test
Start here if you already access to a lab cluster with the Flux operator installed.   

This demo is slightly simplified compared with my original setup; there I used private GitHub repos and PAT authentication, while for this version the repos have given public visibility and the authentication code has been removed from the manifest files. The demo came to life as a verification tool so it is basically just a hack that has been polished up a little, but it has its uses as an example.

Flux' documentation can be a little confusing as different terms are used for the same concepts and one term that is used for a Flux concept - `kustomization` - is more commonly used in a Kubernetes context for another concept. The term `sync` is also used by Flux for the same thing, which is what I use here.

The Flux stack in the demo is three tier and has two scenarios. The first tier is the operator + controller, installed above; this tier is installed once per cluster and is shared. The second tier is the **tenant tier** - namespace, service account, role binding, authorisation etc - and the third tier is the **sync tier**, which defines the repo along with the reconciliation.

The Flux documentation refers to *bootstrapping* in various places; in the Flux operator context this appears to be included in the tenant and sync tiers.

The two scenarios are of a one-to-one single git repo to one namespace/environment and a one-to-many single git repo to two namespaces/environments. Each namespace for the second scenario has its own tenant and sync. Other scenarios are possible, but haven't been tested.

Scenario one uses the `dev` namespace, while scenario two uses the `stage` and `prod` namespaces. These names are purely arbitrary and reused in different places in the code; mostly there is no significance.

The manifest files were generated using the Flux cli tool; it is not a requirement for using Flux as the operator, but it can be useful for lab work.

## Scenario repos
This demo includes three public GitHub repos; this one with simple documentation and manifest files for setting up the GitOps sync plus repos with the two scenarios described above:

- [Scenario 1](https://github.com/wolcn/flux-dev)
- [Scenario 2](https://github.com/wolcn/flux-stage-prod)

## Applying the demo files

Once the operator and controllers are installed on the cluster, the tenants and syncs from this can be installed. These can currently be found in the sub directories `dev`, `stage` and `prod` in this repo. Because the tenant manifest needs to be applied prior to the sync manifest, I have prefixed the file names with `1_` and `2_` to ensure they get deployed in the right order when running the command `kubectl apply -f .`

Once the sync is applied, it should only take a couple of minutes or so for the demo pods to appear in the appropriate namespaces.

Things to be checked when troubleshooting are the status of the sync objects, which in this case are `gitrepository` objects. Do this with:
```
kubectl get gitrepository -A
```
Check the logs of the operator and controller pods, and, if you have the Flux cli installed, the Flux logs with:
```
flux logs -A
```

**Important note**
We have found that when using a service mesh (Istio in our case) in a Kubernetes cluster where network policy support is enabled, health and readiness probes the Flux operator and controller pods will no longer be accessible and deployment will fail.

The reason for this is that the service mesh takes over network traffic and the IP range used by the mesh is not white-listed by any network policy so gets blocked. A network policy that allowed access to the ports used by the probes from the mesh IP address range solved the issue.

A manifest file for the network policy is included in this repo in the folder [network](./network).

There is very little information about this particular issue and what was there was somewhat confusing, so it's not clear how common it is and YMMV anyway. The solution given here is specific for our situation, but should give some useful clues about solving the issue.

## Deploying application changes

To get started it's enough to install the operator as described above and apply the manifest files included in this repo for one or both of the scenarios. 

But to experience the full GitOps experience you'll need to clone at least one of the scenario repos and update the sync details to point to your own repo. If you make the repo public, you won't need set up authentication and as the demo sync is configured to poll the repo every 60 seconds it shouldn't take long before the demo application is deployed or you start seeing error messages. Once sync is active, you can change values in the demo application manifest and check what happens with the deployment generation value. The web page can reloaded also, if you have configured access to the service endpoint.

If you have `yq` installed locally, checking the current deployment generation in for example the `fluxdemo-dev` namespace is fairly simple:
```
kubectl -n fluxdemo-dev get deploy kcheck -oyaml | yq .metadata.generation
```

Simplest though is to change the number of replicas in the deployment and check for that. 

## The demo application

The demo app `kcheck` is a quick hack I wrote a few years ago when I was playing around with **Rust**; it does what I want it to. It checks the values of a couple of environmental variables and serves a simple web page that changes slightly according to the values of those variables (information about the variables is included in the manifest file). The web server is exposed as a ClusterIP service for this demo; normally I install MetalLB in my local clusters (both Kind and bare metal) to provide load balancing and ingress services.

### Multi-tenancy

Works as expected; changes to namespaces other than that of the tenant are blocked. More comprehensive verification should probably be carried out though as this was just a simple check.

### Pruning

[Pruning](https://fluxcd.io/flux/components/kustomize/kustomizations/#prune) is garbage collection of Kubernetes resources which have had their definition code removed from Flux; it needs to be enabled as it is off by default. There are a number of settings available that give more fine grained control of the garbage collection, which include pruning policy and the disabling of garbage collection for specified resources. 

Not tested.

### Logs

The Flux cli used as described above gets me the logs I've needed so far. It is quite flexible, but is described as *in preview and under development* so probably should only be used manually at this stage.

  - Flux logs cli [`flux logs`](https://fluxcd.io/flux/cmd/flux_logs/)

### OCI artifacts

The container images used in this demo are stored as OCI artifacts in the Github Container Registry and best practice would be to do the same with the manifest packages for the various deployments. This would however reduce visibility of the code and therefore the pedagogical value of this demo package, so the manifests remain stored in old style Github repos where they can be read.

Packaging the manifests into OCI artifacts has been tested though and worked as expected. The Flux sync file - `3_oci_sync.yaml` - for this verification has been left in the [dev](./dev) folder in this repo.

The Flux cli tool provides some useful capabilities for building and managing artifacts:

  - Build [`flux build artifact`](https://fluxcd.io/flux/cmd/flux_build_artifact/)
  - Push [`flux push artifact`](https://fluxcd.io/flux/cmd/flux_push_artifact/)




 



