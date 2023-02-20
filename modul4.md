# Networking

## Container Network Interface (**`CNI`**)

### Basics which we should know already 

- each Node can talk with other Nodes
- each Pod can talk with other Pods
- each Pod is getting uniq IP address
- K8's by it self do not control / manage whole network communication

### CNI
- specification with a set of libraries for creating plugins (CNI)
- thanks to this, we are separating the topic of running containers from networking
- by default the container / pod has no network interface attached 
- CNI plugin creates network interface and IP addressing configurations for Pod
- CNI plugin is responsible for communication between Pods
- CNI plugin removes interface when Pod is removed

### Popular CNI plugins 
- kubenet - simple networking plugin for Linux 
- Calico 
- Weave
- Azure CNI
- Amazon CNI
- and there is more of them.

### Introduction to K8's Services

```
                        HTTP / HTTPS
                             |
                             |
                             |
                             |
+-Namespace------------------+--------------------------------------+
| Stage                      |                                      |
|                +-SVC-------------------------+                    |
|                |        Static IP            |                    |
|  +-EP---+      |        DNS registry         |                    |
|  | IP   |  <>  |        Label                |                    |
|  |Pools +------+ - NodePort                  |                    |
|  |[...] |      | - LoadBalancer              |                    |
|  +------+      | - ClusterIP                 |                    |
|                | - ExternalName              |                    |
|                |                             |                    |
|                +-----------------------------+                    |
|                            / | \                                  |
|                           /  |  \                                 |
|                          /   |   \                                |
| +-Deployment------------+----+----+----------------------------+  |
| |                      /     |     \                           |  |
| | +-ReplicaSet--------+------+------+------------------------+ |  |
| | |                  /       |       \                       | |  |
| | | +-----Pod1-----+ +-----Pod2-----+ +-----Pod3-----+       | |  |
| | | |  app1-svc    | |              | |              |       | |  |
| | | | Label        | | Label        | | Label        |       | |  |
| | | | 10.1.1.10    | | 10.1.1.11    | | 10.1.1.12    |       | |  |
| | | +--------------+ +--------------+ +--------------+       | |  |
| | +----------------------------------------------------------+ |  |
| +--------------------------------------------------------------+  |
+-------------------------------------------------------------------+
```
- we could access each pod via their own IP address to get into the Pod
    - but this have few problems
        - if the pods gets recreated
        - pods are fleeting thing so they will change 
        - when pod crashes
        - the IP address is the only one we know
- Services guaranty static IP address for Pod's also uniq DNS name in cluster
- DNS names are created in specific schema 
    - `pod_name.namespace.svc.cluster.local`
    - `app1-svc.stage.svc.local` == `10.1.1.10`
- Services types
    - ClusterIP
    - NodePort
    - LoadBalancer
    - ExternalName
- Services uses `Labels` to communicate with Pod's
- Component (`EP`) endpoint is delivering the Pods IP addresses to Service
- When request comes
    - its going ot service (SVC) 
    - Service has it Label and is sending traffic to pods with the same Label
    - Service is asking EP for address IP 
    - Endpoint component is getting back one of the Pod IP address

How this would look in object declaration
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata: 
    name: rs-cart
    labels:
        app: cart
spec:
    replicas: 3
    selector:
        matchLabels: 
            app: cart
    template:
        metadata:
            labels:
                app: cart
        spec:
            containers:
            - name: webapp
              image: k8smaestro/cart:1.0
              ports:
                - containerPort: 8080
```
```yaml
apiVersion: v1
kind: Service
metadata:
    name: cart-svc
spec:
    ports: 
        - port: 80
        - targetPort: 8080
    selector:
        app: cart
