# kubectl-cheat-sheet

## Node
````shell script
kubectl get nodes {node-name} --show-labels 
kubectl describe node {node-name}
kubectl get nodes {node-name} -o yaml | grep -B {NUMBER_OF_LINES_BEVOR_KEYWORD} -A {NUMBER_OF_LINES_AFTER_KEYWORD} {KEYWORD}
````

## Namespace
````shell script
kubectl create namespace {namespace-name}
kubectl config set-context --current --namespace={namespace-name}
kubectl config view --minify | grep namespace:                      # Get current namespace
````

## Label and Selectors
````shell script
kubectl label nodes {node-name} {LABEL_KEY}={LABEL_VALUE}
kubectl label pods {pod-name} {LABEL_KEY}={LABEL_VALUE}

kubectl label --overwrite {pod-name} {LABEL_KEY}={LABEL_VALUE}

kubectl get pods --selector {LABEL_KEY}={LABEL_VALUE}
````

Example to show how Labels are used:
````yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:                                               # Label of Replicaset
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:                                             # Selector to bind Replicaset to Pods
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:                                           # Label of Pod
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
````


## Pod
````shell script
kubectl run {pod-name} 
    --image={image-name} 
    --replicas={number-of-replicas}
    --env="key1=value1" --env="key2=value2"
    --port={port-of-container}
    --labels="env=prod,app=webserver"
    --annotations="annotation=value,annotation2=value2" # TODO: Check if this works like that!
    --command -- {command} {arg1} {arg2} {arg3} ...     
    -- {arg1} {arg2} {arg3} ...                         # Use default command but different args!
    --dry-run=client -o yaml > my-pod.yaml              # Save as yaml
````

### Edit a Pod
You can just edit following specs of a pod:
* spec.containers[*].image
* spec.initContainers[*].image
* spec.activeDeadlineSeconds
* spec.tolerations
 ````shell script
kubectl edit pod {pod-name} --image={new-image}
````
Else you have to export the actual manifest, edit it and then recreate it:
`````shell script
kubectl get pod {pod-name} -o yaml > my-pod-to-edit.yaml
vim my-pod-to-edit.yaml
kubectl delete pod {pod-name}
kubectl apply -f my-pod-to-edit.yaml
`````

### Manifest
````yaml
kind: Pod
apiVersion: v1
metadata:
  name: myPod
  namespace: mynamespace
  annotations:
  labels:
    key: value
    color: blue
spec:
  serviceAccount: {serviceaccount-name} # If nothing is specified, it received automatically the default-ServiceAccount!
  automountServiceAccountToken: false   # If you don't want to map the default ServiceAccount.
  securityContext:
      securityContext:                  # SecurityContext on Pod-Level (Affect all Containers)
          runAsUser: 2000
  imagePullSecrets:
    - name: {NAME_OF_DOCKERREGISTRY_SECRET} # Using private Container-Registry
  tolerations:
    - key: "app"
      operator: "Equal"                 # Equal (default) | Exists (just the key)
      value: "blue"
      effect: "NoSchedule"              # NoSchedule|PreferNoSchedule|NoExecute
  nodeSelector:
      size: Large                       # Label of a Node
  affinity:
      nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: size
                  operator: In
                  values:
                    - Large
                    - Medium
  volumes:
    - name: data-volume                 # A Volume created by a Directory on a node
      hostPath:
          path: /data                   # Path on the Node
          type: Direcoty
  containers:
    - name: webserver
      image: nginx
      ports:
        - containerPort: 80
      securityContext:                  # SecurityContext on Container-Level
          runAsUser: 1000               # ID of User (Root has ID 0) (Overrides values on Pod-Level!)
          privileged: true
          capabilities:                 # Only supported on Container-Level!!!!
              add: ["MAC_ADMIN", "CHOWN", "SYS_ADMIN"]
      volumeMounts:                     # Mount a volume into the container
        - mountPath: /opt               # Path to the Directory where the volume should be mounted 
          name: data-volume
      env:
        - name: APP_COLOR
          value: pink
        - name: APP_COLOR
          valueFrom:
              configMapKeyRef:
                  name: {NAME_OF_CONFIGMAP}
                  key: {KEY_OF_SPECIFIC_VALUE_IN_CONFIGMAP}
        - name: APP_COLOR
          valueFrom:
              secretKeyRef:
                  name: {NAME_OF_SECRET}
                  key: {KEY_OF_SPECIFIC_VALUE_IN_SECRET}
      envFrom:                           
          - configMapRef:                 # Bind all ENVs of Configmap to Pod!
              name: {NAME_OF_CONFIGMAP}                    
          - secretRef:                    # Bind all ENVs of Secret to Pod!
              name: {NAME_OF_CONFIGMAP}
      command: ["sh"]                     # ENTRYPOINT in Docker
      args: ["-c", "echo 'Hello World'"]  # CMD in Docker
      resources:
          requests:
              memory: "1Gi"               # 1Gi = 1'073'741'824 Bytes, 1G = 1'000'000'000 Bytes
              cpu: 1                      # 1Mi = 1'048'576 Bytes, 1M = 1'000'000 Bytes
          limits:                         # 1 CPU = 1 HyperThread, 1 AWS vCPU, 1 Azure Core
              memory: "2Gi"               # Just terminates the container if it goes over the limit
              cpu: 2                      # Throttle the container if it goes over the limit
