# Configurations

## Resource Requirements
- minimum amount of CPU, memory, and disk that a pod requires in order to run successfully
- used by the scheduler to determine which nodes to put the pods on when deistributing across nodes in a cluster
    - will only put pods on nodes that have the available resources
    - will only scale nodes if necessary
- done with the pod spec `spec.containers[].resources.requests` property:
    - `memory`: a string getting either `XG` for X amount of GB, or `XM` for X amount of MB
    - `cpu`: a number describing the number of CPU that the application needs
        - what does this value mean?
            - means a **vCPU**, or a core, reserved for this application
- can also set resource limits on the pods
    - done with the pod `spec.containers[].resources.limits` property
    - used to ensure good behavior, ensuring that pods don't starve resources on a node and crash the node
    - amount values are defined the same way
    - note that:
        - exceeding CPU will get the extended CPU usage throttled by the hypervisor
        - exceeding memory won't get an immediate penalty, but long-term abuse will cause the pod to be terminated
- remember that a resource request and limit are set on the containers inside the pod
    - each container has its own resource specs, and will get the resources it requests
    - thus, each pod gets the total number of resources asked for, for all of the containers in the pod

## Taints & Tolerations
- how you can restrict what pods are placed on what nodes
    - sets restrictions on what pods can be scheduled on what nodes
- apply a taint to a node to prevent nodes from being placed on them
    - by default, pods don't have any tolerations
    - any taint will repel all pods by default
- apply a toleration to a pod in order for that pod to "tolerate" the taint
    - all other pods will be distributed to nodes that are not "tainted"
- how to do this:
    - `kubectl taint nodes <node-name> <key>=<value>:<taint-effect>`
- there are three possible **taint-effects**:
    - `NoSchedule`: no interolerable pods will be scheduled (but existing intollerable pods will remain as long as they are running)
    - `PreferNoSchedule`: the scheduler will try not to schedule intolerable pods to the tainted node, if possible
    - `NoExecute`: same as `NoSchedule`, but in this case, exising pods will be evicted
- tolerations are added to the pod spec:
    - `spec.tolerations[]`
    - each toleration entry has four properties, which correspond to a taint definition:
        - `key`: the key of the taint
        - `operator`: usually `Equal`
        - `value`: the value to complete the key-value pair
        - `effect`: the effect of the taint that we're tolerating
- example use cases:
    - dedicate a node to a specific application
        - taint the node with `app=<app-name>:NoSchedule`
        - apply the toleration to the deployment with that app name
    - have a pod target a node with a specific underlying infrastructure
- note that the master node is actually tainted 
    - this is why no pods are ever scheduled on the master node
    - to see this taint: `kubectl describe node kubemaster | grep Taint`

## Node Selectors
- example use case:
    - you want to run data processing apps on specific nodes that have better data computing horsepower, e.g. high GPU
- use the pod spec property `spec.nodeSelector`, e.g.:
    ```yaml
    spec:
        nodeSelector:
            size: Large
    ```
    - where does this key-value pair come from? The node labels, of course!
- labeling nodes with key-value pairs via command line
    - `kubectl label nodes <node-name> <label-key>=<label-value>`
- limitations of node selectors:
    - cannot achieve "OR" clauses
    - nor "NOT" clauses

## Node Affinity
- ensure that pods are hosted on particular nodes
- done in the pod spec:
    ```yaml
    spec:
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
    ```
    - this example demostrates how to handle the "OR" condition of only scheduling to either Large or Medium labeled nodes
    - alternatively, you can accomplish "NOT" clauses with the `NotIn` operator
    - or, you can use the `Exists` operator to see if the label key exists on a node (no `values` needed)
        - this would give the "NOT" by default when the key in question is only applied to the nodes you'd like to use

### Types of Node Affinity
- two phases of the pod existence lifecycle:
    - DuringScheduling: when a pod is going to be placed
    - DuringExecution: when a pod is already running, and a change in the env might change the affinity of the node
- `requiredDuringSchedulingIgnoredDuringExecution`:
    - mandates that a pod is placed on a node with given rules
    - pod will not be scheduled if it cannot be matched
    - will not affect existing/running pods if the environment changes
- `preferredDuringSchedulingIgnoredDuringExecution`:
    - scheduler will ignore node affinity rules if a node cannot be found
    - will not affect existing/running pods if the environment changes
- `requiredDuringSchedulingRequiredDuringExecution`: (planned, not yet available)
    - same as the top, except that changes in the env will cause pods to be evicted if the matching affinity label is removed from the node

## Taints & Tolerations vs Node Affinity
- taints/tolerations ensure that only those pods that you have in mind will end up on the right nodes
    - does not exclude nodes that you might not have in mind
- node affinity ensures that pods end up on only the selected nodes and none other
    - does not exclude pods that might also end up on those nodes
- when used in combination, they can achieve the right amount of exclusivity
    - pods only show up on their targetted nodes
    - targetted nodes will only have the intended pods