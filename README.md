## GitOps Flux-based HomeLab: Fully automated Kubernetes and GitOps setup (flux2 + kustomize + helm)

[![test](https://github.com/adavarski/homelab-flux/workflows/test/badge.svg)](https://github.com/adavarski/homelab-flux/actions)
[![e2e](https://github.com/adavarski/homelab-flux/workflows/e2e/badge.svg)](https://github.com/adavarski/homelab-flux/actions)


### Objective: GitOps Flux-based HomeLab: Fully automated Kubernetes and GitOps setup

For this example homelab we assume a scenario with three clusters: dev-1, staging and production.
The end goal is to leverage Flux and Kustomize to manage all clusters while minimizing duplicated declarations.

We will configure Flux to install, test and upgrade a demo app using
`HelmRepository` and `HelmRelease` custom resources.
Flux will monitor the Helm repository, and it will automatically
upgrade the Helm releases to their latest chart version based on semver ranges.

### Dependencies

- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- [Go Task](https://taskfile.dev/installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [k3d](https://k3d.io/#installation)
- [flux](https://fluxcd.io/flux/installation/)
- [Weave GitOps CLI](https://docs.gitops.weave.works/docs/open-source/getting-started/install-OSS/)


### Prerequisites

You will need a Kubernetes cluster version 1.21 or newer.
For a quick local test, you can use [k3d](https://k3d.io/#installation).
Any other Kubernetes setup will work as well though.

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Install the Flux CLI on MacOS or Linux:

```sh
brew install fluxcd/tap/flux
```

Or install the CLI by downloading precompiled binaries using a Bash script:

```sh
curl -s https://fluxcd.io/install.sh | sudo bash
```
Go Task installation (Note: Go Task as a more modern iteration of the Makefile utility):

```sh
$ sudo sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d -b /usr/local/bin
```

<!-- USAGE -->
## Usage
> **NB.** The K3s cluster is using Calico instead of Flannel in order to be able to use Network Policies.

Fork your own copy of this repository to your GitHub account and create a [personal access token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) and export the following variables:

```sh
export GITHUB_TOKEN=<YOUR_PERSONAL_ACCESS_TOKEN>
export GITHUB_USER=<YOUR_GITHUB_USERNAME>
export GITHUB_REPO=<YOUR_FORKED_GITHUB_REPO_NAME>
```

### Local K3s dev-1 cluster
Create the cluster:

```sh
task k8s:create-dev1-cluster
```

Verify that Calico controller deployment is ready:
```sh
task k8s:verify-calico
```

Verify that k3s cluster-1 satisfies flux2 prerequisites:
```sh
task flux:check-prerequisites
```

Install Flux and configure it to manage itself from a Git repository:
```sh
task flux:bootstrap-dev1-cluster
```

Flux2 is configured to deploy content of the `infrastructure` items using Helm before the application. Verify that the infrastructure Helm releases are synchronized to the cluster:
```sh
task flux:get-helmreleases
```

Verify that the api and client applications are synchronized to the cluster:
```sh
task flux:get-kustomizations
```

You can also check Helm and Git repositories and their status:
```sh
task flux:get-helmrepositories
task flux:get-gitrepositories
```

## Repository structure

The Git repository contains the following top directories:

- **apps** dir contains Helm releases with a custom configuration per cluster
- **infrastructure** dir contains common infra tools such as ingress-nginx and cert-manager
- **clusters** dir contains the Flux configuration per cluster

```
├── apps
│   ├── base
│   ├── dev-1
│   ├── production 
│   └── staging
├── infrastructure
│   ├── configs
│   └── controllers
└── clusters
    ├── dev-1
    ├── production
    └── staging
```

### Applications

The apps configuration is structured into:

- **apps/base/** dir contains namespaces and Helm release definitions
- **apps/dev-1/** dir contains the dev-1 Helm release values
- **apps/production/** dir contains the production Helm release values
- **apps/staging/** dir contains the staging Helm release values

```
./apps/
├── base
│   └── podinfo
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── release.yaml
│       └── repository.yaml
├── dev-1
│   ├── kustomization.yaml
│   └── podinfo-patch.yaml
├── production
│   ├── kustomization.yaml
│   └── podinfo-patch.yaml
└── staging
    ├── kustomization.yaml
    └── podinfo-patch.yaml
```

In **apps/base/podinfo/** dir we have a Flux `HelmRelease` with common values for three(all) clusters:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  releaseName: podinfo
  chart:
    spec:
      chart: podinfo
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
  interval: 50m
  values:
    ingress:
      enabled: true
      className: nginx
```

In **apps/dev-1/** dir we have a Kustomize patch with the dev-1 specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0"
  values:
    ingress:
      hosts:
        - host: podinfo.production
```

In **apps/staging/** dir we have a Kustomize patch with the staging specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0-alpha"
  test:
    enable: true
  values:
    ingress:
      hosts:
        - host: podinfo.staging
```

Note that with ` version: ">=1.0.0-alpha"` we configure Flux to automatically upgrade
the `HelmRelease` to the latest chart version including alpha, beta and pre-releases.

In **apps/production/** dir we have a Kustomize patch with the production specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0"
  values:
    ingress:
      hosts:
        - host: podinfo.production
```

Note that with ` version: ">=1.0.0"` we configure Flux to automatically upgrade
the `HelmRelease` to the latest stable chart version (alpha, beta and pre-releases will be ignored).

### Infrastructure

The infrastructure is structured into:

- **infrastructure/controllers/** dir contains namespaces and Helm release definitions for Kubernetes controllers
- **infrastructure/configs/** dir contains Kubernetes custom resources such as cert issuers and networks policies

```
./infrastructure/
├── configs
│   ├── cluster-issuers.yaml
│   ├── network-policies.yaml
│   └── kustomization.yaml
└── controllers
    ├── cert-manager.yaml
    ├── ingress-nginx.yaml
    ├── weave-gitops-dashboard.yaml
    └── kustomization.yaml
```

In **infrastructure/controllers/** dir we have the Flux `HelmRepository` and `HelmRelease` definitions such as:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
  interval: 30m
  chart:
    spec:
      chart: cert-manager
      version: "1.x"
      sourceRef:
        kind: HelmRepository
        name: cert-manager
        namespace: cert-manager
      interval: 12h
  values:
    installCRDs: true
```

Note that with ` interval: 12h` we configure Flux to pull the Helm repository index every twelfth hours to check for updates.
If the new chart version that matches the `1.x` semver range is found, Flux will upgrade the release.

In **infrastructure/configs/** dir we have Kubernetes custom resources, such as the Let's Encrypt issuer:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    # Replace the email address with your own contact email
    email: fluxcdbot@users.noreply.github.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-nginx
    solvers:
      - http01:
          ingress:
            class: nginx
```

In **clusters/production/infrastructure.yaml** we replace the Let's Encrypt server value to point to the production API:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infra-configs
  namespace: flux-system
spec:
  # ...omitted for brevity
  dependsOn:
    - name: infra-controllers
  patches:
    - patch: |
        - op: replace
          path: /spec/acme/server
          value: https://acme-v02.api.letsencrypt.org/directory
      target:
        kind: ClusterIssuer
        name: letsencrypt
```

Note that with `dependsOn` we tell Flux to first install or upgrade the controllers and only then the configs.
This ensures that the Kubernetes CRDs are registered on the cluster, before Flux applies any custom resources.

### Clusters 

Bootstrap dev-1, staging and production clusters.

The clusters dir contains the Flux configuration:

```
./clusters/
├── dev-1
│   ├── apps.yaml
│   └── infrastructure.yaml
├── production
│   ├── apps.yaml
│   └── infrastructure.yaml
└── staging
    ├── apps.yaml
    └── infrastructure.yaml
```

In **clusters/staging/** dir we have the Flux Kustomization definitions, for example:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  dependsOn:
    - name: infra-configs
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/staging
  prune: true
  wait: true
```

Note that with `path: ./apps/staging` we configure Flux to sync the staging Kustomize overlay and 
with `dependsOn` we tell Flux to create the infrastructure items before deploying the apps.

## MANUAL USAGE: not using Go Task file and k3d cluster & local k8s content
Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Verify that your staging cluster satisfies the prerequisites with:

```sh
flux check --pre
```

Set the kubectl context to your staging cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=staging \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/dev-1
```

The bootstrap command commits the manifests for the Flux components in `clusters/staging/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Watch for the Helm releases being installed on staging:

```console
$ watch flux get helmreleases --all-namespaces

NAMESPACE    	NAME         	REVISION	SUSPENDED	READY	MESSAGE 
cert-manager 	cert-manager 	v1.11.0 	False    	True 	Release reconciliation succeeded
flux-system  	weave-gitops 	4.0.12   	False    	True 	Release reconciliation succeeded
ingress-nginx	ingress-nginx	4.4.2   	False    	True 	Release reconciliation succeeded
podinfo      	podinfo      	6.3.0   	False    	True 	Release reconciliation succeeded
```

Verify that the demo app can be accessed via ingress:

```console
$ kubectl -n ingress-nginx port-forward svc/ingress-nginx-controller 8080:80 &

$ curl -H "Host: podinfo.staging" http://localhost:8080
{
  "hostname": "podinfo-59489db7b5-lmwpn",
  "version": "6.2.3"
}
```

Bootstrap Flux on production by setting the context and path to your production cluster:

```sh
flux bootstrap github \
    --context=production \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/production
```

Watch the production reconciliation:

```console
$ flux get kustomizations --watch

NAME             	REVISION     	SUSPENDED	READY	MESSAGE                         
apps             	main/696182e	False    	True 	Applied revision: main/696182e	
flux-system      	main/696182e	False    	True 	Applied revision: main/696182e	
infra-configs    	main/696182e	False    	True 	Applied revision: main/696182e	
infra-controllers	main/696182e	False    	True 	Applied revision: main/696182e	
```


## Weave GitOps (OPTIONAL)

On each cluster, we'll install [Weave GitOps](https://docs.gitops.weave.works/) (an OSS UI for Flux)
to visualise and monitor the workloads managed by Flux.

![flux-ui-apps.png](.github/screens/flux-ui-apps.png)

Install Weave GitOps Open Source on Your Cluster

### Install the gitops CLI
Weave GitOps includes a command-line interface to help users create and manage resources. The gitops CLI is currently supported on Mac (x86 and Arm) and Linux, including Windows Subsystem for Linux (WSL). Windows support is a planned enhancement.

```
curl --silent --location "https://github.com/weaveworks/weave-gitops/releases/download/v0.26.0/gitops-$(uname)-$(uname -m).tar.gz" | tar xz -C /tmp
sudo mv /tmp/gitops /usr/local/bin
gitops version
```
### Deploy Weave GitOps
```
PASSWORD="<A new password you create, removing the brackets and including the quotation marks>"
PASSWORD="flux"
$ echo -n $PASSWORD | gitops get bcrypt-hash
$2a$10$9/V8eA7uFM/boe7Gn.cwyOO9/ILoXKrZxFU03dBs558tohCD3wouW
$ gitops create dashboard weave-gitops \
  --password=$PASSWORD \
  --export > ./infrastructure/controllers/weave-gitops-dashboard.yaml
$ git add -A && git commit -m "Add Weave GitOps Dashboard"
$ git push
```
### Validate that Weave GitOps
```
$ kubectl get pods -n flux-system
NAME                                      READY   STATUS    RESTARTS   AGE
helm-controller-c8466f78b-v7zdh           1/1     Running   0          125m
source-controller-557989894-hjmj2         1/1     Running   0          125m
notification-controller-55d78c78c-bmqmc   1/1     Running   0          125m
kustomize-controller-666f8f4b5f-tkfm4     1/1     Running   0          125m
weave-gitops-58f8bbb47b-985wv             1/1     Running   0          124m

$  kubectl get svc -n flux-system
NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
notification-controller   ClusterIP   10.43.227.18   <none>        80/TCP     152m
source-controller         ClusterIP   10.43.83.89    <none>        80/TCP     152m
webhook-receiver          ClusterIP   10.43.72.157   <none>        80/TCP     152m
weave-gitops              ClusterIP   10.43.47.189   <none>        9001/TCP   151m
```
### Create ingress
```
$ cat ingress-weave-gitops.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: weave-gitops-ingress
  namespace: flux-system
spec:
  ingressClassName: nginx
  rules:
  - host: "gitops.192.168.1.99.nip.io"
    http:
      paths:
      - path: "/"
        pathType: ImplementationSpecific
        backend:
          service:
            name: weave-gitops
            port:
              number: 9001

$ kubectl apply -f ingress-weave-gitops.yaml -n flux-system

$  kubectl get ing -n flux-system
NAME                   CLASS   HOSTS                        ADDRESS                            PORTS   AGE
weave-gitops-ingress   nginx   gitops.192.168.1.99.nip.io   172.27.0.2,172.27.0.3,172.27.0.4   80      70m

```
### Access the Flux UI (Weave GitOps)

To access the Flux UI on a cluster, first start port forwarding with:

```sh
kubectl -n flux-system port-forward svc/weave-gitops 9001:9001
```
Navigate to http://localhost:9001 and login using the username `admin` and the password `flux`.

Or Browser (via nginx ingress): 
- Flux UI: http://gitops.192.168.1.99.nip.io:8888 (admin/flux)

[Weave GitOps](https://docs.gitops.weave.works/) provides insights into your application deployments,
and makes continuous delivery with Flux easier to adopt and scale across your teams.
The GUI provides a guided experience to build understanding and simplify getting started for new users;
they can easily discover the relationship between Flux objects and navigate to deeper levels of information as required.

![flux-ui-depends-on](.github/screens/flux-ui-depends-on.png)

You can change the admin password bcrypt hash in **infrastructure/controllers/weave-gitops-dashboard.yaml**:
```
$ echo -n $PASSWORD | gitops get bcrypt-hash
$2a$10$9/V8eA7uFM/boe7Gn.cwyOO9/ILoXKrZxFU03dBs558tohCD3wouW
```

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: weave-gitops
  namespace: flux-system
spec:
  # ...omitted for brevity
  values:
    adminUser:
      create: true
      username: admin
      # bcrypt hash for password "flux"
      passwordHash: "$2a$10$9/V8eA7uFM/boe7Gn.cwyOO9/ILoXKrZxFU03dBs558tohCD3wouW"
```

To generate a bcrypt hash please see Weave GitOps
[documentation](https://docs.gitops.weave.works/docs/configuration/securing-access-to-the-dashboard/#login-via-a-cluster-user-account). 

Note that on production systems it is recommended to expose Weave GitOps over TLS with an ingress controller and
to enable OIDC authentication for your organisation members.
To configure OIDC with Dex and GitHub please see this [guide](https://docs.gitops.weave.works/docs/guides/setting-up-dex/).

## Add clusters

If you want to add a cluster to your fleet, first clone your repo locally:

```sh
git clone https://github.com/${GITHUB_USER}/${GITHUB_REPO}.git
cd ${GITHUB_REPO}
```

Create a dir inside `clusters` with your cluster name:

```sh
mkdir -p clusters/dev-2
```

Copy the sync manifests from staging:

```sh
cp clusters/staging/infrastructure.yaml clusters/dev-2
cp clusters/staging/apps.yaml clusters/dev-2
```

You could create a dev overlay inside `apps`, make sure
to change the `spec.path` inside `clusters/dev-2/apps.yaml` to `path: ./apps/dev-2`. Fix also apps folder for new cluster (kustomize, etc.). 

Push the changes to the main branch:

```sh
git add -A && git commit -m "add dev-2 cluster" && git push
```

Set the kubectl context and path to your dev cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=dev \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/dev-2
```

## Identical environments

If you want to spin up an identical environment, you can bootstrap a cluster
e.g. `production-clone` and reuse the `production` definitions.

Bootstrap the `production-clone` cluster:

```sh
flux bootstrap github \
    --context=production-clone \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/production-clone
```

Pull the changes locally:

```sh
git pull origin main
```

Create a `kustomization.yaml` inside the `clusters/production-clone` dir:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - flux-system
  - ../production/infrastructure.yaml
  - ../production/apps.yaml
```

Note that besides the `flux-system` kustomize overlay, we also include
the `infrastructure` and `apps` manifests from the production dir.

Push the changes to the main branch:

```sh
git add -A && git commit -m "add production clone" && git push
```

Tell Flux to deploy the production workloads on the `production-clone` cluster:

```sh
flux reconcile kustomization flux-system \
    --context=production-clone \
    --with-source 
```

## Testing

Any change to the Kubernetes manifests or to the repository structure should be validated in CI before
a pull requests is merged into the main branch and synced on the cluster.

This repository contains the following GitHub CI workflows:

* the [test](./.github/workflows/test.yaml) workflow validates the Kubernetes manifests and Kustomize overlays with [kubeconform](https://github.com/yannh/kubeconform)
* the [e2e](./.github/workflows/e2e.yaml) workflow starts a Kubernetes cluster in CI and tests the staging setup by running Flux in Kubernetes Kind

 
## References:
- https://fluxcd.io/flux/get-started/
- https://fluxcd.io/flux/guides/mozilla-sops/
- https://github.com/fluxcd/flux2-multi-tenancy
- https://github.com/fluxcd/flux2-kustomize-helm-example
- https://docs.gitops.weave.works/docs/open-source/getting-started/install-OSS/

## TODO
- Add HashiCorp Vault + k8s External (flux -> infrastructure) based on https://github.com/adavarski/k8s-vault-secrets#demo3-eso-external-secret-operato
- Add SOPS 
