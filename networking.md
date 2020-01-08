# K8s Networking

## Services:
- the k8s node has its own network IP address
    - the internal pod network has its CIDR of IPs on a subnet
    - each pod on a node is given an IP in that CIDR
    - clearly if you're not in that internal pod network, you can't access those pods via IP
    - options:
        - could SSH into the node
            - bummer
        - need soemthing to map a port on the node straight into the running pod
            - seems like a simplistic approach
- NodePort
    - a nodeport service is the type listed above, in its simplest form
    - a port on the node is mapped directly to the deployed pod
        - the port listens for traffic, and forwards traffic to the IP of the deployed pod
    - three ports involved:
        - port on the pod is the **target port**
        - port on the service is the **port**
            - inside the cluster the service has its own IP address
        - port on the node is the **NodePort**
            - NodePorts can only be in a **valid range, which is between 30000 and 32767**
    - what does a service definition look like?
        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
            name: <name>
        spec:
            type: NodePort
            ports:
              - targetPort: 80      # port on the pod
                port: 80            # port on the service, only mandatory field
                nodePort: 30008     # port on the node
            selector:
                <pod-label-key>: <pod-label-value>
        ```
        - if you don't provide a `spec.ports.targetPort`, it is assumed to be the same as `spec.ports.port`
        - if you don't provide a `spec.ports.nodePort`, a free port within the valid range is automatically assigned 
        - note that `spec.ports` is an array
            - can have multiple port mappings within a single service
        - `spec.selector` is pointing to a list of key-value pairs on whatever pods you want to hit with your service
    - if a NodePort service finds mutliple pods with the same label k/v, it will automatically create an internal LB to balance traffic between them
        - distribution algorit is random
        - session affinity applied
    - when a deployment is distributed across multiple nodes, k8s automatically creates a NodePort service across all nodes
        - just like with one node, the port on all nodes where this pod is placed is mapped to the pod
        - so if you go with port 30001, and the pod is on 3 nodes, it'll be port 30001 on all three nodes
- ClusterIP
    - creates a virtual IP within the cluster to enable communication betwen different services inside the cluster
    - helps answer the question of how to handle internal communication distribution
        - cannot rely on pod-specific IP addresses, obviously
    - groups pods together and acts as a single interface for the group
        - random distribution of load balancing between the pods within the service
    - makes it very easy to deploy a microservice application within the cluster
        - each service gets an IP and name assigned
        - **service name is what should be used to connect from other pods.**
    - what does a definition look like?
        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
            name: back-end
        spec:
            type: ClusterIP 
            ports:
              - targetPort: 80
                port: 80 
            selector:
                <pod-label-key>: <pod-label-value>
        ```
        - `ClusterIP` is actually the default Service type
        - `spec.ports[].targetPort` is the port that the service will expose on the pod
        - `spec.ports[].port` is the port that will be used to access the service endpoint
- LoadBalancer
    - an LB is provisioned in the cloud provider to distribute traffic between deployed pods
    - note that this is only available if the underlying infra has a built-in load balancer that can be provisioned
        - and keep in mind, you have to pay for these load balancers

## Ingress
- simply put, it's a proxy server that allows access into your cluster
- helps your users access your application using a single, accessible URL
    - configure to access your cluster services
    - a Layer 7 LB built-in to the k8s cluster that can be stood up using native k8s primitives
    - **things to keep in mind:**
        - still have to expose it by mapping it to a cluster port, or a public cloud LB
        - have to perform all of your auth, load-balancing, SSL, and routing configs on the ingress controller
