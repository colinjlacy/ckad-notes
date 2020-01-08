# Volumes

## Volumes in Docker
- docker containers are meant to be transient
    - called upon when required to process data
    - destroyed once finished
    - same is true for the data *living* in the container
        - data is destroyed along with the container
- to persist data that lives in a container, we attach a volume to the container when created
    - data processed is now placed in the volume
    - when the container dies, the data remains

## Volumes in k8s
- volumes are attached to pods
    - as pods are the atomic structure in k8s
    - even after the pod is deleted, the data remains

## Volumes and Mounts
- simple example for a pod volume mount:
    ```yaml
    spec:
        containers:
          - image: alpine
            name: alpine
            volumeMounts:
              - mountPath: /opt  # path in the pod
                name: data-volume
        volumes:
          - name: data-volume
            path: /data     # path on the host
            type: Directory
    ```
    - simple enough:
        - a volume is created for the host's `/data` path
        - it is mounted for the pod at `/opt`
    - caveats:
        - won't work for a multi-node deployment
            - data would be stored in each node's `/data` path, thus making for disparate data
    
### Volume Types
- because there are complexity challenges in managing your own volume, k8s supports multiple types of storage-as-a-service
- e.g.:
    - to store on an AWS EBS volume, the `spec.volumes[]` value would look like this:
        ```yaml
        volumes:
          - name: data-volume
            awsElasticBlockStore:
                volumeID: <volume-id>
                fsType: ext4
        ```

## Persistent Volumes
- a resource definition in k8s
- definition file:
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
        name: pv-voll
    spec:
        accessModes:
          - ReadWriteOnce
        capacity:
            storage: 1Gi
        hostPath: # path on the host, not to be used in multi-node deploy
            path: /tmp/data
    ```
    - `spec.accessModes`:
        - defines how a volume should be mounted on the host
        - options are:
            - `ReadOnlyMany`
            - `ReadWriteOnce`
            - `ReadWriteMany`
- for a prod, multi-node cluster, you would replace `spec.hostPath` with whatever vendor-specific storage type you mean to target:
    ```yaml
    spec:
        accessModes:
          - ReadWriteOnce
        capacity:
            storage: 1Gi
        awsElasticBlockStore:
            volumeID: <volume-id>
            fsType: ext4
    ```
- deploy with the standard `kubectl apply -f <some-pv-file>`, and access with `kubectl get persistentvolume`

## Persistent Volume Claims
- PersistentVolumes and PersistentVolumeClaims are two seperate objects in the k8s namespace
    - PersistentVolumes are generally created by an admin
    - PVClaims are generally created by a dev to claim access to a persistent volume
- once a claim is created, k8s binds the volumes to claims based on the params in the claim
    - tries to find a volume with the correct details to match the claim
        - **NOTE** that you can still use `labels` in the PV and `selector.matchLabels` in the PVClaim to use a specific volume
    - claims/volumes have a 1-to-1 relationship
        - a claim can only be placed on a single volume
        - only one claim can be attached to any one volume
- if no volumes are available, a PVC will remain `pending` until a volume becomes available
- PVC spec file:
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
        name: myclaim
    spec:
        accessModes:
          - ReadWriteOnce
        resources:
            requests:
                storage: 500Mi
    ```
    - note that there is nothing in this spec that targets a specific volume
        - in the two examples above, we have a volume or 1GB, and a PVC for 500MB
        - in the case where that is the only volume available, the claim would get attached to that only volume
            - the extra capacity would go unused
- how to attach a claim to a pod:
    ```yaml
    spec: 
        containers: ...
        volumes:
          - name: mypd
            persistentVolumeClaim:
                claimName: myclaim
    ```
    - this would attach the claim listed above
    - you can also use this in a ReplicaSet or Deployment to grant multiple pods access to the same claim
        - put it in the pod template spec
        - this is how Deployments have common access to persistent data