````

## Persistent Volumes
Access Modes:
* ReadWriteOnce: enables read and write and can be mounted by only one node
* ReadOnlyMany: enables read only and can be mounted by multiple nodes (but not at the same time)
* ReadWriteMany: both read and write, can be mounted by several nodes (not at the same time)

````yaml
apiVersion: v1
kind: PersistentVolume
metadata:
    name: my-volume
spec:
    accessModes:
      - ReadWriteOnce
    capacity:
      storage: 1Gi
    awsElasticBlockStore:
      volumeID: <volumeId>
      fsType: ext4
````


## Persistent Volume Claim

## ReplicationController
### Manifest
````yaml
apiVersion: v1
kind: ReplicationController
metadata: 
  name: nginx-rc
spec:
  replicas: 3
  selector:
    app: nginx
    tier: dev
  template: 
    metadata:
      name: nginx
      labels: 
        app: nginx
        tier: dev
    spec:
      containers:
      - name: nginx-container
        image: nginx
        ports:
        - containerPort: 80
````

## ReplicaSet
Difference between ReplicaSet and ReplicationController:
* ReplicationController supports equality based selectors
* ReplicaSet supports equality based and set based selectors


Equality Based selectors:
* Must satisfy all the specified label constraints
* supported operators: =, ==, !=

Set Based selectors:
* Allow filtering keys according to a set of values
* supported operators: IN, NOTIN, EXISTS (only Key identifier)
### Manifest
````yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata: 
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      env: prod
    matchExpressions:
    - { key: tier, operator: In, values: [frontend] }
  template:
    metadata:
      name: nginx
      labels: 
        env: prod
        tier: frontend
    spec:
      containers:
      - name: nginx-container
        image: nginx
        ports:
        - containerPort: 80
````

## Deployment
````shell script
kubectl create deployment {deployment-name} 
    --image={IMAGE_NAME} 
    --port={PORT_OF_CONTAINER} 
    --replicas={NUMBER_OF_REPLICAS}
    --dry-run=client -o yaml > my_deployment.yaml

kubectl create -f my_deployment_file.yml --record       # Record the Deployment to history!
````
### Manifest
````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate         # Recreate|RollingUpdate
    rollingUpdate:
      maxUnavailable: 2         # Max Number/Percent of Pods which can be unavailable during Upgrade/Rollout
      maxSurge: 2               # Max Number/Percent of Pods that can be created over the desired number of Pods
  template:                     # POD-Manifest
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
````

### Rolling Updates & Rollbacks
**Strategytypes**:
* **Recreate**: Terminates all Pods and Recreates them. The problem is, that the application will contain a downtime.
* **RollingUpdate**: Terminates and updates every Pod one by one. Your Application won't have any downtime.
````shell script
# Rollout changes with manifest
kubectl apply -f manifest_with_changes.yaml

# Update the image
kubectl set image deployment/{deployment-name} {container-name}={new-image}

# Get Status/History of Rollout
kubectl rollout status deployment/{deployment-name}
kubectl rollout history deployment/{deployment-name}

# Rollback a deployment
kubectl rollout undo deployment/{deployment-name}

