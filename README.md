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
kubectl config view --minify | grep namespace:
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
spec:
  containers:
    - name: webserver
      image: nginx
      ports:
        - containerPort: 80
      securityContext:
        privileged: true
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
      envFrom:                            # Bind all ENVs of Configmap to Pod!
          - configMapRef:
              name: {NAME_OF_CONFIGMAP}
        
                  
      command: ["sh"]                     # ENTRYPOINT in Docker
      args: ["-c", "echo 'Hello World'"]  # CMD in Docker
````

## ReplicaSet

## ReplicaController

## Deployment

## Service

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

## Taint

## Resource-Management

## Label