# Pod Design

## Labels, Selectors, Annotations
- labels and selectors are standard methods of grouping things together
- labels are properties attached to each item
- selectors help filter each item
    - bascally applying selectors to select the specific label valuess you're interested in
- examples labels:
    - app name
    - function
- how to apply labels:
    - use the pod `metadata.labels[]` property, e.g.
        ```yaml
        metadata:
            labels:
                appName: my-awesome-app
                function: front-end
                release: v1
        ```
- how to apply selectors:
    - in imperitive/CLI commands, use the `--selector` option:
        - `kubectl get pods --selector <label-key>=<label-value>[,<label-key>=<label-value>]`
- remember that in a ReplicaSet definition, we use a selector to group pods as targets of that ReplicaSet:
    - `spec.selector.matchLabels[]`: a key-value pair of labels, used to filter to the appropriate pods
    - keep in mind, the ReplicaSet (and Deployment) definition also includes a pod template:
        - `spec.template`, which has the field `spec.template.metadata.labels` for applying labels to the pods it deploys
    - in general, we want these to match if we're doing things that make sense
        - that is, at least one label k/v pair should match
- annotations are used to record other details for inventory purposes
    - e.g. build information, e.g. `buildversion: 1.3.4`
    - applied at the `metadata.annotations[]` property
    - seems like they can't be used to apply selectors

## Rolling Updates and Rollbacks in Deployments
- each time we deploy a new version of a deployment, it's called a `rollout`
    - to see the status of a rollout, run the command:
        - `kubectl rollout status deployment/<deployment-name>`
    - to see the revision history of the deployment:
        - `kubectl rollout history deployment/<deployment-name>`
- two types of deployment strategies:
    - `recreate`: all-at-once
        - incurs downtime
        - not default
    - `rolling`: replace one at a time in a rolling update
        - no downtime
        - the default
- how do you update your deployment?
    - could easily be as simple as changing the version of the `spec.template.spec.containers[].image` value, which would trigger a pull of the new image version and a rolling deployment of that version of the container in your pods
    - can also do this implicitly:
        - `kubectl set image deployment/<deployment-name> \<container-name>=<image-name>:<image-version>`
        - remembe rthat doing it this way puts the actual status out of sync with the definition file
- can see the difference between receate and rolling update in the `describe deployment` command
    - shows you how the deployent was last scaled out
- worth knowing that in a deployment update, a new ReplicaSet is stood up and populated as the original ReplicaSet is spun down individually
- rollbacks are done with the `kubectl rollout undo deployment/<deployment-name>` command
    - destroys the pods in the new ReplicaSet while bringing the pods in the old ReplicaSet back
- the `run` command creates a deployment, not just a pod
    - another way of creating a deployment, by only stating the image name without using a definition file

## Jobs
- different types of workloads that a container can serve
    - e.g. web app and database
        - meant to run for a long period of time until manually taken down
    - other types of workloads such as batch processes, analytics, or reporting
        - carry out a specific task and then finish
        - e.g. computation on a large data set, anayze an image, etc.
- how this works in Docker:
    - `docker run ubuntu expr 3 + 2` => Ubuntu adds 3 and 2
    - docker executes the process expected of the container, and then exits
    - running `docker ps` shows the return code/status of the exited container (`0` on success)
- how we would do this in k8s with a pod definition file:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
        name: math-pod
    spec:
        containers:
          - name: math-add
            image: ubuntu
            command: ['expr', '3', '+', '2']
    ```
    - when the pod runs, it executes nad exits, and then goes into a `Completed` state, which can be seen when running `kubectl get pods`
        - probably because of the exit 0
    - **Problem is, it will recreate the container in an attempt to leave it running**
        - why? because the default behavior of pods is to restart the container whenever it is in a down state
            - this is because of the pod's `spec.restartPolicy` which is set to `Always` by default (that is, when not set)
            - can override this policy with the values `Never` or `OnFailure`
    - but this doesn't handle complex things like parallel workloads
- **Jobs** are similar to ReplicaSets, in that they specify a nuber of pods deployed out to do a thing
    - however the big difference is that a Job expects those pods to run to completion
- what does a Job definition file look like?
    ```yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
        name: math-add-job
    spec:
        template:
            spec:
                containers:
                  - name: math-add
                    image: ubuntu
                    command: ['expr', '3', '+', '2']
                restartPolicy: never
    ```
    - bunch of things to note:
        - has a `spec.template`, just like ReplicaSet (and therefore Deployment)
        - `spec.template` defines a `spec` for pods and containers
        - specifically setting the pod's `restartPolicy` to `Never`
        - has its own `kind`
        - has a different `apiVersion`
    - get jobs with `kubectl get jobs`
    - if you run `kubectl get pods` you should see the pod in a Completed state
    - see log output by running `kubrctl logs <name-of-pod>`
- how to run multiple pods:
    - set the `spec.completions` property to the number of pods you want to run
        - e.g. `spec.completions: 3` will run 3 pods
        - using this property, k8s will create pods sequentially, on after another, until 3 completions have successfully run
            - so the second one won't be created until the first one is done
            - that also means if the second one errors out, a new "second one" will be attempted until a second completion is achieved
            - **HOWEVER** you have to increase the `BackOffLimit` property in order to stop the k8s master from treating it as an erroneous release
        - big takeaway is that this is not a parallel run by default
    - parrallelism is achieved by adding the `spec.parallelism` property with the number of pods you want to see run in parallel
        - using the example above, if you want all three to run in parallel, you'd set `spec.parallelism: 3`

## CronJobs
- a job that can be scheduled
- e.g. a job that generates a report and sends an email
    - not run by hand, not completed as a one-off
    - scheduled to run periodically
    - schedule takes a lin
- what does a CronJob template look like?
    ```yaml
    apiVersion: batch/v1beta1
    kind: CronJob
    metadata:
        name: reporting-cron-job
    spec:
        schedule: <linux-like-cron-string>
        jobTemplate: <job-template-section>
    ```
    - quick overview of linux-like cron strings
        - five char, space seperated
            - first is the minute of an hour    (0-59)
            - second is the hour of the day     (0-23)
            - third is day of the month         (0-31)
            - fourth is month of the year       (1-21)
            - fifth is day of the week          (0-6)
    - the `spec.jobTemplate` value will be the `spec` section of what would be a normal one-off Job, including the `spec` key
        - end of the day you should have three `spec` sections:
            - `spec` for the CronJob
            - `spec.jobTemplate.spec` for the Job that will run on Cron
            - `spec.jobTemplate.spec.template.spec` defines the pod template