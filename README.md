# Crossplane + Flux + App using CloudSQL  
This repository is part (the management cluster part) of a demo that shows how to use
Crossplane and Flux to automate the provisioning of cloud infrastructure resources at scale,
and by using GitOps principles.

On high level, the solution relies on a management cluster that can be used to bootstrap
and monitor the status of other 'workload clusters', where the applications/workloads
are deployed.

By design, there was a goal to  not create too much dependency from workload clusters 
to the management cluster beyond the bootstrapping and initial setup phase. As a result, 
the management cluster runs an instance of Crossplane that can spin up workload clusters (GKE clusters)
by leveraging the defined Kubernetese Composite resource and compositions.

Beyond the initial bootstrapping and setup, each workload cluster runs its own instance of Crossplane
that can provision other GCP managed services needed by the application, in this case CloudSQL for PostgreSQL.

Since this is a PoC, there are still a number of steps that need to be performed 
manually. Understanding those steps with the goal of automating as much of them 
as possible will be another benefit of this demo.

## Overview
The following steps describe how to run this demo. There are also some prerequisites
to be able run the demo.

### Prerequisites
  * A Kubernetes cluster for running the management cluster. This can be either
  a local cluster such as `Minikube`, `Kind`, or `k3d` or a managed Kubernetes 
  offering such as `GKE`, `EKS`, or `AKS`.
  * Kubernetes CLI, [kubectl](https://kubernetes.io/docs/tasks/tools/)
  * Flux CLI, [flux](https://fluxcd.io/flux/cmd/)
  * GPG and Mozilla SOPS, [sops](https://fluxcd.io/flux/guides/mozilla-sops/)
  * Access to a managed Kubernetes service. In this demo we use GKE, so GCP access and [gcloud](https://cloud.google.com/sdk/gcloud)
  * A Git repository provider. In this demo, we use github. Make sure you create your
    [Personal Access Token (PAT)](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html) before starting the demo.

### Steps
  0. Install all the prerequisites tools on your machine.
  1. Follow [these instructions](https://fluxcd.io/flux/guides/mozilla-sops/) to encrypt your GCP credentials before storing them in the git repo.
  2. Bootstrap Flux in the management cluster.
  3. Create a secret to provide Flux with the key for decrypting GCP credentials in the cluster.
  4. Instruct Flux to deploy Crossplane, Crossplane GCP provider, and all the
     needed configurations and XRDs in the management cluster.
  5. Submit a claim for a gke cluster to the management cluster.
  6. Wait for the workload (GKE) cluster to be in ready state.
        * Run `gcloud container clusters get-credentials CLUSTER_NAME --region=REGION_NAME`
          to fetch the kubeconfig for the GKE cluster.

## Bootstrap Flux in the management cluster
Once you have brough up the management cluster, whether on your local machine or 
in a cloud environment, bootstrap flux on it:

```
GITHUB_USERNAME=<your github username>
GITHUB_PAT=<your GitHub Personal Access Token>

flux bootstrap github \
  --owner=${GITHUB_USERNAME} \
  --repository=edc-demo \
  --branch=main \
  --path=./management-cluster \
  --personal
```
## Create a sops secret for decrypting secret in the cluster
Make sure you have already created your own GPG key following [these instructions](https://fluxcd.io/flux/guides/mozilla-sops/)

Flux Kustomize controller has built-in support for SOPS and given the configuration,
it can decrypt the encrypted credentials and populate the secret that Crossplane
GCP provider needs in order to access GCP APIs. 

Regarding the configuration that Flux needs, look under `/infra-blueprints/deployment-artifacts/secrets` directory.

Flux also needs the encryption key to be able decrypt the credentials stored
in the git repo. To provide it with the key, do the following:

```
gpg --list-secret-keys crossplane-gcp-provider-creds  
# this command prints the key's fingerprint (key fp):
#   sec   rsa4096 2020-09-06 [SC]
#       1F3D1CED2F865F5E59CA564553241F147E7C5FA4

export KEY_FP=1F3D1CED2F865F5E59CA564553241F147E7C5FA4

# which we use in the following command:
gpg --export-secret-keys --armor "${KEY_FP}"  | kubectl create secret generic sops-gpg --namespace=flux-system --from-file=sops.asc=/dev/stdin
```
## Instruct Flux to deploy Crossplane and related artifacts
All the setup has already been taken care of in the repo under
`/infra-blurprints/crossplane` and `/infra-blueprints/deployment-artifacts`. In addition, 
flux is configured to apply/sync these manifests to the cluster.

## Bootstrap a workload cluster
With Flux and Crossplane deployed and configured, the management cluster
is ready to bootstrap workload clusters.

To create the first workload-cluster, simply copy a cluster claim template
from `utils/claim-templates` to `/workload-clusters` directory, and modify if needed.

```
cp /utils/claim-templates/workload-cluster-1.yaml /workload-clusters
```
and commit the file to the git repository. 

After around a minute, you should see that a new gke cluster is being created.

Once the creation of workload gke cluster has been completed, do not forget to d
ownload the kubeconfig file for the newly created cluster, e.g.:

gcloud container clusters get-credentials workload-cluster-1 --region=us-central1-a 