# Module 3

## Passing arguments 

### ENTRYPOINT vs CMD in docker and equivalent commands in K8s

Example of recommended form:

Definition:
```dockerfile
ENTRYPOINT ["executable", "param1", "param2"]
CMD ["executable", "param1", "param2"]
```
Example:
```dockerfile
ENTRYPOINT ["nginx", "-g", "daemon off"]
CMD ["nginx", "-g", "daemon off"]
```

```dockerfile
FROM node:lts
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 8080

ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["node", "server.js"]
```

the `CMD` command is passing arguments to the `ENTRYPOINT`

How this looks in K8's 

```yaml
apiVersion: app/v1
kind: Pod
metadata: 
  name: web-app
spec: 
  containers: 
    - name: webapp
      image: k8smaestro/web-app:1.0
      command: ["docker-entrypoint.sh"]
      args: ["bash"]
```

K8's `command` it is equivalent ot `ENTRYPOINT` and `args` are equivalent to `CMD`

### Environment variables

- K8's as Docker uses Env Variables to pass configuration parameters 
    - passing URL address to the backend 
    - passing URL's to  some microservice 
    - to setup configuration of the service 
    - setting flag ON/OFF
- in K8's we have more options to pass environment variables 

as example in docker:

```sh
> docker run --name frontend -e API_URL=https://api.example.com \
    k8smaestro/projectapp-frontend:1.0
```
*`-e API_URL=https://api.example.com `* the way we passed it to docker

and example how we would do this in K8's

```yaml
apiVersion: apps/v1
kind: Pod
metadata: 
  name: frontend
spec: 
  containers: 
    - name: frontend-ctr
      image: k8smaestro/projectapp-frontend:1.0
      ports:
        containerPort: 8080
      env: 
      - name: API_URL
        value: "https://api.example.com"
```

that how we passed env variables in k8's
```yaml 
env: 
- name: API_URL
  value: "https://api.example.com"
```

### Other ways of how to pass environment vars 

1. key-value 
    ```yaml
    env: 
    - name: API_URL
      value: "https://api.example.com"
    ```
2. ConfigMap
    ```yaml
    env:
    - name: API_URL
      valueFrom: 
        configMapKeyRef:
    ```
3. Secrets
    ```yaml
    env:
    - name: API_URL
      valueFrom: 
        secretKeyRef:
    ```
### ConfigMaps

- in Docker - config files were mounted with volume to the container
- in K8's this do not have sense since we have distributed environment 
- mounting config files form host in k8s it is to be considered as anti-pattern
- therefor we have ConfigMaps
- and cos we have ConfigMaps, when we need to share the configurations between other containers we could use just one ConfigMap definition and use it as many times as needed

### How we create / use ConfigMap

- in Yaml key-value
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: multimap
data:
  given: Auggie
  family: Chmiel
```
and we could use this as this in a pod with nginx as example simple config:
```yaml
apiVersion: v1
kind: ConfigMap
metadata: 
  name: nginx-conf
data: 
  nginx.conf: |
    server {
        listen 80;
        listen [::]:80;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
        }
    }
```
### How we create / use Secrets

- we should use when we passing sensitive data like 
    - passwords 
    - tokens 
    - keys (ssh-keys)

```yaml
apiVersion: v1
kind: Secret
metadata: 
  name: redis-secret 
  labels: 
    app: mysql
type: Opaque
data:
  DB_USER: azDdfdehl224==
  DB_PASSWORD: de2fesd8aW==
```

### full example with ConfigMaps 

#### Key-Value

```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: mysql-pod
  labels: 
    apps: mysql
spec:
  containers:
  - name: mysql-ctr
    image: mysql:5.7
    env:
    - name: MYSQL_RANDOM_ROOT_PASSWORD
      valueFrom:
        configMapKeyRef:
          name: mysql-config
          key: ROOT_RANDOM_PASSWORD
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data: 
  ROOT_RANDOM_PASSWORD: 1
```

#### Volumes

```yaml
spec:
  containers: 
  - name: webapp-ctr
    image: httpd:2.4.47
    ports:
    - containerPort: 80

volumes:
  - name: nginx-conf
    configMap:
      name: nginx-conf
      items: 
        - key: nginx.conf
          path: default.conf
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata: 
  name: nginx-conf
data: 
  nginx.conf: |
    server {
        listen 80;
        listen [::]:80;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
        }
    }
