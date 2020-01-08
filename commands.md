# Commands and Arguments

## Commands in Docker
- containers are meant to run a specific task or process
    - thus, not necessarily built to be long-running
    - as opposed to a VM
    - only lives while the process inside it is alive, then exits
- who defines the process that will determine the lifecycle of the image?
    - the `CMD` property in the Dockerfile
- can append a command to a `docker run` command
    - e.g. `docker run <docker-image> sleep 5`
- how to make that additional command permanent?
    - create a second image from the base image (which uses the first `CMD`)
    - and then have the second image use the second `CMD`
- how to parameterize into a Dockerfile?
    - e.g. `docker run <docker-image> 10`
    - this is where the `ENTRYPOINT` instruction in the Dockerfile comes in
        - `ENTRYPOINT ["sleep"]` will act as a base command, to which paramters can/will be attached
        - the in above example, this results in `sleep 10` being run on container startup
        - used as opposed to `CMD`
    - can combine `ENTRYPOINT` and `CMD` to provide default params:
        ```dockerfile
        ENTRYPOINT: ["sleep"]
        CMD: ["5"]
        ```
        - this will result in `sleep 5` being run if no param is passed
        - if a param is passed (as in the example above), then the `CMD` param will be overwritten
    - can also overwrite the `ENTRYPOINT` if necessary using the `--entrypoint` option on the `docker run` command:
        - `docker run --entrypoint sleep2.0 <docker-image> 10`

### Commands and Arguments in a K8s Pod
- appending args to the pod definition file's internal `docker run` command is simple:
    - add an array of string arguments to the `spec.containers[].args` property, e.g.:
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
            name: ubuntu-sleeper
        spec:
            containers:
                - name: ubuntu-sleeper
                  image: ubuntu-sleeper
                  agrs: ["10"]
        ```
        - in this example, the `"10"` gets attached to the pod's internal `docker run` command, and subsequently the `ENTRYPOINT` property of the image Dockerfile
        - overwrites the Dockerfile `CMD` property
- can also overwrite the `ENTRYPOINT` entirely:
    - using the `spec.containers[].command` property, e.g.:
        ```yaml
        spec:
            containers:
                - name: ubuntu-sleeper
                  image: ubuntu-sleeper
                  command: ["sleep2.0"]
                  agrs: ["10"]
        ```
    