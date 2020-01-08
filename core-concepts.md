# K8s Architecture

## Nodes
- a machine on which k8s is installed
- a worker machine in which containers will be installed/run
- also called minions in the past
- must have multiple nodes in case one node fails
    - helps in sharing load
    - called a cluster

### How do we manage a cluster of Nodes?
- that's the master's job
    - another node iwth k8s installed
- responsible for the actual orchestration of containers on worker nodes

## Components of K8s:
- API server (master)
    - interacting with the k8s cluster
- etcd (master)
    - key-store
    - stores all config data used to manage the cluster
- controller (master)
    - brains behind orchestration
- scheduler (master)
    - responsible for distributing work (containers) across nodes
- kubelet (worker)
    - agent that runs on each node in the cluster
    - ensures node state validity
- container runtime (worker)
    - software used to run containers (docker)

## Kubectl
- manages intreraction with clusters
- examples:
    - `run`: runs a deployment
    - `cluster-info`: describes the state of the cluster
    - `get nodes`: describes the state of nodes

## Pods
- k8s doesn't deploy containers directly to nodes
- instead containers are deployable to pods
- a pod is the smallest object you can create in k8s
- simplest of the simplest:
    - single node
    - single pod
    - running a single container
- to scale:
    - we create new pods that run the same single container
    - can do this in the same node if the node has capacity
    - or can scale to a new node if new capacity is needed

### Multi-container pods
- a single pod can have multiple containers
- generally not multiple containers of the same kind
    - will generally be helper containers doing supporting tasks
    - e.g. processing a file, processing network
- thought is that as they live in the same pod, they live and die together
- share the same network space on `localhost`
- share the same storage space as well

### Easy pod creation
- run `kubectl run nginx --image nginx`
- creates a pod, pulls the Nginx image from Docker Hub
- to check state, run `kubectl get pods`

### Pods with YAML
- required fields in a resource YAML:
    - `apiVersion`: the API path for the resource you're creating
    - `kind`: the kind of objecy you'd like to create
    - `metadata`
        - `name`: name of the resource being deployed
        - `labels`: any key-value pairs you'd like to add
            - gives the ability to group or query resources
    - `spec`: a dictionary of resource definitions
- Pod spec:
    - `containers`: a list of containers being installed in that pod
        - `name`: name of the resource for direct reference
        - `image`: name of the image in you image repository
- to create a pod from a file: `kubectl create -f <resource-file-path>`
- to learn about the state of the pod: `kubectl describe pod/<pod-name>`

## Replication Controllers
- **an older technology that is replaced by Replica Sets**
    - basically the same interface
- helps achieve HA by automatically bringing up the specified number of pods and ensuring they run at all times
- allows for load balancing and auto-scaling
- RC spec:
    - `template`: a template for a pod definition
        - matches the structure of a pod definition file
        - with the exception of the first few lines (i.e. apiVersion and kind)
    - `replicas`: the number of replicas you'd like to have (static number)
- to view deployed RC: `kubectl get replicationstroller`

## Replica Sets
- resource definition file is almost identical to Replication Controller
- major difference:
    - `spec.selector`: any pods that match a label key-value pair:
        - `matchLabels`: the key-value pair to match
            - targets any pods that have a matching label
        - this is required for ReplicaSet, not for ReplicaController

### Labels and Selectors
- why have labels and selectors?
    - role of the ReplicaSet is to monitor pods and create new ones if a shortage exists
    - labelling pods during creation allows RSs to know which pods to monitor
        - done using the `matchLabels` field

### Scaling RSs
- multiple ways to do it
    - easiest is to update the number of replicas in the resource definition
        - and then run `kubeclt`
    - another way is to use the `kubeclt` cli to scale manually
        - down side is that this does not persist to the file
    - later, we'll talk about scaling based on load

## Deployments
- a k8s object that comes higher than the replicaset in the heirarchy
- definition:
    - exactly the same API as the ReplicaSet
    - only difference is that it is of kind `Deployment`
- deployments automatically create ReplicaSets underneath
    - can access via `kubectl get replicaset`
- also, naturally, there are pods created as well.

## Namespaces
- way to ringfence named resources, with their own set of rules and interactions
- the `default` namespace is created automatically when the k8s cluster is set up
    - this is where we interact with the data plain
- two internal namepsaces also created by default
    - `kube-system`: where a bunch of inner workings live that we don't want to mess with
    - `kube-public`: resources made available to all users live here
- namespace isolation
    - create different namespaces to isolate dev and prod resources
    - each has its own set of policies to decide which gets to do what
    - names can exist multiple times in multiple namespaces
        - but not in the same namespace
- in order to cross namespaces
    - must append the name of the service trying to reach with `<namespace>.svc.cluster.local`
- to target a namespace in the CLI, use the `--namespace` flag
    - e.g.: `kubectl get pods --namespace=kube-system`
- to target a namespace in a template, you would use the `metadata.namespace` property in the definition file
- to create a namespace via definition file:
    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
        name: dev
    ```
- to switch namespaces:
    - `kubectl config set-context $(kubectl config current-context) --namespace=<target>`
- to view things in all namespaces, use the `--all-namespaces` flag on the `kubectl get <resource>` command

### ResourceQuota
- set on the namespace
    - uses the `metadata.namespace` property to target a namespace