```
In above example we injecting the content defined in ConfigMap yaml under `data` as `nginx.conf` to configuration file `default.conf`

and then we would need to mount the volume to the container: 
```yaml
spec:
  containers:
  - name: webapp-ctr
    image: httpd:2.4.47
    ports:
    - containerPort: 80
    volumeMounts: 
    - mountPath: "/etc/nginx/conf.d"
      name: nginx-conf
      readOnly: true
  volumes: 
  - name: nginx-conf
    configMap: 
      name: nginx-conf
      items: 
        - key: nginx.conf
          path: default.conf
```

we mounted this this way 
```yaml
volumeMounts: 
    - mountPath: "/etc/nginx/conf.d"
      name: nginx-conf
      readOnly: true
```

### Practical 

```yaml 
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: name-map
data:
  given: auggie
  family: chmiel
---  
apiVersion: v1
kind: Pod
metadata:
  labels:
    house: configmaps
  name: familypod
spec:
  containers:
    - name: house
      image: busybox
      command: [ "/bin/sh", "-c", "watch -n 10 echo First name: $(FIRSTNAME) last name: $(LASTNAME)" ]
      env:
        - name: FIRSTNAME
          valueFrom:
            configMapKeyRef:
              name: name-map
              key: given
        - name: LASTNAME
          valueFrom:
            configMapKeyRef:
              name: name-map
              key: family
```
```she
kubectl apply -f mytests/house.yaml
kubectl logs familypod
```
```Output
Every 10.0s: echo First name: auggie last name: chmiel      2023-02-16 18:36:00

First name: auggie last name: chmiel
```

Next example is more complex and wil show how to setup nginx

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf #the name cannot contain camelcase
data:
  nginx.conf: |
    server {
      listen       80;
      listen  [::]:80;
      server_name  localhost;

      #charset koi8-r;
      #access_log  /var/log/nginx/host.access.log  main;

      location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
      }

      #error_page  404              /404.html;

      # redirect server error pages to the static page /50x.html
      #
      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
          root   /usr/share/nginx/html;
      }

      # proxy the PHP scripts to Apache listening on 127.0.0.1:80
      #
      #location ~ \.php$ {
      #    proxy_pass   http://127.0.0.1;
      #}

      # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
      #
      #location ~ \.php$ {
      #    root           html;
      #    fastcgi_pass   127.0.0.1:9000;
      #    fastcgi_index  index.php;
      #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
      #    include        fastcgi_params;
      #}

      # deny access to .htaccess files, if Apache's document root
      # concurs with nginx's one
      #
      #location ~ /\.ht {
      #    deny  all;
      #}
    }
```
```sh
❯ kubectl apply -f nginx-config-map.yml
configmap/nginx-conf created
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: html-conf
data:
  html.conf: |
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="utf-8">
        <title>Custom Nginx HTML page</title>
      </head>
      <body>
        <p> This is content from Config Map (html.conf)
      </body>
    </html>
```
```sh
❯ kubectl apply -f nginx-html-config-map.yml
configmap/html-conf created
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - mountPath: /etc/nginx/conf.d # mount nginx-conf volume to /etc/nginx/conf.d
      readOnly: true
      name: nginx-conf
    - mountPath: /usr/share/nginx/html/
      readOnly: true
      name: html-conf
  volumes:
  - name: nginx-conf
    configMap:
      name: nginx-conf # place ConfigMap `nginx-conf` on /etc/nginx/conf.d/default.conf file
      items:
        - key: nginx.conf
          path: default.conf
  - name: html-conf
    configMap:
      name: html-conf 
      items:
        - key: html.conf
          path: index.html
```
```
❯ kubectl apply -f nginx-pod.yml
pod/nginx created
```
on this page  http://localhost:8001/api/v1/namespaces/default/pods/http:nginx:/proxy/
after running `kubectl proxy` we should see see the text in configMap `This is content from Config Map (html.conf)`

---

### Practical Secret

secrets in K8's are stored as base64 
```sh
echo -n 'auggieMaestro' | base64
echo -n 'YXVnZ2llTWFlc3Rybw==' | base64 --decode
YXVnZ2llTWFlc3Rybw==
auggieMaestro

```

