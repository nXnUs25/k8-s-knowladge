# Module 1

## Introduction

### HowTo use K8 in cloud - Google Kubernetes Engine (GKE)

#### Installation gcloud CLI

follow steps in [GCP instruction](https://cloud.google.com/sdk/docs/install-sdk) to install `gcloud` command line tool

```shell
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-416.0.0-linux-x86_64.tar.gz
tar -xf google-cloud-cli-416.0.0-linux-x86.tar.gz
./google-cloud-sdk/install.sh
```

reload terminal so that the changes take effect and init `gcloud`

`./google-cloud-sdk/bin/gcloud init`

#### Types of Clusters

- performance
- reliable
- load balancing

Kubernetes is combining all three clusters

- ensures reliability
- ensures performance
- ensures load to be balanced equally between nodes

[Borg & Omega the start of Kubernetes](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/43438.pdf)

#### Control Plane (Master Node)

- scheduler
- api server
- etcd (key value database)

##### HA best practices:

To configure K8s in HA mode recommended value of master nodes is 3, the other example of configuring k8s HA is to use odd numbers 3, 5, 7, 9 for master nodes

Number of master nodes should be odd number because of the **etcd** database require quorum, to correctlly updating and storing the state of cluster. Quorum can be calculated by this pattern ***(n/2)+1*** where *n* is the number of nodes.

##### kube-apiserver

Each component in K8s is talking to API servers

- Controller Manager (Controller of Controllers)
    - Node Controller
    - Deployment Controller
    - Endpoints Controller
- Cloud Controller Manager
    - only in public cloud such as AWS GCP Azure
- Scheduler
    - it is watching the apiserver and is waiting on new events
    - it is responsible for assigning pods to nodes depending on
        - availability nodes
        - labels
        - affinity
- Etcd (kv)
- kubelet
- kubectl

#### Data plane (worker nodes)

Worker node includes components such as:

- kubelet
    - which is communicating with api server
    - it is agent which runs on each data node
    - it is watching containers to be working inside of pods
    - it is reporting to master nodes
- kube-proxy
    - network agent which is working on each working node
    - it is responsible for communicating traffic with *Kubernetes Service* in relevant pods
- container runtime
    - it is responsible for running containers and for downloading images from repositories
    - containerd
    - cri-o
    - it is not a Docker anymore since docker is not compatible with *(CRI)* **Container Runtime Interface**

** ***NOTE*** K8s does not serve to start containers, only to management and certification

### Local environments to configure K8s

Recommended:

1. Docker Desktop (only one node) the easiest version to start learning
2. [Minikube](https://minikube.sigs.k8s.io/docs/start/)
3. KIND (Kubernetes in docker) (it has less configuration options than Minikube)
4. [MicroK8s](https://microk8s.io/) (it has less configuration options than Minikube)

### Exercise

```shell
kubectl version
kubectl get nodes
kubectl run first-pod --image=k8smaestro/web-app:1.0
kubectl get pod
kubectl describe po first-pod
kubectl delete pod first-pod
```

### Alternatively free environment with Kubernetes in Cloud

[Free K8s environments](https://github.com/learnk8s/free-kubernetes) no *EKS*

#### Example with GCP

##### Authorization

###### To connect to a cluster in GCP, you need to execute the following commands:

```shell
gcloud auth login
gcloud config set project k8smaestro
gcloud container clusters get-credentials k8smaestro --zone europe-central2-a
```

###### Checking the current context:

```shell
kubectl config current-context
kubectl cluster-info
```

###### Switching between environments

It is possible to communicate via `kubectl` with many clusters (e.g. with local and cloud cluster) using Kubernetes Contexts

###### Displaying the list of contexts:

`kubectl config get-contexts`

###### Context change

`kubectl config use-context <context_name>`

Example 1: Switch context to cluster in docker-desktop

`kubectl config use-context docker-desktop`

Example 2: Switch context to cluster in GCP

`kubectl config use-context gke_k8s_europe-central2-a_k8s`
