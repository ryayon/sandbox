<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Index

- [Sandbox: Introduction](#sandbox-introduction)
- [Prerequisite](#prerequisite)
- [Deployment](#deployment)
  - [cluster creation](#cluster-creation)
  - [Setup our cluster repository](#setup-our-cluster-repository)
  - [Configuration](#configuration)
  - [Boostrap FluxCD](#boostrap-fluxcd)
  - [What is installed](#what-is-installed)
  - [SKAS (Kubernetes authentication)](#skas-kubernetes-authentication)
  - [Adding a Certificate Authority](#adding-a-certificate-authority)
  - [More about configuration](#more-about-configuration)
- [Roadmap](#roadmap)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# Introduction

Aim of this sandbox is two demonstrate two principles

- A fully automated deployment, starting from a bare kubernetes cluster to a fully featured one, including middleware 
and application. This in a fully automated way.
- Full GitOps, where all action on the cluster will be performed by editing files in git, without direct access to the cluster.

It mostly rely on [Flux](https://fluxcd.io/) 

Here is a short resume of the steps to be performed:

- Create a cluster with Kind
- Copy this repository
- Perform some configuration. 
- Bind the cluster to the repo using the `flux` CLI command.
- Wait for all components to be deployed

# Prerequisite

- Docker Desktop
- [Kind](https://kind.sigs.k8s.io/) 
- For Mac Os: [Docker Mac Net Connect](https://github.com/chipmk/docker-mac-net-connect)
- kubectl
- flux, the [FluxCD CLI client](https://fluxcd.io/flux/installation/#install-the-flux-cli)
- Internet access
- A Github account and a GitHub personal access token with repo permissions. See the GitHub documentation on creating a personal access token.


Optionally:

- [k9s](https://k9scli.io/): A terminal based UI to interact with your Kubernetes clusters
- dnsmasq: To ease resolution of local DNS name. Editing /etc/hosts is a viable alternative.

# Deployment

## cluster creation

```
kind create cluster
```

This will create a fully operational kubernetes cluster, with a single node acting both as control plane and worker.

> It is of course possible to create more sophisticated cluster with several workers or/and control plane node. 
But, take care if you intend to build cluster with more than one control plane node. 
See [kind-fip](https://github.com/kubotal/kind-fip/blob/fixeip/README.md)

You can ensure your cluster is up and running:

```
kubectl get --all-namespaces pods
# or
k9s
```

## Setup our cluster repository

Next step is to create your own copy of this repository. For this, click on the `Use this template` button on the upper 
right corner of this repo main page.

> This procedure assume you have copied the repo into your personal GitHub account, under the name `sandbox`

> It is better to copy the repo using this method than performing a Fork, which is more restrictive.

This repo will be the driver of the state of you cluster. In other words, the included tooling (based on FluxCD) will
permanently reconcile the configuration of the cluster with the content of the repo.

## Configuration

One key point of this sandbox is we will access the provided service through VIP (Virtual IP) on the docker network, 
using a load balancer. This will allow us to refer to services through DNS name, and not by using cumbersome local 
port number. And also this will be more realistic, similar to a 'real' cluster in the cloud or on bare metal.

> This why, on  Mac Os, we require [Docker Mac Net Connect](https://github.com/chipmk/docker-mac-net-connect) to be
installed. It will provide access from the host to the Docker network.

The load balancer used here is [metallb](https://metallb.universe.tf/). We need to provide a range of IP for metallb 
to allocate VIPs. This range must be in the IPv4 subnet used by kind, but without conflict with existing containers.

The first step is to figure out what is this IPv4 subnet. For this, you can issue the following command:

```
docker network inspect kind -f '{{ .IPAM.Config}} }}' | grep -Eo '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}/[0-9]{1,2}'
```

The result is typically `172.18.0.0/16` or `172.19.0.0/16`. For this README, let's assume it is `172.18.0.0/16`. If not, 
you should adjust accordingly.

As docker allocate containers IP from the beginning of the range, we can assume than using IP above 172.18.200.0 is safe.

So, for our cluster we will allocate a small range from `172.18.200.1` to `172.18.200.4`. And the first one will be 
bound to the ingress entry.

To define this range, enter the following in local `/etc/hosts` file:

```
172.18.200.1 first.pool.kind.local ingress.kind.local skas.ingress.kind.local podinfo.ingress.kind.local
172.18.200.4 last.pool.kind.local 
```

`podinfo` and `skas` are applications, which will be accessible through the ingress controller. As the `/etc/hosts` file 
does not accept some wildcard (such as *.ingres.kind.local), each ingress entry must be explicitly bound to the VIP 

Alternatively, if `dnsmasq` is configured on your system, you can configure the following:

```
address=/first.pool.kind.local/172.18.200.1 
address=/.ingress.kind.local/172.18.200.1 
address=/last.pool.kind.local/172.18.200.4 
```

> `address=/.ingress.kind.local/172.18.200.1` is the 'dnsmasq way' to configure wildcard name. So 
`podinfo.ingress.kind.local` and `skas.ingress.kind.local` will resolve to `172.18.200.1`

These value must now be configured in the Git repository. Edit the file `/clusters/kind/kind/context.yaml`:

```
# Context specific to a kind cluster named 'kind'

context:

  cluster:
    name: kind

  apiServer:
    portOnLocalhost: 53220    # <== To configure

  metallb:
    ipRanges:
      - first: 172.18.200.1  # <== To configure
        last: 172.18.200.4   # <== To configure

  ingress:
    urlRoot: ingress.kind.local
    vip: 172.18.200.1    # <== To configure
```

to reflect your subnetwork, if different.

Also, you must configure `apiServer.portOnLocalhost` with the value of the port on localhost exposing the k8s API api 
server. It is the external port bound the the internal port `6443`. To find it; just enter `docker ps`:

```
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS       PORTS                       NAMES
7ef081115829   kindest/node:v1.29.2   "/usr/local/bin/entr…"   7 hours ago   Up 7 hours   127.0.0.1:53220->6443/tcp   kind-control-plane
                                                                                                    -----
```

> If you edit these file locally, after cloning your repo, don't forget to commit....

## Boostrap FluxCD

Now, we can bootstrap our deployment process, by using our Github token and the `flux` CLI command

First, you need to setup some environment variables:

```
export GITHUB_USER=<Your username>
export GITHUB_TOKEN=<your token>
export GITHUB_REPO=sandbox
```

Some points to note here:

- It is assumed the repo was copied into your **personal** GitHub account, under the name `sandbox`. 
If not the case, the command above should be slightly modified, by replacing the option `--personal` with `--owner <your organization>`
- The repository will be updated by the `flux` command. So, the provided token must allow such access.

Then, enter the following:

```
flux bootstrap github \
--owner=${GITHUB_USER} \
--repository=${GITHUB_REPO} \
--branch=main \
--interval 15s \
--personal \
--path=clusters/kind/kind/flux
```

You can have a look on the deployment by using `k9s`.  **It will take several minutes. Be patient.**

> There is some stages in the deployment which involve a restart of the API server. This means the cluster will seem 
frozen for several minutes. Again, be patient. 

All deployments are instances of `helmRelease` FluxCD resources. The deployment processing ends when all `helmReleases` 
are in the ready state:

```
$ kubectl get -n flux-system Helmreleases
NAME                    AGE     READY   STATUS
cert-manager-issuers    6m30s   True    Helm install succeeded for release cert-manager/cert-manager-issuers.v1 with chart kad-issuers@0.1.0+26b4afce3722
cert-manager-main       6m31s   True    Helm install succeeded for release cert-manager/cert-manager-main.v1 with chart cert-manager@v1.14.4
cert-manager-trust      6m31s   True    Helm install succeeded for release cert-manager/cert-manager-trust.v1 with chart trust-manager@v0.9.2
ingress-nginx-main      6m31s   True    Helm install succeeded for release ingress-nginx/ingress-nginx-main.v1 with chart ingress-nginx@4.10.0
kad-controller          6m37s   True    Helm install succeeded for release flux-system/kad-controller.v1 with chart kad-controller@0.2.0-a1
metallb-main            6m30s   True    Helm install succeeded for release metallb/metallb-main.v1 with chart metallb@0.14.4
metallb-pool            6m30s   True    Helm install succeeded for release metallb/metallb-pool.v1 with chart kad-metallb-pool@0.1.0+26b4afce3722
podinfo-main            6m31s   True    Helm install succeeded for release podinfo/podinfo-main.v1 with chart podinfo@6.5.4
reloader-main           6m31s   True    Helm install succeeded for release kube-tools/reloader-main.v1 with chart reloader@1.0.72
replicator-main         6m31s   True    Helm install succeeded for release kube-tools/replicator-main.v1 with chart kubernetes-replicator@2.9.2
secret-generator-main   6m31s   True    Helm install succeeded for release kube-tools/secret-generator-main.v1 with chart kubernetes-secret-generator@3.4.0
skas-main               6m31s   True    Helm upgrade succeeded for release skas-system/skas-main.v2 with chart skas@0.2.2-snapshot
```

You should new be able to connect to the sample application `podinfo` by pointing your favorite browser to 
[https://podinfo.ingress.kind.local/](https://podinfo.ingress.kind.local/). Note, as we currently use only a 
self-signed certificate, you will have to go through some security warning. See below to fix this.

## What is installed

Here is a list of installed components:

- [cert-manager](https://cert-manager.io/)
- [ìngress-nginx](https://github.com/kubernetes/ingress-nginx)
- [metallb](https://metallb.universe.tf/)
- [reloader](https://github.com/stakater/Reloader)
- [replicator](https://github.com/mittwald/kubernetes-replicator)
- [secret-generator](https://github.com/mittwald/kubernetes-secret-generator)
- [skas](https://www.skas.skasproject.com/)

## SKAS (Kubernetes authentication)

SKAS will allow you to act on your kubernetes cluster as an authenticated user and to restrict user's right using 
standard RBAC permissions.

To use SKAS, you will need to install locally a kubectl extension. 
[Instruction here](https://www.skas.skasproject.com/installation/#installation-of-skas-cli) 

Then, you should follow instruction for the [user guide](https://www.skas.skasproject.com/userguide/). 
But, for the impatient, here is a quick process:

First, you must configure your local `~/.kube/config` file:

```
$ kubectl sk init https://skas.ingress.kind.local --authInsecureSkipVerify=true
```

> If you encounter an error such as `The connection to the server 127.0.0.1:53220 was refused....`, the problem could
be an incorrect value of `apiServer.portOnLocalhost` in the `/clusters/kind/kind/context.yaml` described before. 
Set the correct value, commit the change and wait for the skas-main pod to restart automatically.

Now, kubectl access to the cluster need an authentication. At this stage, the only account available is `admin` with password `admin`:

```
$ kubectl get nodes
Login:admin
Password:
Error from server (Forbidden): nodes is forbidden: User "admin" cannot list resource "nodes" in API group "" at the cluster scope
```

The error is fine: We are correctly identified as user `admin`, but this user does not have any rights on the cluster. 
It is only able to manage SKAS users. So, we can bind this user to an existing group, with full rights on the cluster:

```
kubectl sk user bind admin "system:masters"
```

After logout/login, we can now act as system administrator: 

```
$ kubectl sk logout
Bye!

$ kubectl get nodes
Login:admin
Password:
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   96m   v1.29.2
```

At this stage, it is better to change the `admin` password:

```
$ kubectl sk password
Will change password for user 'admin'
Old password:
New password:
Confirm new password:
Password has been changed sucessfully.
```

As `admin` is member of the group `skas-admin`, it will be able to create users and grant them some rights:

```
$ kubectl sk user create larry --commonName "Larry SIMMONS " --email "larry@his-company.com" --password larry123
User 'larry' created in namespace 'skas-system'.

$ kubectl sk user bind larry "system:masters"
GroupBinding 'larry.system.masters' created in namespace 'skas-system'.

$ kubectl sk user bind larry "skas-admin"
GroupBinding 'larry.skas-admin' created in namespace 'skas-system'.

$ kubectl sk login larry
Password:
logged successfully..

$ kubectl get nodes
NAME                 STATUS   ROLES           AGE    VERSION
kind-control-plane   Ready    control-plane   104m   v1.29.2
```

Please, refer to the [SKAS documentation](https://www.skas.skasproject.com/) for more features

## Adding a Certificate Authority

TODO

## More about configuration

TODO

# Roadmap

- Ubuntu VM as host
- Windows as host (If possible ?)
- Vagrant/kubespray based cluster.