- Commands to apply secrets settings passed via environment variables
```sh
kubectl create -f mysql-secret.yml
kubectl apply -f mysql-pod.yml
kubectl get pod
kubectl logs mysql-pod
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  labels:
    app: mysql
type: Opaque
data:
  DB_USER: azhzbWFlc3Rybw==
  DB_PASSWORD: dmVyeVNlY3VyZQo=
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  labels:
    app: mysql
spec:
  containers:
  - name: mysql-ctr
    image: mysql:5.7
    env:
    - name: MYSQL_USER
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: DB_USER
    - name: MYSQL_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret  
          key: DB_PASSWORD
    - name: MYSQL_RANDOM_ROOT_PASSWORD
      value: "1"
```
- Commands to pass secrets via volumes

```sh
kubectl create -f webapp-secret.yml
kubectl create -f webapp-pod.yml
kubectl exec -it webapp-pod -- bash
cd /etc/webapp
ls -l
cat username
cat ./secure/password
exit
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secret
  labels:
    app: web-app
type: Opaque
data:
  username: azhzbWFlc3Rybw==
  password: dmVyeVNlY3VyZQo=
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-pod
spec:
  containers:
  - name: webapp-ctr
    image: httpd:2.4.47
    ports:
    - containerPort: 80
    volumeMounts:
    - mountPath: "/etc/webapp"
      name: webapp-secret-vol
      readOnly: true
  volumes:
  - name: webapp-secret-vol
    secret: 
      secretName: webapp-secret
      defaultMode: 0600 # if you don't specify it, 0644 will be used.
      items:
      - key: username
        path: username
      - key: password
        path: secure/password
        mode: 0400 #overriding defaultMode
```

```bash
❯ kubectl exec -it webapp-pod -- bash
root@webapp-pod:/usr/local/apache2# cd /etc/webapp
root@webapp-pod:/etc/webapp# ls -l
total 0
lrwxrwxrwx 1 root root 13 Feb 16 19:14 secure -> ..data/secure
lrwxrwxrwx 1 root root 15 Feb 16 19:14 username -> ..data/username
root@webapp-pod:/etc/webapp# cat username
k8smaestro
root@webapp-pod:/etc/webapp# cat ./secure/password
verySecure
root@webapp-pod:/usr/local/apache2# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay         348G   33G  315G  10% /
tmpfs            64M     0   64M   0% /dev
/dev/nvme0n1p3  348G   33G  315G  10% /etc/hosts
tmpfs           126G  8.0K  126G   1% /etc/webapp
shm              64M     0   64M   0% /dev/shm
tmpfs           126G   12K  126G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs            63G     0   63G   0% /proc/acpi
tmpfs            63G     0   63G   0% /proc/scsi
tmpfs            63G     0   63G   0% /sys/firmware
root@webapp-pod:/usr/local/apache2# 
root@webapp-pod:/etc/webapp# exit
exit
```
`tmpfs           126G  8.0K  126G   1% /etc/webapp`

### Job

- Job is creating one or more pods all depends on needs
- Job are used for long running tasks, which are going to execute one or more times
- Usage
    - executing long and complex computing tasks
    - analytics / data processing
    - image compression / image rendering
    - ...
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  suspend: true
  parallelism: 1
  completions: 5
  template:
    spec:
      ...
```
- if container within one pod wil return error as many times as we specified in `backOffLimit`, Job will be marked as failed
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  suspend: true
  backOffLimit: 4
  completions: 5
  template:
    spec:
      ...
```
- above limit is per one pod, not for all pods within a Job
- this means that each of the five Pods can fail three times and the Job can still be successfully completed

### Practical Job

```sh
kubectl apply -f job-km.yml
kubectl get job/job-km
kubectl logs -f -l job-name=job-km
kubectl describe jobs/job-km
```
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-km
  labels:
    jobgroup: k8smaestro
spec:
  parallelism: 1 # only one replica of the job will be created
  completions: 1 # how many job replicas need to be completed to mark job as completed
  activeDeadlineSeconds: 1800 # maximum duration the job can run
  template:
    spec:
      containers:
      - name: job-ctr
        image: k8smaestro/job-demo
        args: ["15"]
      restartPolicy: Never ## optionally - OnFailure (will restart the container in the pod), or Always (always restart container), but recommended option is Never. 
  backoffLimit: 4 ## number of retries for a job
```
other resources which will simulate the computing Job they are in this image `image: k8smaestro/job-demo`
```dockerfile
FROM ubuntu:18.04
COPY script.sh /script.sh 
RUN chmod +x script.sh
ENTRYPOINT ["/script.sh"]
```
```sh
#! /bin/bash

