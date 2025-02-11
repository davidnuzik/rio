# Riofile


Rio works with standard Kubernetes YAML files. Rio additionally supports a more user-friendly `docker-compose`-style config file called `Riofile`.
This allows you define rio services, routes, external services, configs, and secrets.

For example, here is an nginx application:

```yaml
configs:
  conf:
    index.html: |-
      <!DOCTYPE html>
      <html>
      <body>

      <h1>Hello World</h1>

      </body>
      </html>
services:
  nginx:
    image: nginx
    ports:
    - 80/http
    configs:
    - conf/index.html:/usr/share/nginx/html/index.html
```

Once you have defined `Riofile`, simply run `rio up`.
If you made any change to the `Riofile`, re-run `rio up` to pick up the change.
To use a file not named "Riofile" use `rio up -f nginx.yaml`.

#### Watching Riofile

You can setup Rio to watch for Riofile changes in a Github repository and deploy Riofile changes automatically. For example:
```bash
$ rio up https://github.com/username/repo
```

By default, Rio will poll the branch in 15 second intervals, but this can be configured to use a webhook instead. See [Webhook docs](./webhooks.md) for info.

#### Riofile reference

```yaml
# Configmap
configs:
  config-foo:     # specify name in the section
    key1: |-      # specify key and data in the section
      {{ config1 }}
    key2: |-
      {{ config2 }}

# Externalservices
externalservices:
  foo:
    ipAddresses: # Pointing to external IP addresses
    - 1.1.1.1
    - 2.2.2.2
    fqdn: www.foo.bar # Pointing to fqdn
    service: $namespace/$name # Pointing to services in another namespace

# Service
services:
  service-foo:

    # Scale setting
    scale: 2 # Specify scale of the service. If you pass range `1-10`, it will enable autoscaling which can be scale from 1 to 10. Default to 1 if omitted

    # Revision setting
    app: my-app # Specify app name. Defaults to service name. This is used to aggregate services that belongs to the same app.
    version: v0 # Specify version name. Defaults to v0.
    weight: 80 # Weight assigned to this revision. Defaults to 100.

    # Traffic rollout config. Optional
    rollout:
      increment: 5 # traffic percentage increment(%) for each interval.
      interval: 2 # traffic increment interval(seconds).
      pause: false # whether to perform rollout or not

    # Autoscaling setting. Only required if autoscaling is set
    concurrency: 10 # specify concurrent request each pod can handle(soft limit, used to scale service)

    # Container configuration
    image: nginx # Container image. Required if not setting build
    imagePullPolicy: always # Image pull policy. Options: (always/never/ifNotProsent), defaults to ifNotProsent.
    build: # Setting build parameters. Set if you want to build image for source
      repo: https://github.com/rancher/rio # Git repository to build. Required
      branch: master # Git repository branch. Required
      revision: v0.1.0 # Revision digest to build. If set, image will be built based on this revision. Otherwise it will take head revision in repo. Also if revision is not set, it will be served as the base revision to watch any change in repo and create new revision based changes from repo.
      args: # Build arguments to pass to buildkit https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact. Optional
      - foo=bar
      dockerfile: Dockerfile # The name of Dockerfile to look for.  This is the full path relative to the repo root. Defaults to `Dockerfile`.
      context: ./  # Docker build context. Defaults to .
      noCache: true # Build without cache. Defaults to false.
      imageName: myname/image:tag # Specify custom image name(excluding registry name). Default name: $namespace/name:$revision_digest
      pushRegistry: docker.io # Specify push registry. Example: docker.io, gcr.io. Defaults to localhost registry.
      pushRegistrySecretName: secretDocker # Specify secret name for pushing to docker registry. [link](#set-custom-build-arguments-and-docker-registry)
      stageOnly: true # If set, newly created revision will get any traffic. Defaults to false.
      webhookSecretName: secretGithub # Specify the github secret name. Used to create Github webhook, the secret key has to be `accessToken`
      cloneSecretName: secretGit # Specify secret name for checking our git resources
      pr: true # Enable pull request feature. Defaults to false
      timeout: 10 # build timeout setting in seconds
    command: # Container entrypoint, not executed within a shell. The docker image's ENTRYPOINT is used if this is not provided.
    - echo
    args: # Arguments to the entrypoint. The docker image's CMD is used if this is not provided.
    - "hello world"
    workingDir: /home # Container working directory
    ports: # Container ports, format: `$(servicePort:)containerPort/protocol`. Required if user wants to expose service through gateway
    - 8080:80/http,web # Service port 8080 will be mapped to container port 80 with protocol http, named `web`
    - 8080/http,admin,internal=true # Service port 8080 will be mapped to container port 8080 with protocol http, named `admin`, internal port(will not be exposed through gateway)
    env: # Specify environment variable
    - POD_NAME=$(self/name) # Mapped to "metadata.name"
    #
    # "self/name":           "metadata.name",
    # "self/namespace":      "metadata.namespace",
    # "self/labels":         "metadata.labels",
    # "self/annotations":    "metadata.annotations",
    # "self/node":           "spec.nodeName",
    # "self/serviceAccount": "spec.serviceAccountName",
    # "self/hostIp":         "status.hostIP",
    # "self/nodeIp":         "status.hostIP",
    # "self/ip":             "status.podIP",
    #
    cpus: 100m # Cpu request, format 0.5 or 500m. 500m = 0.5 core. If not set, cpu request will not be set. https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/
    memory: 100Mi # Memory request. 100Mi, available options. If not set, memory request will not be set. https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/
    secrets: # Specify secret to mount. Format: `$name/$key:/path/to/file`. Secret has to be pre-created in the same namespace
    - foo/bar:/my/password
    configs: # Specify configmap to mount. Format: `$name/$key:/path/to/file`.
    - foo/bar:/my/config
    livenessProbe: # LivenessProbe setting. https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
      httpGet:
        path: /ping
        port: "9997" # port must be string
      initialDelaySeconds: 10
    readinessProbe: # ReadinessProbe https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
      failureThreshold: 7
      httpGet:
        path: /ready
        port: "9997" # port must be string
    stdin: true # Whether this container should allocate a buffer for stdin in the container runtime
    stdinOnce: true # Whether the container runtime should close the stdin channel after it has been opened by a single attach. When stdin is true the stdin stream will remain open across multiple attach sessions.
    tty: true # Whether this container should allocate a TTY for itself
    runAsUser: 1000 # The UID to run the entrypoint of the container process.
    runAsGroup: 1000 # The GID to run the entrypoint of the container process
    readOnlyRootFilesystem: true # Whether this container has a read-only root filesystem
    privileged: true # Run container in privileged mode.

    nodeAffinity: # Describes node affinity scheduling rules for the pod.
    podAffinity:  # Describes pod affinity scheduling rules (e.g. co-locate this pod in the same node, zone, etc. as some other pod(s)).
    podAntiAffinity: # Describes pod anti-affinity scheduling rules (e.g. avoid putting this pod in the same node, zone, etc. as some other pod(s)).

    hostAliases: # Hostname alias
      - ip: 127.0.0.1
        hostnames:
        - example.com
    hostNetwork: true # Use host networking, defaults to False. If this option is set, the ports that will be used must be specified.
    imagePullSecrets: # Image pull secret https://kubernetes.io/docs/concepts/containers/images#specifying-imagepullsecrets-on-a-pod
    - secret1
    - secret2

    # Containers: Specify sidecars. Other options are available in container section above, this is limited example
    containers:
    - init: true # Init container
      name: my-init
      image: ubuntu
      args:
      - "echo"
      - "hello world"


    # Permission for service
    #
    #   global_permissions:
    #   - 'create,get,list certmanager.k8s.io/*'
    #
    #   this will give workload abilities to **create, get, list** **all** resources in api group **certmanager.k8s.io**.
    #
    #   If you want to hook up with an existing role:
    #
    #
    #   global_permissions:
    #   - 'role=cluster-admin'
    #
    #
    #   - `permisions`: Specify current namespace permission of workload
    #
    #   Example:
    #
    #   permissions:
    #   - 'create,get,list certmanager.k8s.io/*'
    #
    #
    #   This will give workload abilities to **create, get, list** **all** resources in api group **certmanager.k8s.io** in **current** namespace.
    #
    #   Example:
    #
    #   permissions:
    #   - 'create,get,list /node/proxy'
    #
    #    This will give subresource for node/proxy

    # Optional, will created and mount serviceAccountToken into pods with corresponding permissions
    global_permissions:
    - 'create,get,list certmanager.k8s.io/*'
    permissions:
    - 'create,get,list certmanager.k8s.io/*'

# Router
routers:
  foo:
    routes:
      - match: # Match rules, the first rule matching an incoming request is used
          path: # Match path, can specify regxp, prefix or exact match
            exact: /v0
          # prefix: /bar
          # regxp: /bar.*
          methods:
          - GET
          headers:
            - name: FOO
              value:
                regxp: /bar.*
                #prefix: /bar
                #exact: /bar
        to:  # Specify destination
          - app: myapp
            version: v1
            port: 80
            namespace: default
            weight: 50
          - app: myapp
            version: v2
            port: 80
            namespace: default
            weight: 50
        redirect: # Specify redirect rule
          host: www.foo.bar
          path: /redirect
        rewrite:
          host: www.foo.bar
          path: /rewrite
        headers: # Header operations
          add:
            - name: foo
              value: bar
          set:
            - name: foo
              value: bar
          remove:
          - foo
        fault:
          percentage: 80 # Inject fault percentage(%)
          delayMillis: 100 # Adding delay before injecting fault (millseconds)
          abortHTTPStatus: 502 # Injecting http code
        mirror:   # Sending mirror traffic
          app: q
          version: v0
          namespace: default
        timeoutSeconds: 1 # Setting request timeout (milli-seconds)
        retry:
          attempts: 10 # Retry attempts
          timeoutSeconds: 1 # Retry timeout (milli-seconds)

# Use Riofile's answer/question templating
template:
  goTemplate: true # use go templating
  envSubst: true # use ENV vars during templating
  questions:  # now make some questions that we provide answers too
  - variable: NAMESPACE
    description: "namespace to deploy to"

# Supply arbitrary kubernetes manifest yaml
kubernetes:
  manifest: |-
    apiVersion: apps/v1
    kind: Deployment
    ....

```


