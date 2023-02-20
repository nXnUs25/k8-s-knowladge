# Module 2

## The smallest unit in K8s

### Process of implementation in docker vs K8's implementation

---
#### Docker: 

From source of code (**SRC**) (**Dockerfile**) arises docker image, the image goes into the repository of images (**Container Registry**) and then base on the image the container is created with the app (**Container / App**)

*SRC -> Dockerfile -> Docker Image -> Container Registry -> Container / App*

#### K8s: 

In Kubernetes the container is not created directly, but instead of this created is (**Pod**) which is running  container / containers inside of it.

*SRC -> Dockerfile -> Docker Image -> Container Registry -> Pod -> Container*

---

### Deploying the Pod

`kubectl proxy` to redirect application port to local host

to access pod's

http://localhost:8001/api/v1/namespaces/default/pods/http:hello-pod:/proxy/

http://localhost:8001/api/v1/namespaces/default/pods/http:hello-pod-2:/proxy/


#### Declarative approach

```shell
kubectl apply -f pod.yml
kubectl get pod
kubectl describe pod hello-pod
```

```shell
kubectl apply -f pod-declarative.yml
kubectl get pod webapp-declarative -o yaml > webapp-declarative-snapshot.yml
```

#### Imperative approach

```shell 
kubectl run hello-pod-2 --image k8smaestro/web-app:1.0
kubectl get pod
kubectl describe pod hello-pod-2
```

```shell 
kubectl run webapp --image k8smaestro/web-app:1.0 –replicas=3
kubectl get pod webapp -o yaml > webapp.yml
kubectl create -f pod-imperative-object.yml
kubectl get pod
kubectl delete -f pod-imperative-object.yml
```

#### Cleaning

```shell
kubectl delete -f pod.yml
kubectl delete pod hello-pod-2
```

### Containers vs Pod

- `pod` is the logical and parent entity above the container
- `pod` can contain more than one container
- all containers in `pod` share a network namespace, where in *docker*, each container had a separate, isolated namespace
- containers in `pod` also share namespace IPC (*inter-proc communication*)
this means that all containers running in the pod have access to in-memory segments, semaphores and message queues - in short they share memory
- by default, containers in `pod` have an isolated process tree (*PID namespace*) - however, this can be turned off
```yaml
    spec:
     sharedProcessNamespace: true
```

### Pod vs Docker

```yaml
apiVersion: v1
kind: Pod
metadata: 
    name: web-app
spec:
  containers: 
  - name: web-app-ctr
    image: httpd:2.4.47
    ports: 
    - containerPort: 80
  - name: sync-ctr
    image: k8smaestro/git:2.11.1
```

```shell 
docker run -d --name web-app-ctr httpd:2.4.47
docker run -d --name sync-ctr --network container:web-app-ctr --ipc container:web-app-ctr k8smaestro/git:2.11.1
```

|   Docker CLI	|  Kubectl 	|
|:-:	|:-:	|
|   `docker ps`   	|  `kubectl get pod`	|
|   `docker attach -it <-id->`	|  `kubectl attach -it <-pod-id->`	|
|   `docker exec -ti <-id-> /bin/sh`	|   `kubectl exec -ti <-pod-id-> -- /bin/sh`	|
|   `docker logs -f <-id->`	|   `kubectl logs -f <-pod-id->`	|
|   `docker stop <-id->`    |   `kubectl delete pod<-pod-id->`	|
|    `docker rm <-id->`	    |


### Containers in Kubernetes are fleeting

- Fire & Forget
- the task of Kubernetes is not to ensure that the same container works continuously and without interruption, but that the number and state of the containers match
- this task is performed by one of the basic mechanisms in Kubernetes - **ReplicaSet**

### ReplicationController vs ReplicaSEt

- ReplicaSet is the recommended approach

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: rs-webapp
    labels: 
        app: k8smaestro-webapp

spec:
    replicas: 3

    selector:
        matchLabels:
            app: k8smaestro-webapp
```

### ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-webapp
  labels:
    app: k8maestro-webapp
  spec: 
   replicas: 3
   selector: 
     matchLabels: 
       apps: k8smaestro-webapp
   template: 
     metadata:
       labels:
         app: k8smaestro-webapp
     spec:
       containers:
       - name: webapp
         image: k8smaestro/web-app:1.0
         ports:
           - containerPort: 80
```

```shell
kubectl apply -f replicaset.yml
kubectl get rs rs-webapp
kubectl describe rs/rs-webapp

kubectl apply -f pod-rs.yml
kubectl describe rs/rs-webapp

kubectl delete -f replicaset.yml
kubectl apply -f pod-rs.yml
kubectl get pod
kubectl apply -f replicaset.yml
kubectl describe rs/rs-webapp
kubectl delete -f replicaset.yml
kubectl get pod
```
Cleanup
```shell
kubectl delete -f replicaset.yml
```

### Deployment 