LOOP_COUNT=$1
echo "This Kubernetes Job will echo message $1 times"

for ((i=1;i<=$LOOP_COUNT;i++)); 
do
   sleep 2
   echo $i] Hey K8s Maestro! I will run till the job completes.
done
```
```sh
❯ kubectl get job/job-km
NAME     COMPLETIONS   DURATION   AGE
job-km   0/1           16s        16s
❯ kubectl logs -f -l job-name=job-km
1] Hey K8s Maestro! I will run till the job completes.
2] Hey K8s Maestro! I will run till the job completes.
3] Hey K8s Maestro! I will run till the job completes.
4] Hey K8s Maestro! I will run till the job completes.
5] Hey K8s Maestro! I will run till the job completes.
6] Hey K8s Maestro! I will run till the job completes.
7] Hey K8s Maestro! I will run till the job completes.
8] Hey K8s Maestro! I will run till the job completes.
9] Hey K8s Maestro! I will run till the job completes.
10] Hey K8s Maestro! I will run till the job completes.
11] Hey K8s Maestro! I will run till the job completes.
12] Hey K8s Maestro! I will run till the job completes.
13] Hey K8s Maestro! I will run till the job completes.
14] Hey K8s Maestro! I will run till the job completes.
15] Hey K8s Maestro! I will run till the job completes.
❯ kubectl get job/job-km
NAME     COMPLETIONS   DURATION   AGE
job-km   1/1           38s        42s
```

### CronJobs

- objects in K8's which is designed to perform cyclical tasks (on every specified interval)
- Cron - tool which comes from Linux environment and in serving same purpose 
- Cron format as example: (`*/5 * * * *`) - which mean every 5 mins 
```shell
“At 04:05.”
next at 04:05:00 other day
5       4     *      *       *
minute  hour  day    month   day
             (month)         (week)

*	        any value
,	        value list separator
-	        range of values
/	        step values
@yearly	    (non-standard)
@annually	(non-standard)
@monthly	(non-standard)
@weekly	    (non-standard)
@daily	    (non-standard)
@hourly	    (non-standard)
@reboot	    (non-standard)
```
```
min           (0 - 59)
hour          (0 - 23)
day of month  (1 - 31)
month         (1 - 12)
day of week   (0 - 6) (0 to 6 are Sunday to Saturday, or use names; 7 is also Sunday)
```

### CronJob in K8's example
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata: 
  name: cronjob-km 
spec: 
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cronjob-ctr
            image: k8smaestro/email-sender
          restartPolicy: OnFailure
```
### K8's CronJobs - what to keep in mind

1. concurrency - specify whether if one job is not done yet, the other should start (default is allowed) `.spec.concurrencyPolicy: Allow`
2. if the Job failed (for some reason) - we can specify how much maximum time can elapse to try to run it again `.spec.startingDeadlineSeconds`

Example: when `startingDeadlineSecond: 60`, the Job can execute **ONLY** within 60 seconds of its initially scheduled execution

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-km
spec:
  schedule: "*/1 * * * *" ## every 1 minute
  # Allow, Forbid, Replace
  concurrencyPolicy: "Replace" # if previous job has not finished, it will be canceled and a new one will start.
  suspend: false            
  successfulJobsHistoryLimit: 3 
  failedJobsHistoryLimit: 1     
  jobTemplate:             
    spec:
      template:
        metadata:
          labels:          
            parent: "cronjobpi"
        spec:
          containers:
          - name: cronjob-ctr
            image: perl
            command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(1000)"]
          restartPolicy: OnFailure 
```
```sh
kubectl apply -f cronjob-km.yml
kubectl get cj
kubectl describe cj cronjob-km
kubectl get pods
kubectl logs <POD_name_from_DESCRIBE>
```
---

### Configuration - good practices: Secrets

- Not necessarily secure - base64 encoding https://kubernetes.io/docs/concepts/configuration/secret/#risks
- Options worth considering (these are only options, not recommendations!)
  - HashiCorp Vault integration with Kubernetes via SideCar
    - https://github.com/hashicorp/vault-k8s
    - https://www.youtube.com/watch?v=xUuJhgDbUJQ
  - External Secrets 
    - https://github.com/external-secrets/kubernetes-external-secrets
  - Sealed Secrets 
    - https://github.com/bitnami-labs/sealed-secrets/
  - Recommended tools by ArgoCD 
    - https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/

---