# Edit Rolloutstrategy of Deployment
kubectl edit deployment {deployment-name} 
````
## Service
```shell script
kubectl create service nodeport {service-name}
    --tcp={port}:{target-port}
    --node-port={node-port}

kubectl create service clusterip {service-name}
    --tcp={port}:{target-port}

kubectl create service loadbalancer {service-name}
    --tcp={port}:{target-port}

# Create Service directly from Pod
kubectl expose pod {pod-name} 
    --name={service-name} 
    --port={port-of-service} 
    --target-port={port-of-container}
    --type={"ClusterIP"|"NodePort"|"LoadBalancer"|"ExternalName"}

# Create Service directly from Deployment. Default type is "ClusterIP"
kubectl expose deployment {deployment-name} 
    --name={service-name} 
    --type={"ClusterIP"|"NodePort"|"LoadBalancer"|"ExternalName"}
      
```

### Manifest
NodePort-Service
````yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
      nodePort: 30009
````

ClusterIP-Service
````yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
````

LoadBalancer-Service
````yaml

````

## Ingress
````shell script
kubectl create ingress {ingress-name} 
    --rule={host-name}/{path}={service-name}:{port-of-service}[,tls={my-cert-name}]
    --annotation {my-annotation}={value}
````

### Manifest
````yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wildcard-host
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /   # Rewrites the URL. Example: /mypath --> /
spec:
  rules:
  - host: "foo.bar.com"
    http:
      paths:
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: "*.foo.com"
    http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: service2
            port:
              number: 80
````

## NetworkPolicies
````shell script

````
### Manifest
````yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:                      # Select on which Pods the NetworkPolicy affects!
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:                          # Incoming traffic, you can set multiple from/to fields!
  - from:                           # Rules about what kind of Pods / Servers are allowed to access this pod
    - ipBlock:                      # IP Block Rule
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:            # Namespace Rule
        matchLabels:
          project: myproject
    - podSelector:                  # Pod Rule
        matchLabels:
          role: frontend
    - podSelector:                  # Combined Rules
        matchLabels:
          role: backend
      namespaceSelector:
        matchLabels:
          project: myproject
    ports:                          # Allowed Port to receive traffic on the affected Pod
    - protocol: TCP
      port: 6379
  egress:                           # Outgoing traffic
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
````

## ConfigMaps
````shell script
kubectl create configmap {configmap-name} --from-literal=KEY=VALUE --from-literal=KEY2=VALUE2 ...
kubectl create configmap {configmap-name} --from-file={PATH_OF_FILE}
````

### Manifest
````yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: NAME
  namespace: NAMESPACE
data:
  KEY1: VALUE1
  KEY2: VALUE2
````

## Secrets
````shell script
kubectl create secret generic {secret-name} --from-literal=key1=value1 --from-literal=key2=value2
kubectl create secret generic {secret-name} --from-file={PATH_TO_FILE}
kubectl create secret docker-registry regcred \ 
    --docker-server=<your-registry-server> \
    --docker-username=<your-name> \
    --docker-password=<your-pword> \
    --docker-email=<your-email>
````
When you use from literal, you don't have to encode the values. If you use manifests, you have to encode the values!
#### Encode
````shell script
echo -n 'mypassword' | base64
bXlwYXNzd29yZA==
````

#### Decode
```shell script
echo -n 'bXlwYXNzd29yZA==' | base64 -d
mypassword
```
### Manifest
````yaml
apiVersion: v1
kind: Secret
metadata:
  name: NAME
  namespace: NAMESPACE
data:
  username: BASE64_ENCODED_USERNAME
  password: BASE64_ENCODED_PASSWORD
type: Opaque
````
Pls... don't use secrets in your repo like that. Use instead Sealed Secrets: https://github.com/bitnami-labs/sealed-secrets

## ServiceAccount
When you create a ServiceAccount, a Token is created as a Secret and mapped to this ServiceAccount.
When you create a Pod and no ServiceAccount is mapped, Kubernetes maps automatically the default-ServiceAccount as a Volume under /var/run/secrets/kubernetes.io/serviceaccount!
````shell script
kubectl get serviceaccount
kubectl create serviceaccount {serviceaccount-name}
````

## LimitRange
Set default Memory Requests and Limits for a Namespace (per container)
````yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: {limitrange-name}
  namespace: {namespace-name}
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 1
    defaultRequest:
      memory: 256Mi
      cpu: 0.5
    type: Container
````

Set min and max Memory and CPU Requests for a Namespace (per container)
````yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-min-max-demo-lr
spec:
  limits:
  - max:
      memory: 1Gi
      cpu: "800m"
    min:
      memory: 500Mi
      cpu: "200m"
    type: Container
````

## Taint
`````shell script
kubectl taint nodes {node-name} key=value:{taint-effect}  # NoSchedule|PreferNoSchedule|NoExecute
kubectl describe node {node-name} | grep Taint
`````