```

### Services type short description

- ClusterIP 
    - Default type of service
    - Getting it's own IP address
    - IP address is accessible only inside the cluster
- NodePort
    - Port number is assigned for cluster which is accessible on each node
    - almost the same what `docker run -p` 
    - port is also accessible outside the cluster
    - how this works
    ```
            Public IP :80
        Node1 IP - 192.168.4.1:30080
        Node2 IP - 192.168.4.5:30080
        Node3 IP - 192.168.4.7:30080
            SVC NodePort: 30080
        Pod1 IP - 10.1.1.10:8080
        Pod2 IP - 10.1.1.11:8080
        Pod3 IP - 10.1.1.12:8080
    ```
    - definition in YAML
    ```yaml
    apiVersion: v1
    kind: Service 
    metadata: 
      name: nodeport-svc
    spec:
      ports:
      - port: 80
        targetPort: 8080
        nodePort: 30080
      type: NodePort
    ```
- LoadBalancer
    - foreseen for K8's in cloud such EKS AKS GKS IKS
    - integrates with cloud Load Balancer (alb, nlb)
    - underneath it also uses **NodePort**, and routes traffic to individual nodes
    ```
    Cloud Load Balancer
        |
    K8's Cluster
    Public IP
    NodePort: 30080
    SVC - LoadBalancer
    POD's
    ```
- ExternalName
    - maps Service to DNS entry
    - only IPv4 compatible DNS name is accepted (IP addresses are not supported)
    - does not operate on **`Labels`**
    - can be useful when migrating parts of the system to Kubernetes
    - Example: communication between Kubernetes and an external database
    - how this works
    ```
                        ----------------->  mydatabase.some.dns
    SVC - ExternalName                          Database
      db-service        <-----------------  external DB
       / | \
    POD's
    Server=db-service,Port=5432;Database=myDataBase;UserId=someUser;Password=myPWD
    ```
    Please see that is a better solution then using strict IP address or DNS of the database because if we decide to move DB to the K8's we would need to change the Service type to `PortNode` and only this (we would not need to redeploy Pods) 

### ClusterIP Practice

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deploy
spec:
  replicas: 3
  selector: 
    matchLabels:
      name: k8s-webapp
  template:
    metadata:
      labels:
        name: k8s-webapp
    spec:
      containers:
      - name: webapp-ctr
        image: k8smaestro/web-app:1.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-clusterip-svc
spec:
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  type: ClusterIP
  selector:
    name: k8s-webapp
```
```sh
kubectl apply -f cluster-ip.yml
kubectl get pod -o wide
kubectl get svc -o wide
kubectl describe svc webapp-clusterip-svc
```

### NodePort Practice 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deploy
spec:
  replicas: 3
  selector: 
    matchLabels:
      name: k8s-webapp
  template:
    metadata:
      labels:
        name: k8s-webapp
    spec:
      containers:
      - name: webapp-ctr
        image: k8smaestro/web-app:2.0
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport-svc
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      nodePort: 30080
  type: NodePort
  selector:
    name: k8s-webapp
```

```sh
kubectl apply -f node-port.yml
kubectl get pod -o wide
kubectl get svc -o wide
kubectl describe svc webapp-nodeport-svc
```
### LoadBalancer Practice

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lb-demo-deploy
spec:
  replicas: 3
  selector: 
    matchLabels:
      name: lb-demo
  template:
    metadata:
      labels:
        name: lb-demo
    spec:
      containers:
      - name: lb-ctr
        image: k8smaestro/host-detector:1.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: lb-demo-svc
spec:
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  type: LoadBalancer
  selector:
    name: lb-demo
```

```sh
kubectl apply -f load-balancer.yml
kubectl get svc -o wide
kubectl describe svc lb-demo-svc
```

### Ingress

- object in Kubernetes that manages external access to Kubernetes Services running inside the cluster
- has more possibilities than e.g. Kubernetes Load Balancer Service
- ensuring communication via SSL
- targeting/separating traffic based on URL
- advanced Load Balancing
- IMPORTANT: in Kubernetes we have two types of objects:
    - Ingress Resources 
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: ingress-resources
          annotations: 
            kubernetes.io/ingress.class: "nginx"
        spec:
        ....
        ...
        ..
        .
        ```
        - Ingress Resources itself won't really do anything on its own
        - we need Ingress Controller, which is ***necessary*** to get the desired effect:
            - distribution of traffic to individual Service in the cluster
    - Ingress Controller
        - we must install at least one (or several) Ingress Controller in the cluster ourselves
        - installation: Deployment and a number of additional objects <br/>
        <sub><sup>(ConfigMap, ServiceAccount, RoleBinding, Service, ClousterRole etc.)</sup></sub>
        ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          labels:
            ....
            app.kubernetes.io/component: controller
            ...
          name: ingress-nginx-controller
          namespace: ingress-nginx
        spec:
          ....
          ....
        ```
        - Ingress Controller options:
            - Load Balancer in GCP - GCE
            - Nginx
            - Contour
            - HAProxy
            - Traefik

            <sub><sub>**NOTE** Nginx and GCE are officially supported and developed by Kubernetes</sub></sub>
#### Ingress Controller - Responsibilities

- it's not a Load Balancer - Load Balancer is in this case one of the Ingress Controller elements
- continuous cluster monitoring
    - detection of new Service and Ingress Resource definitions
    - Configuration of the server (e.g. Nginx) according to the current objects in kubernetes