### How to use Rio to deploy an application with arbitary YAML
_for Rio v0.6.0 and greater_

In this example we will see how to define both a normal Rio service and arbitary Kubernetse manifests and deploy both of these with Rio. Follow the quickstart to get Rio installed into your cluster and ensure the output of `rio info` looks similar to this:
```
Rio Version:  >=0.6.0
Rio CLI Version: >=0.6.0
Cluster Domain: enu90s.on-rio.io
Cluster Domain IPs: <cluster domain ip>
System Namespace: rio-system
System Ready State: true
Wildcard certificates: true

System Components:
gateway-v2 status: Ready
rio-controller status: Ready
```

First, lets use a Riofile to define a a basic Rio service.

```
configs:
  conf:
    index.html: |-
      <!DOCTYPE html>
      <html>
      <body>

      <h1>Hello World</h1>

      </body>
      </html>
services:
  nginx:
    image: nginx
    ports:
    - 80/http
    configs:
    - conf/index.html:/usr/share/nginx/html/index.html
```

Next, we can augment this service with the Kubernetes [sample guestbook](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/)


```yaml
configs:
    conf:
        index.html: |-
            <!DOCTYPE html>
            <html>
            <body>

            <h1>Hello World</h1>

            </body>
            </html>
services:
    nginx:
        image: nginx
        ports:
            - 80/http
        configs:
            - conf/index.html:/usr/share/nginx/html/index.html

kubernetes:
    manifest: |-
      apiVersion: v1
      kind: Service
      metadata:
        name: redis-master
        labels:
          app: redis
          tier: backend
          role: master
      spec:
        ports:
        - port: 6379
          targetPort: 6379
        selector:
          app: redis
          tier: backend
          role: master
      ---
      apiVersion: apps/v1 #  for k8s versions before 1.9.0 use apps/v1beta2  and before 1.8.0 use extensions/v1beta1
      kind: Deployment
      metadata:
        name: redis-master
      spec:
        selector:
          matchLabels:
            app: redis
            role: master
            tier: backend
        replicas: 1
        template:
          metadata:
            labels:
              app: redis
              role: master
              tier: backend
          spec:
            containers:
            - name: master
              image: k8s.gcr.io/redis:e2e  # or just image: redis
              resources:
                requests:
                  cpu: 100m
                  memory: 100Mi
              ports:
              - containerPort: 6379
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: redis-slave
        labels:
          app: redis
          tier: backend
          role: slave
      spec:
        ports:
        - port: 6379
        selector:
          app: redis
          tier: backend
          role: slave
      ---
      apiVersion: apps/v1 #  for k8s versions before 1.9.0 use apps/v1beta2  and before 1.8.0 use extensions/v1beta1
      kind: Deployment
      metadata:
        name: redis-slave
      spec:
        selector:
          matchLabels:
            app: redis
            role: slave
            tier: backend
        replicas: 2
        template:
          metadata:
            labels:
              app: redis
              role: slave
              tier: backend
          spec:
            containers:
            - name: slave
              image: gcr.io/google_samples/gb-redisslave:v1
              resources:
                requests:
                  cpu: 100m
                  memory: 100Mi
              env:
              - name: GET_HOSTS_FROM
                value: dns
                # If your cluster config does not include a dns service, then to
                # instead access an environment variable to find the master
                # service's host, comment out the 'value: dns' line above, and
                # uncomment the line below:
                # value: env
              ports:
              - containerPort: 6379
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: frontend
        labels:
          app: guestbook
          tier: frontend
      spec:
        # if your cluster supports it, uncomment the following to automatically create
        # an external load-balanced IP for the frontend service.
        # type: LoadBalancer
        ports:
        - port: 80
        selector:
          app: guestbook
          tier: frontend
      ---
      apiVersion: apps/v1 #  for k8s versions before 1.9.0 use apps/v1beta2  and before 1.8.0 use extensions/v1beta1
      kind: Deployment
      metadata:
        name: frontend
      spec:
        selector:
          matchLabels:
            app: guestbook
            tier: frontend
        replicas: 3
        template:
          metadata:
            labels:
              app: guestbook
              tier: frontend
          spec:
            containers:
            - name: php-redis
              image: gcr.io/google-samples/gb-frontend:v4
              resources:
                requests:
                  cpu: 100m
                  memory: 100Mi
              env:
              - name: GET_HOSTS_FROM
                value: dns
                # If your cluster config does not include a dns service, then to
                # instead access environment variables to find service host
                # info, comment out the 'value: dns' line above, and uncomment the
                # line below:
                # value: env
              ports:
              - containerPort: 80
```