## NodeSelector and NodeAffinity
Define on which node the pod should be scheduled!
You can achieve this with "nodeSelector" (setting label of node) or with NodeAffinity (multiple Rules for labels).
````yaml
<POD-Manifest>
spec:
  nodeSelector:
    size: Large
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution: # preferredDuringSchedulingIgnoredDuringExecution
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In      # In|NotIn|Exists|DoesNotExist|Gt|Lt
            values:
            - Large
            - Medium
````

## Readiness and Liveness Probe
Defines a procedure how Kubernetes can check if a Container is ready or alive.
````yaml
<POD-Manifest>
spec:
  containers:
    - name: app
      image: myapp
      ports:
        - containerPort: 8080
      readinessProbe:         # Just use one of these Probes!
        httpGet:              # Http Probe
          path: /api/ready
          port: 8080

        tcpSocket:            # TCP Probe
          port: 3306

        exec:                 # "Execution of own Script"-Probe
          command:
          - cat
          - /app/is_ready
        
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 8   # Default is 3
      livenessProbe:
        '' # Same usage like readinessProbe
````

## Logs
```shell script
kubectl logs {pod-name}
kubectl logs -f {pod-name} # Live Logs
kubectl logs -f {pod-name} {container-name}
```

## Metrics
Install Metrics
````shell script
git clone https://github.com/kodekloudhub/kubernetes-metrics-server.git
kubectl create -f .
````
Check Metrics
```shell script
kubectl top node
kubectl top pod
```

## Jobs
````shell script
kubectl create job --image={image-name} --dry-run=client -o yaml > job.yaml
````
### Manifest
````yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  completions: 3        # Number of Pods that should execute the job successfully
  parallelism: 3        # Number of Pods that should execute the job parallel
  template:             # Pod-Template
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
````
## CronJobs
````shell script
kubectl create cronjob {cronjob-name} 
    --image={image-name} 
    --schedule="30 20 * * *" 
    --dry-run=client -o yaml > mycronjob.yaml
````
### Manifest
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:                # Job Template
    spec:
      template:               # Pod Template
        spec:
          containers:
          - name: hello
            image: busybox
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

## RBAC
Enable RBAC
````shell script
kube-apiserver --authorization-mode=RBAC
````

### Role and ClusterRole
Contains rules that represent a set of permissions.
A Role always sets permissions withing a particular namespace.
A ClusterRole is a non-namespaced resource.

Example of a Role with permission on namespace "default" see pods:
````yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
````

With a ClusterRole you can scope the permissions on nodes, non-resource endpoints (like /healthz) and namespaces resources (across all namespaces).

````yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Secret
  # objects is "secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
````

### RoleBinding and ClusterRoleBinding

## Create a Cluster with kubeadm

### Container Runtime
On each node you need to install a container runtime.
````shell script
sudo apt-get update
sudo apt-get install docker.io

# Check your version
docker --version

# Start and enable docker
sudo systemctl enable docker

# Verify that it's running
sudo systemctl status docker
# Else you can start it
sudo systemctl start docker

# Install curl if it's not installed yet
sudo apt-get install curl

# Add a signing key to your package manager for repositories of kubernetes (kubeadm kubelet kubectl)
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add

# Add kubernetes repositories to your package manager
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

# Install Kubernetes tools (kubeadm kubelet and kubectl)
sudo apt-get install kubeadm kubelet kubectl
sudo apt-mark hold kubeadm kubelet kubectl

# Verify the installation
kubeadm version

# Turn off swap memory
sudo swapoff -a

# Assign a unique hostname for each host (master, worker01, etc.)
sudo hostnamectl set-hostname master
````

### Install control-plane (master)

Let's do it
````shell script
# Install control-plane
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Copy kubeconfig-File to .kube Folder
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Lets see the result
root@master:~# kubectl get nodes
NAME     STATUS     ROLES           AGE     VERSION
master   NotReady   control-plane   3m43s   v1.25.3

# Deploy a Pod Network (to allow communication between different nodes).
# We will use Flannel as a virtual network
sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# Lets check again
root@master:~# kubectl get nodes
NAME     STATUS   ROLES           AGE     VERSION
master   Ready    control-plane   5m43s   v1.25.3