- how does it work?
    - when you deploy, you deploy a reverse proxy technology, like Nginx, HAProxy, or Traefik
        - whatever solution you deploy is called your **ingress controller**
            - according to this guy, k8s ingress doens't come configured with a specific ingress controller by default
            - GCE and Nginx are supported by the k8s community
            - keep in mind, there's more to an ingress than just the ingress controller
                - a bunch of other stuff built around it to support ingress routing, like auto-updating configurations when new routes are posted
            - how do you deploy an ingress controller?
                - it's a deployment!
                    ```yaml
                    apiVersion: extensions/v1beta1
                    kind: Deployment
                    metadata:
                        name: nginx-ingress-controller
                    spec:
                        replicas: 1
                        selector:
                            matchLabels:
                                name: nginx-ingress
                        template:
                            metadata:
                                labels:
                                    name: nginx-ingress
                            spec:
                                containers:
                                  - name: nginx-ingress-controller
                                    # special image used for k8s ingress
                                    image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0  
                                # the following is specific to this nginx ingress image
                                args:
                                  - /nginx-ingress-controller
                                  - --configmap=$(POD_NAMESPACE)/nginx-configuration # need to pass in a configmap for nginx configs, can be blank
                                env:
                                  - name: POD_NAME
                                    valueFrom: 
                                        fieldRef:
                                            fieldPath: metadata.name
                                  - name: POD_NAMESPACE
                                    valueFrom:
                                        fieldRef:
                                            fieldPath: metadata.namespace
                                ports:
                                  - name: http
                                    containerPort: 80
                                  - name: https
                                    containerPort: 443
                    ```
                - here is a simple configmap example, which, as stated, can be empty to start:
                    ```yaml
                    kind: ConfigMap
                    apiVersion: v1
                    metadata:
                        name: nginx-configuration
                    ```
                - now we need a NodePort service to expose it to a network access point:
                    ```yaml
                    apiVersion: v1
                    kind: Service
                    metadata: 
                        name: nginx-ingress
                    spec:
                        type: NodePort
                        ports:
                          - port: 80
                            targetPort: 80
                            protocol: TCP
                            name: http
                          - port: 443
                            targetPort: 443
                            protocol: TCP
                            name: https
                        selector:
                            name: nginx-ingress
                    ```
                - in order for the ingress controller to monitor resources for configuration changes, it needs a service account with the right set of permissions:
                    ```yaml
                    apiVersion: v1
                    kind: ServiceAccount
                    metadata:
                        name: nginx-ingress-serviceaccount
                        # and role stuff...
                    ```
    - then you configure your routing and SSL rules
        - whatever rules you configure are called your **ingress resources**
            - created using k8s definition files:
                - example of a single-service ingress:
                    ```yaml
                    apiVersion: extensions/v1beta1
                    kind: Ingress
                    metadata:
                        name: <ingress-name>
                    spec:
                        backend:
                            serviceName: <target-service-name>
                            servicePort: <target-service-port>
                    ```
                - example of configuration using path rules:
                    ```yaml
                    apiVersion: extensions/v1beta1
                    kind: Ingress
                    metadata:
                        name: <ingress-name>
                    spec:
                        rules:
                          - http:
                                paths:
                                  - path: /some-path
                                    backend:
                                        serviceName: <target-service-name>
                                        servicePort: <target-service-port>
                                  - path: /some-other-path
                                    backend:
                                        serviceName: <other-target-service-name>
                                        servicePort: <other-target-service-port>
                    ```
                - example of configuration using domain name rules:
                    ```yaml
                    apiVersion: extensions/v1beta1
                    kind: Ingress
                    metadata:
                        name: <ingress-name>
                    spec:
                        rules:
                          - host: <some-subdomain>.<domain>.com
                            http:
                                paths:
                                  - backend:
                                        serviceName: <target-service-name>
                                        servicePort: <target-service-port>
                          - host: <some-other-subdomain>.<domain>.com
                            http:
                                paths:
                                  - backend:
                                        serviceName: <other-target-service-name>
                                        servicePort: <other-target-service-port>
                    ```
                    - notice that in this case, we have specified `spec.rules[].host` fields
                        - if you don't specify said field, then k8s assumes all traffic is on the base host and routes by paths
                    - can still have multiple path definitions in these domain-based definitions, as we did in the previous example
            - when you `kubectl describe` your ingress, you'll notice that there will be a `Default backend` associated with your ingress
                - this is for 404 routing
                    - when a user accesses a path or sub-domain that is not configured
                - need to ensure you have a 404 service configured once you expose an ingress

## Network Policies
- basics:
    - two types of traffic:
        - ingress
            - traffic originating from outside the system, coming into the system
        - egress
            - traffic originating from inside the system, going out of the system
- network security
    - each node, pod, and service has its own IP address
    - whatever solution you implement, the pods should be able to communicate with each other without having to configure additional properties like routes
        - using the built-in DNS
        - **with the built-in k8s "All Allow" rule for network traffic**
- how do you shut down a potential communication path?
    - e.g. you want to prevent a user-facing web-server from being able to communicate directly with your DB server
    - you use a **NetworkPolicy** resource
- NetworkPolicy is assigned to one or more Pods
    - define rules within the network policy
        - e.g. only allow ingress traffic from a certain pod on a certain port
        - only applicable to the pod on which a network policy is applied
- how do you link a NetworkPolicy to a Pod?  using lables and selectors
    - e.g., on the pod:
        ```yaml
        metadata:
            labels:
                role: db
        ```
    - and on the networkpolicy:
        ```yaml
        podSelector:
            matchLabels:
                role: db
        ```
- what's in a policy
    - rules to allow:
        - ingress traffic
        - egress traffic
        - or both
    - example ingress policy, matching the example use case above:
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
            name: db-policy
        spec:
            podSelector:
                matchLabels:
                    role: db
            policyTypes:
                - Ingress
            ingress:
                - from:
                    - podSelector:
                        matchLabels:
                            name: api-pod
                  ports:
                    - protocol: TCP
                      port: 3306
        ```
    