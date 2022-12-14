# k8s-notes
k8s notes with basic info and commands
information learnt/gotten from books and pluralsight courses will be added here.

# **Kubernetes**
Multi Master H/A Control Plane
H/A = Highly Available

## **Masters:**
1 is better than 2

3 are better than 1

5 are better than 3 if looking for resilence

5+ can become messy.

Run User/business apps in NODES

### **kube-apiserver**
- Front-end to the control plane
- Exposes the API (REST)
- Consumes JSON/YAML

### **Cluster Store**
- Only persistant component in the entire control plane
- Persists cluster state and config
- Based on etcd 
- Performance is critical

### **Kube-controller-manager**
- Controller of controllers:
  - Node controller
  - Deployment controller
  - Endpoints/EndpointSlice controller
- Watch loops
- Reconciles observed state with desired state

### **Kube-scheduler**
- Watches API Server for new work tasks
- Assigns work to cluster nodes
  - Affinity/Anti-affinity
  - Constraints
  -   Taints
  - Resources...

## **Nodes**:

### **Kubelet**
- Main Kubernetes agent
- Registers node with clusters
- Watches API server for work tasks (Pods)
- Executes Pods
- Reports back to Masters

### **Container runtime**
- Can be Docker
- Pluggable: Container Runtime Interface (CRI)
- Docker, containerd, CRI-O, Kata... (Notes: Google gVisor / katacontainers)
- Low-level container intelligence

### **Kube-proxy**
- Networking component
- Pod IP addresses
- Basic load-balancing

A node is an environment which holds Pods, it is to a pod what it is to an app.

An abstraction level higher than a Pod with the similar purpose of holding and grouping apps, but Pods in this case.

## **Declarative Model and Desired State**:
### **Declarative model**
Describe what you want (desired state) in a manifest file

specs.yaml
```yaml
kind: Kitchen
spec:
    type: New
    location: Rear
    style: OpenPlan
    heating:
        type: underfloor
        medium: Water
    windows:
        type: FloorToCeiling
        aspect: South
    doors:
        type: FireDoor
        accessTo: Garage
    island: Yes
    roofGarden: Yes  
```

_k8s-manifest.spec.yaml_
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: test
spec:
    replicas: 6
    selector:
        matchLabels:
            app: ps-test
    template:
        spec:
            containers:
                - name: c1
                  image: web1:1.3
                  ports: 8080
```

## **Pods**
_Every pod is an execution env similar to a single OS or user session which can run multiple apps at the same time. Therefore these apps share the same IP. For handling use cases for the same IP they must have unique ports, just like a single computer instance running  multiple apps, with the same ip and port they will collide and 1 or the other won't execute._


### **Scaling**
_You scale by adding new pods with the apps within them, NOT by adding more instances of the app to an existing pod with it_

Containers in a pod are always scheduled to the same node

Pods are atomically inmortal, they do self-heal

### **Phases**
- Pending
- Running
- Succeeded/Failed


## **Kubernetes service object**
Stable name and IP, and DNS used by the load balancing

The way that a Pod belongs to a service or makes it onto the list of Pods that a service will forward traffic to is via Labels

_LABELS ARE THE SIMPLEST AND YET VERY VERY POWERFULL COMPONENT IN K8S_

### **Facts**
- Services only sends traffic to healthy pods
- Can do session affinity
- Can send traffic to endpoints outside the cluster
- Cand do tcp and udp


## **Game Changing Deployments**
Zero downtime rolling updates

Manages either updates and rollbacks

Replica count, self healing and previous versions

## **Working with Pods**
**apiVersion: v1** \
**v1alpha1 > v1beta2 > v1 (GA/Stable Prod ready)**

livenessProbe (is it alive) \
readinessProbe (is it ready to receive traffic)

### **Imperative Way**
```sh
kubectl run [podname] --image=nginx:alpine 
# Enable pod container to be called externally
kubectl port-forward [name-of-pod] 8080:80 

kubectl delete pod [name-of-pod]
# Delete deployment that manages the Pod
kubectl delete deployment [name-of-deployment]
```
### **Declarative Way**
```sh
kubectl apply -f <manifest-file-path>.yml 
kubectl apply -f ./Pods/pod.yml \
kubectl apply -f ./Pods/multi-pod.yml
# Validate creation/configuration output without
kubectl apply -f nginx.pod.yml --dry-run=client 
kubectl get pod my-nginx -o yaml