root@master:~# kubectl get pods --all-namespaces
NAMESPACE      NAME                             READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-4wknq            1/1     Running   0          26s
kube-system    coredns-565d847f94-8s2gr         1/1     Running   0          5m23s
kube-system    coredns-565d847f94-tmrxr         1/1     Running   0          5m23s
kube-system    etcd-master                      1/1     Running   0          5m35s
kube-system    kube-apiserver-master            1/1     Running   0          5m35s
kube-system    kube-controller-manager-master   1/1     Running   0          5m35s
kube-system    kube-proxy-fm96s                 1/1     Running   0          5m23s
kube-system    kube-scheduler-master            1/1     Running   0          5m35s

# I also want to run my pods on the master node. Therefore I have to remove a label:
````shell script
kubectl taint nodes master node-role.kubernetes.io/control-plane-
````

# We are good to go!
````

### Add a worker node
After our `kubeadm init` we received a command that looks like this
````shell script
kubeadm join {IP_OF_MASTER_NODE}:6443 --token {MY_AWESOME_TOKEN} --discovery-token-ca-cert-hash {AWESOME_HASH}
````
This token last for 24 hours. If you want to create a new one, just enter:
````shell script
kubeadm token create --print-join-command
````
We have to execute this command to add our worker nodes to our super awesome cluster! You have to execute the following commands on your new Worker Node!!! 
Let's do it.

````shell script
# We named our worker node worker01
sudo hostnamectl set-hostname worker01

# Join our worker node to the cluster
kubeadm join {IP_OF_MASTER_NODE}:6443 --token {MY_AWESOME_TOKEN} --discovery-token-ca-cert-hash {AWESOME_HASH}

# Check on the master node if the worker node is joined
root@master:~# kubectl get nodes
NAME       STATUS   ROLES           AGE     VERSION
master     Ready    control-plane   21m     v1.25.3
worker01   Ready    <none>          3m43s   v1.25.3

# Lets label our worker node as "worker"
root@master:~# kubectl label node worker01 node-role.kubernetes.io/worker=worker
````

## Create a HA Cluster
For a HA Cluster you need 3 Control Planes and 3 worker nodes.
The kube-apiserver needs to be behind a Load Balancer (Port 6443).

Install the first control plane
````shell script
sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs
````

Join the rest of the control-planes
````shell script
sudo kubeadm join {IP_OF_FIRST_CONTROLPLANE}:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
````

## Upgrade your cluster with kubeadm
You have to do following steps:
* Upgrade your primary control-plane
* Upgrade additional control-planes
* Upgrade your worker nodes

### Upgrade control planes

````shell script
# Check your current version
root@master:~# kubeadm version -o yaml
clientVersion:
  buildDate: "2022-10-12T10:55:36Z"
  compiler: gc
  gitCommit: 434bfd82814af038ad94d62ebe59b133fcb50506
  gitTreeState: clean
  gitVersion: v1.25.3
  goVersion: go1.19.2
  major: "1"
  minor: "25"
  platform: linux/amd64
  
# Determine which version to upgrade
apt update
apt-cache madison kubeadm

# Upgrade your control-plane
# replace x in 1.25.x-00 with the latest patch version
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.25.x-00
apt-mark hold kubeadm

# Verify the upgrade plan
kubeadm upgrade plan

# Upgrade your control plane
sudo kubeadm upgrade apply v1.25.x

# You have to check if you have to upgrade your CNI profiver (in my usecase Flannel)

# Upgrade your other control planes
sudo kubeadm upgrade node v1.25.x

# Drain the node to upgrade kubelet and kubectl
kubectl drain <node-to-drain> --ignore-daemonsets

# Upgrade kubelet and kubectl
# replace x in 1.25.x-00 with the latest patch version
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.25.x-00 kubectl=1.25.x-00 && \
apt-mark hold kubelet kubectl

# Restart the kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Uncordon the node
kubectl uncordon <node-to-uncordon>
````

### Upgrade Worker nodes

````shell script
# Upgrade kubeadm
# replace x in 1.25.x-00 with the latest patch version
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.25.x-00 && \
apt-mark hold kubeadm
sudo kubeadm upgrade node

# Drain the node
kubectl drain <node-to-drain> --ignore-daemonsets

# Upgrade kubectl and kubelet
# replace x in 1.25.x-00 with the latest patch version
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.25.x-00 kubectl=1.25.x-00 && \
apt-mark hold kubelet kubectl

# Restart the kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Uncordon the node
kubectl uncordon <node-to-uncordon>

# Verify the status of your cluster
kubectl get nodes
````











