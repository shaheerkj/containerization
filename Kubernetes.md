# Kubernetes Components
![](./Images/k8-cluster.png)
A Kubernetes cluster consists of a control plane and one or more worker nodes.

### 1. Control Plane Components:
Manage the overall state of the cluster

- [kube-apiserver](https://kubernetes.io/docs/concepts/architecture/#kube-apiserver): The core component server that exposes the Kubernetes HTTP API.
- [etcd](https://kubernetes.io/docs/concepts/architecture/#etcd): Consistent and highly-available key value store for all API server data.
- [kube-scheduler](https://kubernetes.io/docs/concepts/architecture/#kube-scheduler): Looks for Pods not yet bound to a node, and assigns each Pod to a suitable node.
- [kube-controller-manager](https://kubernetes.io/docs/concepts/architecture/#kube-controller-manager): Runs [controllers](https://kubernetes.io/docs/concepts/architecture/controller/) to implement Kubernetes API behavior.
- [cloud-controller-manager](https://kubernetes.io/docs/concepts/architecture/#cloud-controller-manager) (optional): Integrates with underlying cloud provider(s).

### Node Components:

Run on every node, maintaining running pods and providing the Kubernetes runtime environment:

- [kubelet](https://kubernetes.io/docs/concepts/architecture/#kubelet): Ensures that Pods are running, including their containers.
- [kube-proxy](https://kubernetes.io/docs/concepts/architecture/#kube-proxy) (optional): Maintains network rules on nodes to implement [Services](https://kubernetes.io/docs/concepts/services-networking/service/).
- [Container runtime](https://kubernetes.io/docs/concepts/architecture/#container-runtime): Software responsible for running containers. Read [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) to learn more.

# Working with Kind & Kubectl:

[Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) is a tool that we can use to test kubernetes locally using docker containers.**Kind** → the tool that creates clusters. **kindest/node** is the **base image** kind uses to _build_ each Kubernetes node.

```bash
kind create cluster --image kindest/node --name cluster-lab
```

That image already has:
- kubelet
- kubeadm
- kubectl
- containerd
- correct CNI setup

We also need to install `kubectl`, which is a command line client that helps us communicate with our cluster. `kubectl` version should match the `kindest/node` version. (e.g if kindest/node version 1.3, kubectl can be 1.2, 1.3, 1.4).

### Creating multi-node cluster:

For this we need a `.yaml` file that tells kind the configuration we want. So:

```yaml
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

Now we can create the multi-node cluster by specifying our config file:

```bash
kind create cluster --image kindest/node --name cluster-lab --config file.yaml
```

#### Context:

Context points to the current cluster we are utilizing. 

```
# So currently we are using the kind-cka-cluster-2
C:\Users\shahe\OneDrive\Desktop>kubectl config get-contexts
CURRENT   NAME                 CLUSTER              AUTHINFO             NAMESPACE
          kind-cka-cluster     kind-cka-cluster     kind-cka-cluster
*         kind-cka-cluster-2   kind-cka-cluster-2   kind-cka-cluster-2
          kind-kind            kind-kind            kind-kind
```

Some common commands:

```bash
# What nodes are currently in the cluster
kubectl get nodes

# get pods
kubectl get pods

# get pods with their labels
kubectl get pods nginx-pod --show-labels

# To switch context
kubectl config use-context kind-kind
```

##### Ways to change cluster state:

###### **Imperative way:** 

You issue direct commands to change the cluster state:

```bash
kubectl run nginx --image=nginx
kubectl expose pod nginx --port=80
kubectl delete pod nginx
```

- kubectl sends a **one-time instruction** to the API server
- Kubernetes does **exactly that action**
- No long-term desired state is stored in a file
###### **Declarative way:**

You **declare the desired end state** in YAML or JSON  files, and Kubernetes continuously works to match reality to that state.

Example YAML declaration:

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    env: demo
    type: frontend
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

Four primary fields:
- `apiVersion` can be checked with `kubectl explain pods`.
- `kind` is the kind of resource we are defining.
- `metadata` has name, and in labels, we can add our own key pairs.
- `spec` defines what the pod runs.

What happens in kubernetes when you use declarative syntax:
- Kubernetes stores the **desired state**
- Controllers continuously reconcile:
    - `desired state ≠ current state → fix it`

Once we have the `yaml` file, we can use this command to tell kubernetes our desired state:
```bash
kubectl create -f declare.yaml

#-f, --filename=[]:
#Filename, directory, or URL to files to use to create the resource
```

Deleting a pod:
```bash
kubectl delete pod nginx-pod
```

###### Changing the yaml file and matching the desired state:

Lets say we change the `image` tag in the yaml file to `nginx123` (which is not a valid image), we save it again and use the following command to reapply:

```bash
kubectl apply -f declare.yaml
```

We will get an error here, we can use the following command to view error:

```bash
kubectl describe pod nginx-pod
```

We can also edit the configurations of a pod using `kubectl`

```bash
kubectl edit pod nginx-pod

#You cannot change the container image, resource limits, or volumes in an existing pod. 
#Pods are immutable in those core specs.
# Kubernetes will delete and recreate the pod if a change is applied that requires a spec update.
```

###### Getting interactive shell inside the pod:

```bash
kubectl exec -it nginx-pod -- bash
```

###### Getting a yaml file from imperative commands:

Instead of writing lengthy yaml scripts, we can also use the following imperative command, and it will output a yaml file that we can put in a `.yaml` file.

```bash
#This generates a yaml output that we can copy paste to a .yaml file

kubectl run nginx --image=nginx --dry-run -o yaml
```


## Deployment & ReplicaSet:

###### **ReplicationController (RC)**

- **Old / legacy Kubernetes object**
- Ensures a **fixed number of pod replicas** are running at all times
- If a pod dies, it **creates a new one automatically**
- Uses **label selectors** to manage pods
- **Mostly replaced by ReplicaSet** (rarely used now)

###### **ReplicaSet (RS)**

- **Newer and recommended replacement** for ReplicationController
- Same core job: **maintains desired number of pod replicas**
- Supports **advanced label selectors** (`matchExpressions`)
- Usually **not created directly** — managed by a **Deployment**
- Enables **rolling updates & rollbacks** (via Deployment)

Replication can span multiple nodes.

#### Making a `ReplicaController` / `ReplicaSet`:

Using a `yaml` file for deployment. When we are making a ReplicaController, we have to specify a template in `spec`, so that it knows what the containers are running behind it.
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
  labels:
    env: demo
spec:
  template:
    metadata:
      labels:
        env: demo
      name: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
  replicas: 3
```

After this, we can run the following command:

```bash
kubectl apply -f declare.yaml

# Output
#C:\Users> kubectl get po
#NAME             READY   STATUS    RESTARTS   AGE
#nginx-rc-75578   1/1     Running   0          57s
#nginx-rc-cnr7r   1/1     Running   0          57s
#nginx-rc-vf5ln   1/1     Running   0          57s
```

We can get the information about replicationControllers with this command:

```bash
kubectl get rc <name>

kubectl describe rc <name>
```

We would normally prefer to use `ReplicaSet` over `ReplicationController`. With `ReplicaSet`, we can also manage the pods that weren't created with the same `ReplicaSet` by using the "matchLabels" tag in its `yaml` declaration.

So now changing some of the content in our previous yaml:

```yaml
apiVersion: apps/v1
kind: ReplicaSet # -> Changed to ReplicaSet
metadata:
  name: nginx-rc
  labels:
    env: demo
spec:
  template:
    metadata:
      labels:
        env: demo
      name: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
  replicas: 3
  selector: # -> ADDED A SELECTOR
    matchLabels: 
      env: demo # -> MATCHES THE PODS WHERE LABELS ARE "demo"
```

So, now we can manage other pods that are labelled demo regardless of whether or not they were deployed with the same `ReplicaSet`. 

Also worth noting, that the `apiVersion` field changes too. We can check the version using the following command:

```bash
>kubectl explain rs
GROUP:      apps
KIND:       ReplicaSet
VERSION:    v1
```

From this we can change the `apiVersion: apps/v1` (Group field is involved too)


###### Some commands:

Delete a ReplicationController:

```bash
kubectl delete rc/<name>
```

Making changes in live object, instead of changing `yaml` file and reapplying:

```bash
kubectl edit rs/nginx-rs
# From here we can edit the object and those changes will apply
```

Using the imperative way:

```bash
kubectl scale --replicas=10 rs/<name>
```

Checking the status of a rollout update:

```
kubectl rollout history deploy/<name>
```

Changing the image:

```
kubectl set image nginx <nameOfResource/<name> nginx=nginx:1.19
```

If we want to undo the rollback:

```
kubectl rollout undo deploy/nginx-deploy
```
### Deployment:

A **Deployment** is a **higher-level controller** that manages how your application is **created, updated, scaled, and rolled back**.
**Purpose:**
- Run an application reliably
- Handle **rolling updates**
- Enable **rollbacks**
- Scale pods easily
- Maintain desired state

You normally **interact with Deployments**, not pods or ReplicaSets.

Relationship
```
Deployment
 └── ReplicaSet
      └── Pods
```

Example `yaml` config for deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    env: demo
spec:
  template:
    metadata:
      labels:
        env: demo
      name: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
  replicas: 3
  selector:
    matchLabels:
      env: demo
```

Deployed using:

```bash
kubectl apply -f declare.yaml
```

Now, if we use the following command:
```
>kubectl get all

NAME                                READY   STATUS    RESTARTS   AGE
pod/nginx-deploy-6c7688d775-jvxcb   1/1     Running   0          71s
pod/nginx-deploy-6c7688d775-rmngz   1/1     Running   0          72s
pod/nginx-deploy-6c7688d775-v7r2b   1/1     Running   0          71s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   2d5h

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy   3/3     3            3           72s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deploy-6c7688d775   3         3         3       72s
```

This commands shows everything in the current cluster. We see a `ReplicaSet` also running here because it is created in the deployment. So deployment creates the `ReplicaSet` and **Deployment controls the ReplicaSet, and the ReplicaSet controls the Pods**.

## **Kubernetes Services:**

Kubernetes Services exist to give Pods a stable way to be reached.

###### Node Port:

The port on which the application is exposed on. In Kubernetes, the default port range for a NodePort service is ==**30000-32767**==

- nodeport: 
	- **Where users connect**
	- Open on **every node’s IP**
	- Exposes the service **outside the cluster**
	- **Purpose:**  Entry point from the **external world**
- port: 
	- Port **on the Service**
	- Used for **internal cluster access**
	- Other pods talk to the service using this port
	- **Purpose:** Stable **virtual service port**
- targetport:
	- Port **inside the Pod/container**
	- Where the application is actually listening
	- **Purpose:** Actual **app port**
![[nodeport.png]]

 Implementing:
 