Typically you would track your Riofile with some form of VCS but for now simply save it in a local directory.

Next, run `rio up` in that directory.

You can watch Rio service come up with `rio ps` and the Kubernetes deployments with `kubectl get deployments -w`.

You can check that the sample service came up by going to the endpoint given by `rio ps`
```
NAME      IMAGE     ENDPOINT                                          SCALE     APP       VERSION    WEIGHT    CREATED       DETAIL
nginx     nginx     https://nginx-2c21baa1-default.enu90s.on-rio.io   1         nginx     2c21baa1   100%      4 hours ago
```

We can use Rio to expose the service and provision a LetsEncrypt certificate for it.

` rio router add guestbook to frontend,port=80 `

This will create a route to the service and create an endpoint.

```
rio endpoints
NAME        ENDPOINTS
nginx       https://nginx-default.enu90s.on-rio.io
guestbook   https://guestbook-default.enu90s.on-rio.io
```

We can now access this endpoint over encrypted https!


#### Using answer file 

Rio allows the user to leverage an answer file to customize `Riofile`. Go template and [envSubst](https://github.com/drone/envsubst) can used to apply answers.

Answer file is a yaml manifest with key-value pairs:

```yaml
FOO: BAR
```

For example, to use go templating:

```yaml
services:
{{- if eq .Values.FOO "BAR" }
  demo:
    image: ibuilthecloud/demo:v1
{{- end}

template:
  goTemplate: true # use go templating
  envSubst: true # use ENV vars during templating  
  questions:  # now make some questions that we provide answers too
  - variable: FOO
    description: ""
```

Rio also supports a bash style envsubst replacement, with the following format:

```yaml
${var^}
${var^^}
${var,}
${var,,}
${var:position}
${var:position:length}
${var#substring}
${var##substring}
${var%substring}
${var%%substring}
${var/substring/replacement}
${var//substring/replacement}
${var/#substring/replacement}
${var/%substring/replacement}
${#var}
${var=default}
${var:=default}
${var:-default}
```

A Riofile example with envsubst:

```yaml
services:
  demo:
    image: ibuilthecloud/demo:v1
    env:
    - FOO=${FOO}
```
