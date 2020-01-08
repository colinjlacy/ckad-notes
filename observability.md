# Observability

## Readiness and Liveness Probes
- a pod has a status and some conditions
    - **pending** state: figuring out what/where to do with the pod
    - **container creating**: pulling images required
    - **running**: where it lives until the process ends
- can see the pod status in the output of the `kubectl get` command
- conditions tell us more about the pod status:
    - an array of boolean values:
        - **PodScheduled**
        - **Initialized**
        - **ContainersReady**
        - **Ready**
    - available in the `describe pods` command under the `conditions` section
    - `Ready` condition means that the *container* is running and ready to do its thing
        - however, this does not account for any application setup/startup that takes place once the container is "ready"
        - need a way to tie the Ready condition to the actual state of the applicaiton in the container
- can set up different kinds of tests or "probes"
    - the `spec.containers[].readinessProbe` field
    - HTTP test to a specific endpoint
        - `spec.containers[].readinessProbe.httpGet.path: <your-api-path>`
        - `spec.containers[].readinessProbe.httpGet.port: <your-api-port>`
        - can add additional options:
            - `initialDelaySeconds`: anticipated warmup time before the probe is ready, in sec
            - `periodSeconds`: amount of time to wait before trying again, in sec
            - `failureThreshold`: amount of times to try before calling it quits (default 3)
    - TCP test on a port
        - `spec.containers[].readinessProbe.tcpSocket.port: <your-tcp-port>`
    - run a script to completion (exit 0), indicating that things are all right
        - `spec.containers[].readinessProbe.exec.command[]`
            - a list of command segments
- k8s will use the liveness probe to overcome gaps in the cluster's understanding of whether or not the container is actually alive
    - can be configured to periodically check health on containers
    - as a developer, you get to define what it means for an app to be healthy
    - the `spec.containers[].livenessProbe` field
        - similar to readiness probe, you have the exact same options

## Logging
- to view logs on a pod, we run `kubectl logs <name-of-pod>`
    - add the `-f` flag to follow the live log stream
- if there are multiple containers in a pod, you must specify a container name when calling for logs:
    - `kubectl logs <name-of-pod> <name-of-container>`

## Monitoring
- what would we want to monitor
    - number of nodes in the cluster
    - how many are healthy
    - CPU, memory, network, and disk use
    - pod-level metrics
        - CPU and mem consumption on them
- k8s does not come with a full-featured monitoring solution
    - metrics server
    - originally this was Heapster, which is now deprecated
- number of available solutions:
    - prometheus
    - elastic stack
    - datadog
    - dynatrace
- k8s dev test only requires knowledge of the metrics server

### Metrics Server
- aggregates analytics from pods, aggregates them, and stores them in memory
- does not store metrics on disk
    - cannot see historical perf data
- k8s runs an agent on each node - kubelet
    - kubelet also contains an agent called cAdvisor
        - responsible for collecting metrics and sending them to the metrics server
- to deploy:
    - in minikube:
        - `minikube addons enable metrics-server`
    - in all others:
        - clonse the `kubernetes-incubator/metrics-server` repo from GitHub
        - deploy the build: `kubectl create -f deploy/1.8+/`
- to view metrics:
    - `kubectl top node`: view performance metrics of nodes
    - `kubectl top pod`: perf metrics of pods
