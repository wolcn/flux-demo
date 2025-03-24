## Flux operator and controllers

Ref: [Flux operator site](https://fluxcd.control-plane.io/operator/)    
Ref: [Flux Operator installation](https://fluxcd.control-plane.io/operator/install/)    
Ref: [Flux Controller configuration](https://fluxcd.control-plane.io/operator/flux-config/)    

I install these in the project cluster using a pipeline with the **Terraform** Helm operator, but simplest for a lab cluster is just to use Helm. Default settings are mostly enough for lab work; multi-tenancy needs to be enabled (off by default) and the type of cluster needs to be set to `aws` if deploying on `EKS`. My initial cluster was a Kind cluster with three nodes and for that the  `instance.cluster.type` value is set to `kubernetes`.

The operator:
```
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system \
  --create-namespace
  --set multitenancy.enabled=true
  --wait
```

The controllers:
```
helm -n flux-system install flux oci://ghcr.io/controlplaneio-fluxcd/charts/flux-instance \
  --set instance.cluster.multitenant=true \
  --set instance.cluster.type=kubernetes # replace with aws for eks
```

## Demo/test

This demo is slightly simplified compared with my original setup; I used private GitHub repos and PAT authentication, while for this version the repos have given public visibility and the authentication code has been removed from the manifest files. Given that the demo came to life as a verification tool it is essentially just a hack, but it also functions as an example.

Flux' documentation can be a little confusing as different terms are used for the same concepts and one term that is used for a Flux concept - `kustomization` - is more commonly used in a Kubernetes context for another concept. The term `sync` is also used by Flux for the same thing, which is what I use here.

The Flux stack in the demo is three tier and has two scenarios. The first tier is the operator + controller, installed above; this tier is installed once per cluster and is shared. The second tier is the **tenant tier** - namespace, service account, role binding, authorisation etc - and the third tier is the **sync tier**, which defines the repo along with the reconciliation.

The Flux documentation refers to *bootstrapping* in various places; in the Flux operator context this appears to be included in the tenant and sync tiers.

The two scenarios are of a one-to-one one git repo to one namespace and a one-to-many single git repo to two namespaces. Each namespace for the second scenario has its own tenant and sync. Other scenarios are possible, but haven't been tested.

Scenario one uses the `dev` namespace, while scenario two uses the `stage` and `prod` namespaces. These names are purely arbitrary and reused in different places in the code; mostly there is no significance.

The manifest files were generated using the Flux cli tool; it is not a requirement for using Flux as the operator, but it is very useful.

## Repositories
This demo includes three public GitHub repos; this one with simple documentation and manifest files for setting up the GitOps sync plus repos with the two scenarios described above:

- [Scenario 1](https://github.com/wolcn/flux-dev)
- [Scenario 2](https://github.com/wolcn/flux-stage-prod)

## Applying the demo files

Once the operator and controllers are installed on the cluster, the tenants and syncs from this can be installed. The can currently be found in the sub directories `dev`, `stage` and `prod` in this repo. Because the tenant manifest needs to be applied prior to the sync manifest, I have prefixed the file names with `1_` and `2_` to ensure they get deployed in the right order when running the command `kubectl apply -f .`.

Once the sync is applied, it should only take a couple of minutes or so for the demo pods to appear in the appropriate namespaces.

Things to be checked when trouble shooting are the status of the sync objects, which in this case are `gitrepository` objects. Do this with:
```
kubectl get gitrepository -A`
```
Check the logs of the operator and controller pods, and, if you have the flux cli installed, the flux logs with:
```
flux logs -A
```

## Deploying application changes

To get started it's enough to install the operator as described above and apply the manifest files included in this repo for one or both of the scenarios. 

But to experience the full GitOps experience you'll need to clone at least one of the scenario repos and update the sync details to point to your repo. Make the repo public so you don't need set up authentication and as the demo sync is configured to poll the repo every 60 seconds, it shouldn't take long before the demo application is deployed or you start seeing error messages. Once sync is active, you can change values in the demo application manifest and check what happens with the deployment generation value.

If you have `yq` installed locally, checking the current deployment generation in for example the `fluxdemo-dev` namespace is fairly simple:
```
kubectl -n fluxdemo-dev get deploy kcheck -oyaml | yq .metadata.generation
```

Simplest though is to change the number of replicas.

## The demo application

The demo app `kcheck` is a quick hack I wrote a few years ago when I was playing around with **Rust**; it does what I want it to. It checks the values of a couple of environmental variables and serves a simple web page that changes slightly according to the values of those variables (information about the variables is included in the manifest file). The web server is exposed as a ClusterIP service for this demo; normally I install MetalLB in my local clusters (both Kind and bare metal) so I can use a LoadBalancer service to access the web page without having to do any port forwarding.


## Multi-tenancy

Seems to work; when I check in changes that would result in changes being applied outside of a tenant's specified namespace, nothing happens. To be investigated further.




 