kubectl get pods --watch 
kubectl get pods --show-labels 
kubectl get pods -o wide # More information 
kubectl describe pods/<pod-name> 
kubectl describe pods <pod-name> 
kubectl delete -f \<manifest-file-path>.yml
```

_pod.yml_
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: hello-pod
  labels:
    app: web
spec:
  containers:
    - name: web-ctr
      image: sunfiremohawk/getting-started-k8s:1.0
      ports:
        - containerPort: 8080
```

_multi-pod.yml_
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: main-ctr
    imag e: nigelpoulton/nginxadapter:1.0
    ports:
    - containerPort: 80
  - name: helper-ctr
    image: nginx/nginx-prometheus-exporter
    args: ["-nginx.scrape-uri","http://localhost/nginx_status"]
    ports:
    - containerPort: 9113
```
## **Kubernetes Services**:
Services will be the DNS to export to the outside ports under a certain NAME and low-key load balancing the requests to the underlying Pods routed to the service

Pods have IPs assigned by K8s \
Services have a NAME created by CoreDNS which resolves to the Pods matched to the label
### **Facts**
- ClusterIP for internal access
- NodePort for external access
- LoadBalancer seemingless integrate with the cloud provider native provider to provide access to the Internet

- NodePorts are between teh range of 30000 to 32767. 
- ClusterIPs are default

### **Imperative Way _(not the k8s way)_**

```
kubectl expose pod <pod-name> --name=<name-svc> \ --target-port=8080 --type=NodePort 
kubectl expose pod hello-pod --name=hello-svc  \ --target-port=8080 --type=NodePort
```

```
kubectl delete svc <name-svc> 
kubectl delete svc hello-svc 
```

### **Declarative Way _(the k8s way)_**

kubectl apply -f ./Services/svc-nodeport.yml

_svc-nodeport.yml_
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ps-nodeport #dns-name
spec:
  type: NodePort # ClusterIP (default) / NodePort / LoadBalancer
  ports:
  - port: 80 # Port the service listens INSIDE THE CLUSTER. 
#                   Therefore. Any app looking to connect to ps-nodeport uses internally port 80 to connect to it.
    targetPort: 8080 # internal cluster port
    nodePort: 31111 # port mapped to external connectivity
    protocol: TCP
  selector:
    app: web # Pod resolving the connection
```
_svc-lb.yml_
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ps-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: web
```

_external-example.service.yml_
```yml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: api.acmecorp.com
  ports:
  - port: 9000
```

## **K8s Storage**
_Pods live and die so their file system is short-lived (epehemeral)_ \
_Volumes can be used to starte state/data and use it in a Pod_ \
_A Pod can have multiple Volumes attached to it_ \
_Containers rely on a **mountPath** to access a Volume_


### **Kubernetes Supports**
- Volumes
- PersistentVolumes
- PersistentVolumeClaims
- StorageClasses

_Volumes in containers have a similar purpose within the Kubernetes context, although, K8s Volumes have more options_

#### **Volumes**
- A Volume references a storage location.
- Must have a unique name.
- Attached to a Pod and may or may not be tied to the Pod's lifetime (depending on the Volume type).
- A Volume Mount references a Volume by name and defines a mountPath.

#### **Volumes Type**
- emptyDir
  - Shares a Pod's lifetime.
  - Empty directory for storing "transient" data.
  - Useful for sharing files between containers running in a Pod.
- hostPath
  - Pod mounts into the node's filesystem.
  - Tied to Node lifetime.
  - Possible lost of data when the Node goes down.
- nfs (Network File System)
  - Share mounted within the Pod
- configMap/secret
  - Special types of volumes
  - Provide Pods access to K8s resources
- persistentVolumeClaim
  - A far more resilence storage option for Pods
  - Abstracted from details
- Cloud
  - Cluster-wide storage
- Others
  - awsElasticBlockStore, azureDisk, azureFile, cephfs, csi, downwardAPI, fc, flexVolume, flocker, gcePersistentDisk, glusterfs, iscsi, local, projected, portworxVolume, quobyte, rbd, scaleIO, storageos, vsphereVolume

### **emptyDir**
_As long as the pod is alive the data will be available. If the pod goes down, the data created from the pod will be lost_

```yaml
apiVersion: v1
kind: Pod
spec:
  volumes:
  - name: html
    emptyDir: {}
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
      - name: html
        mountPath: /usr/share/nginx/html
        readOnly: true
  - name: html-updater
    image: alpine
    command: ["/bin/sh", "-c"]
    arg:
      - while true; do date >> /html/index.html;
        sleep 10; done
    volumeMounts:
      - name: html
        mountPath: /html