- binds together the application / solution that we want to run on kubernetes
- it can be compared to docker-compose, with one difference deployment is per service, 
- docker-compose aggregates many services together
- but deployment has much more possibilities which we can do:
    - will update the versions
    - will rollback to the previous version
    - will scale down or up

```
Deployment                                  Deployment - high level control, 
+------------------------------------+                   as version upgrades, rollbacks
|   ReplicaSet                       |      ReplicaSet - takes care of running pods
|   +----------------------------+   |                   state, Desired state, scaling
|   |   Pod                      |   |      Pod        - Specification of how container
|   |   +--------------------+   |   |                   containers will be running
|   |   |                    |   |   |
|   |   |   App <container>  |   |   |
|   |   |                    |   |   |
|   |   +--------------------+   |   |
|   |                            |   |
|   +----------------------------+   |
|                                    |
+------------------------------------+
```

#### Deployment example in **`YAML`** definition 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deploy
  labels: 
    apps: k8smaestro-webapp
spec:
  replicas: 3
  template:
    metadata: 
      labels:
        apps: k8smaestro-webapp
    spec:
      containers:
      - name: webapp
        image: k8smaestro/web-app:1.0
  selector:
    matchLabels:
      app: k8smaestro-webapp
```

```shell
kubectl apply -f deployment.yml
kubectl get deployment
kubectl describe deployment webapp-deploy
kubectl get all
kubeclt get rs
kubectl get deploy webapp-deploy -o yaml
kubectl rollout history deployment/webapp-deploy
kubectl rollout undo deployment/webapp-deploy
kubectl get all
kubectl get deploy webapp-deploy -o yaml
```

### Namespace

- most objects in kubernetes are assigned to a specific namespace:
    - pod
    - replicaset
    - deployment
    - service
- if we do not specify where the object should be placed, it will go to namespace *`default`*
- examples of objects which are not assigned to any namespace 
    - `kubectl api-resources --namespaced=false`
    - PersistentVolume
    - IngressClass
    - StorageClass
    - and so on .. there is more of it :) 

#### Namespace YAML

```yaml
apiVersion: v1
kind: Namespace 
metadata: 
  name: webapp-namespace
```

#### `namespace` as kind of `virtual cluster`

- we can use namespace to create several environments for applications within the same physical cluster, for example: Dev-1, Pre-Staging, Staging
- so-called "poor-insulation"
- it is not recommended to isolate the production version of the application with namespaces - unless it is done consciously, more later in Network Policies 

To obtain the list of resources which belong to namespaces we can get by running this command:
`kubectl api-resources --namespaced=true`

```shell
❯ kubectl api-resources --namespaced=true
NAME                        SHORTNAMES   APIVERSION                     NAMESPACED   KIND
bindings                                 v1                             true         Binding
configmaps                  cm           v1                             true         ConfigMap
endpoints                   ep           v1                             true         Endpoints
events                      ev           v1                             true         Event
limitranges                 limits       v1                             true         LimitRange
persistentvolumeclaims      pvc          v1                             true         PersistentVolumeClaim
pods                        po           v1                             true         Pod
podtemplates                             v1                             true         PodTemplate
replicationcontrollers      rc           v1                             true         ReplicationController
resourcequotas              quota        v1                             true         ResourceQuota
secrets                                  v1                             true         Secret
serviceaccounts             sa           v1                             true         ServiceAccount
services                    svc          v1                             true         Service
controllerrevisions                      apps/v1                        true         ControllerRevision
daemonsets                  ds           apps/v1                        true         DaemonSet
deployments                 deploy       apps/v1                        true         Deployment
replicasets                 rs           apps/v1                        true         ReplicaSet
statefulsets                sts          apps/v1                        true         StatefulSet
localsubjectaccessreviews                authorization.k8s.io/v1        true         LocalSubjectAccessReview
horizontalpodautoscalers    hpa          autoscaling/v2                 true         HorizontalPodAutoscaler
cronjobs                    cj           batch/v1                       true         CronJob
jobs                                     batch/v1                       true         Job
leases                                   coordination.k8s.io/v1         true         Lease
endpointslices                           discovery.k8s.io/v1            true         EndpointSlice
events                      ev           events.k8s.io/v1               true         Event
ingresses                   ing          networking.k8s.io/v1           true         Ingress
networkpolicies             netpol       networking.k8s.io/v1           true         NetworkPolicy
poddisruptionbudgets        pdb          policy/v1                      true         PodDisruptionBudget
rolebindings                             rbac.authorization.k8s.io/v1   true         RoleBinding
roles                                    rbac.authorization.k8s.io/v1   true         Role
csistoragecapacities                     storage.k8s.io/v1              true         CSIStorageCapacity
```
```shell 
❯ kubectl api-resources --namespaced=false
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
componentstatuses                 cs           v1                                     false        ComponentStatus
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
persistentvolumes                 pv           v1                                     false        PersistentVolume
mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io/v1              false        APIService
tokenreviews                                   authentication.k8s.io/v1               false        TokenReview
selfsubjectaccessreviews                       authorization.k8s.io/v1                false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io/v1                false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io/v1                false        SubjectAccessReview
certificatesigningrequests        csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
flowschemas                                    flowcontrol.apiserver.k8s.io/v1beta2   false        FlowSchema
prioritylevelconfigurations                    flowcontrol.apiserver.k8s.io/v1beta2   false        PriorityLevelConfiguration
ingressclasses                                 networking.k8s.io/v1                   false        IngressClass
runtimeclasses                                 node.k8s.io/v1                         false        RuntimeClass
clusterrolebindings                            rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io/v1           false        ClusterRole
priorityclasses                   pc           scheduling.k8s.io/v1                   false        PriorityClass
csidrivers                                     storage.k8s.io/v1                      false        CSIDriver
csinodes                                       storage.k8s.io/v1                      false        CSINode
storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment
```

General commands per namespace
```shell
kubectl apply -f namespace.yml
kubectl get namespaces
kubectl get pods -n webapp-namespace
kubectl get all -n webapp-namespace
kubectl delete -f namespace.yml
```
Specifying the default namespace for querying objects
```shell
kubectl config set-context --current --namespace=webapp-namespace
kubectl get pods
kubectl config set-context --current --namespace=default
```
Resource Quota
```shaell
kubectl apply -f ns-resource-quota.yml
kubectl get pod -n dev
kubectl get resourcequota -n dev
```

### Deploying an app 

`all-in-one.yml`
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: game-2048
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: game-2048
  name: deployment-2048
spec:
  selector:
    matchLabels:
      name: app-2048
  replicas: 5
  template:
    metadata:
      labels:
        name: app-2048
    spec:
      containers:
      - image: k8smaestro/2048
        imagePullPolicy: Always
        name: app-2048
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  namespace: game-2048
  name: service-2048
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: ClusterIP
  selector:
    name: app-2048
```