- directing traffic to the appropriate Services based on the definitions contained in the Ingress Resource
- how this works 
    For example if we would not have Ingress Controller specified, we would need to create for each Service Cloud Load Balancer
    ```
    Cloud LB 1 --|--> Service A ---> Pod1 Pod2 Pod3...
    Cloud LB 2 --|--> Service B ---> Pod1 Pod2 ...
    ...
    ```
    so above would not have sense of exist :) 
    therefor we have Ingress Controller to direct the traffic base on Ingress Resources to appropriate Service 
    ```
    Cloud LB --|---> Ingress Controller <----> Ingress Resources
               |        |
               |        |
               |        +--> Service
               |                |
               |                +---> Pod1 Pod2 ...
    ```
- how this would look on real example, for example for domain k8smaestro.com
```
k8smaestro.com
    |
Ingress Resources
    |
Portal Service
    |
   Pod
```
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resources
  annotations:
    kubernetes.io/ingress.class: "nginx" # by annotation we define which ingress type we use
spec:
  backend:
    serviceName: portal-service
    servicePort: 80
```
but lets say we wanna split the traffic to two different `URI`'s
```
 k8smaestro.com
/blog   /product
      |
Ingress Resources
      |
Portal Service
  |         |
 Pod1      Pod2
 /blog     /product
```
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resources
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - http:
      paths:
      - path: /blog
        backend:
          service:
            name: blog-svc
            port: 
              number: 80
      - path: /product
        backend: 
          service:
            name: product-svc
            port:
              number: 80
```
another example with subdomain `example.k8smaestro.com`
```
+-- k8smaestro.com
|    /blog   /product
|    
+-- example.k8smaestro.com
      |
Ingress Resources
      |
Portal Service ----------+
  |         |            |
 Pod1      Pod2         Pod3
 /blog     /product     example.
```
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resources
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: k8smaestro.com
    http:
      paths:
      - path: /blog
        backend:
          service:
            name: blog-svc
            port: 
              number: 80
      - path: /product
        backend: 
          service:
            name: product-svc
            port:
              number: 80
  - host: example.k8smaestro.com
    http:
      paths:
        - backend
            serviceName: example-svc
            servicePort: 80
```

### Ingress Practice 

As short example of ingress and DNS creations
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: bar.foo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service2
            port:
              number: 80
```

```sh
kubectl apply -f ingress-passage.yml
kubectl apply -f ingress-passage-v2.yml
kubectl apply -f ingress-web-api.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: passage-webapp-svc
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    name: passage-webapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: passage-deploy
spec:
  replicas: 3
  selector: 
    matchLabels:
      name: passage-webapp
  template:
    metadata:
      labels:
        name: passage-webapp
    spec:
      containers:
      - name: webapp-ctr
        image: k8smaestro/web-app:1.0
        ports:
        - containerPort: 80
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: passage-v2-webapp-svc
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    name: passage-v2-webapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: passage-v2-deploy
spec:
  replicas: 3
  selector: 
    matchLabels:
      name: passage-v2-webapp
  template:
    metadata:
      labels:
        name: passage-v2-webapp
    spec:
      containers:
      - name: webapp-ctr
        image: k8smaestro/web-app:2.0
        ports:
        - containerPort: 80
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: weather-api-svc
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    name: weather-api
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: weatherapi-deploy
spec:
  replicas: 3
  selector: 
    matchLabels:
      name: weather-api
  template:
    metadata:
      labels:
        name: weather-api
    spec:
      containers:
      - name: webapp-ctr
        image: k8smaestro/web-api:1.0
        ports:
        - containerPort: 80
```

For minikube `minikube addons enable ingress`
```sh
❯ kubectl get ns
NAME              STATUS   AGE
default           Active   22d
ingress-nginx     Active   3m2s
kube-node-lease   Active   22d
kube-public       Active   22d
kube-system       Active   22d
❯  kubectl get -n ingress-nginx po
NAME                                       READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-h4fc8       0/1     Completed   0          3m13s
ingress-nginx-admission-patch-jjcg9        0/1     Completed   0          3m13s
ingress-nginx-controller-87c64747b-bz5h8   1/1     Running     0          3m13s
```

for other cluster checkout this link https://kubernetes.github.io/ingress-nginx/deploy/
and install appropriate version depend on your K8's cluster configurations. To check K8's version use command `kubectl version`
here we can see for version of k8's is supporting https://github.com/kubernetes/ingress-nginx#supported-versions-table

and then we can install ingress controller 
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v[version]/deploy/static/provider/cloud/deploy.yaml
kubectl get pods -n ingress-nginx -l name=ingress-nginx --watch
kubectl get svc --namespace=ingress-nginx
```
and please replace [version] with accurate version of your chose 