```

### **hostPath**
- Basic setup
- Very easy to set up
- If single worker node it provides and easy way to get started with volumes.
- Data can be lost if the worker node goes down.

```yaml
apiVersion: v1
kind: Pod
spec: 
  volumes:
    - name: docker-socket
      hostPath:
        path: /var/run/docker.sock
        type: Socket
  containers:
  - name: docker
    image: docker
    command: ["sleep"]
    args: ["100000"]
    volumeMounts:
      - name: docker-socket
        mountPath: /var/run/docker.sock
```

### **Cloud Volumes**
- Azure - Azure Disk and Azure FIle
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  volumes:
  - name: data
    azureFile:
      secretName: <azure-secret>
      shareName: <share-name>
      readOnly: false
  containers:
  - image: someimage
    name: my-app
    volumeMounts:
    - name: data
      mountPath: /data/storage
```
- AWS - Elastic Block Store
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
sepc:
  volumes:
  - name: data
    awsElasticBlockStore:
      volumeID: <volume_ID>
      fsType: ext4
  containers:
  - image: someimage
    name: my-app
    volumeMounts:
    - name: data
      mountPath: /data/storage
```
- GCP - GCE Persitent Disk
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  volumes:
  - name: data
    gcePersistentDisk:
      pdName: datastorage
      fsType: ext4
  containers:
  - image: someimage
    name: my-app
    volumeMounts:
    - name: data
      mountPath: /data/storage
```

### **Persistent Volume**
_Is a cluster-wide storage unit provisioned by an administrator with a lifecycle independent from a Pod._

- It could talk to cloud storage
- It could talk to local storage
- Is used in conjunction of PersistentVolumeClaim
- Cluster-wide
- Relies on network-attached-storage (_NAS_)
- Usually provisioned by a cluster administrator
- Available to a Pod even if it gets rescheduled to a different Node

### **Persistent Volume Claim (PVC)**
_Is a request for a storage unit (Persitent Volume)_

### **Viewing a Pod's Volumes**
```sh
kubectl describe pod [pod-name]
kubectl get pod [pod-name] -o yaml
```


## **Common commands**
```sh
kubectl delete -f <manifest-file-path>.yml 
kubectl apply -f ./Services/svc-lb.yml 
kubectl apply -f ./Services/svc-nodeport.yml
kubectl delete -f ./Services/svc-nodeport.yml 
kubectl delete -f ./Services/svc-lb.yml 
kubectl delete svc ps-nodeport
```

## **Kubernetes Deployments**
### **Commands**
```sh
kubectl get deployments 
kubectl get deploy
kubectl get deployment --show-labels 
kubectl get deployment --all-namespaces 
kubectl get deploy -l app=my-nginx 
kubectl delete deploy [deploy-name] 
kubectl scale deploy [deploy-name] --replicas=5 
kubectl scale -f my-nginx.deploy.yaml --replicas=5
```

## **Best Practices**
### **Create Pods with Readiness, LivenessProbes and Resource Limits**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
  labels:
    app: nginx
    rel: stable
spec:
  containers:
  - name: my-nginx
    image: nginx:alpine
    resources:
      limits:
        memory: "128Mi" #128 MB
        cpu: "200m" #200 millicpu (.2 cpu or 20% of the cpu)
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /index.html
        port: 80
      initialDelaySeconds: 15
      timeoutSeconds: 2 # Default is 1
      periodSeconds: 5 # Default is 10
      failureThreshold: 1 # Default is 3
    readinessProbe:
      httpGet:
        path: /index.html
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5 # Default is 10
      failureThreshold: 1 # Default is 3
```

### **Create Deployments with roollingUpdate strategy, imagePullPolicy**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
  labels:
    app: web
spec:
  selector:
    matchLabels:
      app: web
  replicas: 5
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: web
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - name: hello-pod
        image: sunfiremohawk/getting-started-k8s:2.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```
## **Kubectl commands**
```sh
kubectl version 
kubectl cluster-info 
kubectl get all 
kubectl run [container-name] --image=[image-name] 
kubectl port-forward [pod] [ports]
```
