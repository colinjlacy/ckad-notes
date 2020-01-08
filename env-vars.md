# Environment Variables

## Most Direct Way
- (which we should never do)
- in the pod spec, we can list a property called `spec.containers[].env`
    - `spec.containers[].env` is an array of values
    - each value contains a `name` and (again, doing this directly) a `value`
        - the more proper way to do this is using `valueFrom` or `envFrom`, which we'll talk about later

## ConfigMaps
- used to pass configuration data in the form of key-value pairs as referenceable resources
- 2 phases:
    - create the config map
        - imperative way - using command line
            - `kubectl create configmap <config-name> --from-literal=<key>=<value>`
            - `kubectl create configmap <config-name> --from-file=<path-to-file>`
        - declarative way - using resource files
            - create a definition file:
                - required fields:
                    - `apiVersion: v1`
                    - `kind: ConfigMap`
                    - `metadata`
                        - `name`
                    - `data`: the dict of key-value pairs that will make up your env vars, e.g:
                        - `APP_COLOR: blue`
                        - note that this does not take the form of a list in YAML, since each key its unique
    - inject it into the pod
        - use the `spec.containers[].envFrom` property
            - a list of config map references: `spec.containers[].envFrom[].configMapRef`
            - each `configMapRef` entry has a `name` property
            - pulls in the entire ConfigMap's data set as env vars
        - use the `spec.containers[].env[].valueFrom` property
            - corresponds to a `spec.containers[].env[].name` property, as mentioned above in the direct, hard-coded way
            - pulls an individual value from a ConfigMap by map name and key reference, e.g.:
                ```yaml
                env:
                  - name: APP_COLOR
                    valueFrom: 
                        configMapKeyRef:
                            name: app-color
                            key: APP_COLOR
                ```
- can create as many config maps as you need for your purposes
    - e.g. create a config map for your application, for your DB port and configs, for your redis, etc.
- to view data in config map, use the `kubectl describe` command
    - key-value pairs are listed in the `DATA` section of the output

## Secrets
- used to store sensative informations, like passwords or keys
    - similar to config maps, but stored in an encoded format
- same two steps to use secrets as with configmaps
    - create the secret
        - imperative way, without using a definition file
            - `kubectl create secret generic <secret-name> --from-literal=<key>=<value>`
            - `kubectl create secret generic <secret-name> --from-file=<path-to-file>`
            - NOTE: in this approach, the values are not base64 encoded when entered
        - declarative way, using a definition file
            - `apiVersion: v1`
            - `kind: Secret`
            - `metadata:`
                - `name: <name>`
            - `data:`: key-value pairs of env var secrets, encoded/hashed before uploading, e.g.:
                - `DB_HOST: bXlzcWw=`
            - how to encode the string you're trying to hash?
                - on a linux host: `echo -n "<string-to-hash>" | base64`
    - apply the secret to a pod
        - again, use `envFrom` list, but this time, an entry is `secretRef` with a `name` of a stored secret
        - or, can do single keys, using the method described above with configmaps, but using the property `secretKeyRef` instead of `configMapKeyRef`
        - or, you can mount it as a volume on the container
            ```yaml
            volumes:
            - name: app-secret-volume
              secret:
                secretName: <name-of-secret>
            ```
            - each key will be a file stored at `/opt/<volume-name>`
                - so in the above example, `/opt/app-secret-volume`
                - if there was a key called `DB_PASSWORD`:
                    - it would now be a file located at `/opt/app-secret-volume/DB_PASSWORD` on the container
                    - its value would be the only content
- to view secret, run `kubectl describe secret/<secret-name>`
    - will show the keys, but not the values
- to view secret values, run ``kubectl describe secret/<secret-name> -o yaml`
    - will show full secret description, but values will be base64 encoded
    - how to decode a base64 string?
        - `echo -n "<string-to-hash>" | base64 --decode`
    