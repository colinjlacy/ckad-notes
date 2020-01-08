# Security

## Security Contexts

### Docker Security
- each host, including those with docker installed, have a bunch of processes running
- unlike VMs, containers are not isolated from their hosts
- containers are instead isolated using namespaces and links
- all of the processes run by the containers are run on the host itself, but on its own namespace
    - that is, the container can only see its own proceses, and none on the host
    - however, the host can see all of the processes, including those inside the docker container
        - from the host's PoV, the docker process will have a different process ID
- the host has a set of users
    - root plus whatever else
    - by default, docker processes are run as root
        - docker implements a set of security features that limit what the root user in the container can do on the host, preventing bad things
        - processes running inside the container don't have the priveledges to do bad things on the host
            - can overwrite this restriction:
                - with the with the `--priveledged` flag to set all host priveledges
                - with the `--cap-add` flag to set individual priveledges
    - alternatively, you can run docker processes by specifying a user ID during the `docker run` command with the `--user` option set:
        - `docker run --user=1000 ...`
    - can have the user ID defined in the Dockerfile    
        - removes the need to have to define the user at runtime
        - expects the user to exist

### Kubernetes Security
- can configure security settings on the pod and the container
    - settings set on the pod will be inherited by the container
    - settings set on the container will override those set on the pod
- setting security on the pod can be done with the `spec.securityContext` property:
    ```yaml
    spec:
        securityContext:
            runAsUser: 1000
    ```
- setting security on the container leverages the same definitions, but moved into the `spec.containers[]` definition listing:
    ```yaml
    containers:
        - name: ubuntu
          image: ubuntu
          command: ["sleep", "3600"]
          securityContext:
            runAsUser: 1000
    ```
    - at the container level, you can add individual capabilities
        - NOT supported at the pod level
        ```yaml
        securityContext:
            runAsUser: 1000
            capabilities:
                add: ["MAC_ADMIN"]
        ```

## Service Accounts
- lnked to other security concepts - e.g. auth, RBAC, etc
- two types of accounts used:
    - user accounts, used by humans
        - launch apps, scale up/down
    - service accounts, used by machines
        - monitoring, automted app launch (a la Jenkins)
        - things that interact with APIs in an automated way
- to create:
    - `kubectl create serviceaccount <sa-name>`
- to list:
    - `kubectl get serviceaccounts`
- tokens:
    - tokens are used by the service account when requesting access to information on variou APIs
    - automatically generated at the time a service account is created
        - stored as a Secret
        - the secret is then linked to the service account
        - read like any other Secret: `kubectl describe secret <secret-name>`
- if an application using a service account is deployed on the k8s cluster
    - you can mount the secret to the deployed app as a volume, so that its token value can be read easily from inside the application
        - using the `spec.serviceAccount` field in the definition:
            ```yaml
            spec:
                constiners:
                  - name: nginx
                    image: nginx
                serviceAccount: <name-of-service-account>
            ```
- there is always a default token created for a default service account
    - similarly, the token for the default service account is always mounted as a volume to every pod that is deployed on the k8s cluster
        - stored at `/var/run/secrets/kubernetes.io/serviceaccount`
    - to specify a different service account, use the spec definition described above
        - will store the specified SA token at the same file path