```shell
❯ k apply -f  all-in-one.yml
namespace/game-2048 created
deployment.apps/deployment-2048 created
service/service-2048 created
```

Run `k proxy` in a separate terminal window 
to access it goto http://localhost:8001/api/v1/namespaces/game-2048/services/http:service-2048:/proxy/
in browser


accessing app in a pod via proxy 
- http://localhost:8001/api/v1/namespaces/default/pods/http:hello-pod:/proxy/
accessing app by service via proxy
- http://localhost:8001/api/v1/namespaces/game-2048/services/http:service-2048:/proxy/

If this is tested on minikube to get LoadBalancer public IP do 

```shell
❯ minikube service -n k8sdemo k8sdemo-frontend
|-----------|------------------|-------------|---------------------------|
| NAMESPACE |       NAME       | TARGET PORT |            URL            |
|-----------|------------------|-------------|---------------------------|
| k8sdemo   | k8sdemo-frontend |          80 | http://192.168.58.2:30884 |
|-----------|------------------|-------------|---------------------------|
🎉  Opening service k8sdemo/k8sdemo-frontend in default browser...
```
```shell
minikube service -n k8sdemo list
|-----------|------------------|--------------|---------------------------|
| NAMESPACE |       NAME       | TARGET PORT  |            URL            |
|-----------|------------------|--------------|---------------------------|
| k8sdemo   | k8sdemo-crystal  | No node port |
| k8sdemo   | k8sdemo-frontend |           80 | http://192.168.58.2:30884 |
| k8sdemo   | k8sdemo-nodejs   | No node port |
|-----------|------------------|--------------|---------------------------|
```

to get Loabalancer externalIP 
```shell
❯ minikube ip
192.168.58.2
```

```shell
❯ k edit svc -n k8sdemo k8sdemo-frontend
service/k8sdemo-frontend edited
❯ k get service -n k8sdemo
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
k8sdemo-crystal    ClusterIP      10.103.155.195   <none>         80/TCP         17m
k8sdemo-frontend   LoadBalancer   10.102.240.111   192.168.58.2   80:30884/TCP   17m
k8sdemo-nodejs     ClusterIP      10.109.20.121    <none>         80/TCP         17m
```

we edited the service yaml config and added `externalIPs`
```yaml
❯ k get svc -n k8sdemo k8sdemo-frontend -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"k8sdemo-frontend","namespace":"k8sdemo"},"spec":{"ports":[{"port":80,"protocol":"TCP","targetPort":3000}],"selector":{"app":"k8sdemo-frontend"},"type":"LoadBalancer"}}
  creationTimestamp: "2023-02-13T03:20:19Z"
  name: k8sdemo-frontend
  namespace: k8sdemo
  resourceVersion: "950966"
  uid: e2c5aa5e-9ee7-4032-9582-2b91930edfbf
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 10.102.240.111
  clusterIPs:
  - 10.102.240.111
  externalIPs:
  - 192.168.58.2
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 30884
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app: k8sdemo-frontend
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer: {}
```