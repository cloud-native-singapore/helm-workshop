Helm Chart Conventions

Coding conventions for Helm Charts at Honestbee. This document is subject to change based on the ongoing [Helm best practices discussion](https://github.com/kubernetes/charts/issues/44)

## Kubernetes Chart requirements

Adapted from the [Official Kubernetes Chart Requirements](https://github.com/kubernetes/charts/blob/master/CONTRIBUTING.md)

### Technical requirements

Recommended chart requirements, if any of these are not met - kindly highlight in `README.md`

* All Chart dependencies should also be submitted independently
* Must pass the linter (`helm lint`)
* Must successfully launch with default values (`helm install .`)
    * All pods go to the running state
    * All services have at least one endpoint
* Must link source GitHub repositories for images used in the Chart
* Images should not have any major security vulnerabilities
* Must be up-to-date with the latest stable Helm/Kubernetes features
    * Use Deployments in favor of ReplicationControllers
* Should follow Kubernetes best practices
    * Include Health Checks wherever practical
    * Allow configurable [resource requests and limits](http://kubernetes.io/docs/user-guide/compute-resources/#resource-requests-and-limits-of-pod-and-container)
    * [Container Best Practices](http://docs.projectatomic.io/container-best-practices/)
* Provide a method for data persistence (if applicable)
* Support application upgrades
* Allow customization of the application configuration
* Provide a secure default configuration
* Do not leverage alpha features of Kubernetes
* Includes a [NOTES.txt](https://github.com/kubernetes/helm/blob/master/docs/charts.md#chart-license-readme-and-notes) explaining how to use the application after install

### Documentation requirements

* Must include an in-depth `README.md`, including:
    * Short description of the Chart
    * Any prerequisites or requirements
    * Customization: explaining options in `values.yaml` and their defaults
* Must include a short `NOTES.txt`, including:
    * Any relevant post-installation information for the Chart
    * Instructions on how to access the application or service provided by the Chart

## Honestbee Guidelines

### General

- Use yaml with 2 spaces indentation
- Use single `#` to document parameters in `values.yaml`
- `values.yaml` must be included as it serves documentation purposes
- ensure not to commit any secrets in `values.yaml`

### File Naming convention

Each resource type should have its own file under the `templates/` directory (do not stream multiple yaml documents into a single yaml file). These files have the following naming convention:

1. Prefix the files with the name of the Chart component (i.e an `elasticsearch` cluster has `master`, `client` and `data` components).

1. Use abbreviations of Kubernetes Resource Types (if they exist):

   | Abbreviation | Full name               |
   | ------------ | ----------------------- |
   | svc          | service                 |
   | deploy       | deployment              |
   | cm           | configmap               |
   | secret       | secret                  |
   | ds           | daemonset               |
   | rc           | replicationcontroller   |
   | petset       | petset                  |
   | po           | pod                     |
   | hpa          | horizontalpodautoscaler |
   | ing          | ingress                 |
   | job          | job                     |
   | limit        | limitrange              |
   | ns           | namespace               |
   | pv           | persistentvolume        |
   | pvc          | persistentvolumeclaim   |
   | sa           | serviceaccount          |

For example:

```bash
master-deploy.yaml  # deployment of a master component
web-deploy.yaml     # deployment of a web component
web-svc.yaml        # service definition of a web component
registry-rc.yaml    # replication controller of a registry component
```

### Template Names

Template names are global, even when used in sub-charts. Due to this, template names should always be scoped to their chart.

Wrong:
```mustache
{{- define "fullname" -}}
{{- printf "%s-%s" .Release.Name "kibana" | trunc 63 -}}
{{- end -}}
```

Right:
```mustache
{{- define "kibana.fullname" -}}
{{- printf "%s-%s" .Release.Name "kibana" | trunc 63 -}}
{{- end -}}
```

### Upper or Lower case Variables

Helm Classic used variables starting with an upper case letter.

When Helm was incubated by the Kubernetes project, Kubernetes maintainers however prefer to separate local variables by using lower case letters.

Additionally, lower case variables match the Kubernetes Manifest naming conventions, giving you the ability to include snippets from `values.yaml` directly into your Manifests:

```mustache
template:
    metadata:
        labels:
          app: {{ template "scoped.fullname" . }}
      spec:
        containers:
        - name: {{ template "scoped.fullname" . }}
          image: "{{ .Values.image }}"
          resources:
{{ toYaml .Values.resources | indent 10 }}
```
[Reference](https://github.com/kubernetes/charts/pull/158/files/1a933d69708c826a5351b3d814feb46f926ee065#diff-ce347aed3be8e5790b15443e71eb23ddR45)

**Drawback?**: this requires a default values.yaml to be present with resource default, allowing users to specify additional configuration values using the `-f custom-values.yaml` flag.

Going forward, all variables in Charts written at Honestbee should start with a lower case letter.

### Flat or Nested Configuration Values

Flat names for values are preferred (as advised in the Helm guidelines),
but nested variables can be very useful when used carefully.

At Honestbee, the following rule of thumb can be used:

> If you have multiple parameters related to a single entity and
> at least 1 parameter is mandatory, use nested variables.

Reasoning: you can't omit the root of the nested variable, even if you intend every option in the hierarchy to be optional.

Following examples can be used as a guideline:

1.  Related Parameters of which none are mandatory:

    Nested:
    ```yaml
    Ports:
      http: 5000
      node: 30650
      web: 80
      service: 80
    ```

    vs Flat:
    ```yaml
    httpPort: 5000
    nodePort: 30650
    webPort: 80
    servicePort: 80
    ```

    If every Port has a default, it is better to use the flat version. With the flat version you can completely omit the whole Ports map.

1.  Related (repeated) Parameters of which at least 1 is mandatory

    Flat:
    ```yaml
    masterReplicaCount: 1
    masterMemRequest: 1Gi
    masterMemLimit: 2Gi
    masterImage: master
    masterIsSchedulable: true
    workerReplicaCount: 3
    workermemRequest: 1Gi
    workermemLimit: 1Gi
    workerImage: worker
    ```

    vs Nested:
    ```yaml
    master:
      replicaCount: 1
      isSchedulable: true
      memRequest: 1Gi
      memLimit: 2Gi
      image: master
    worker:
      replicaCount: 3
      memRequest: 1Gi
      memLimit: 1Gi
      image: worker
    ```

    Nested variables can improve readability and maintainability. In
    this case, at least 1 key for each map needs to exist, with all
    defaults this may look like this:

    ```yaml
    master:
      replicaCount: 1
    worker:
      replicaCount: 3
    ```

### Labels

A cluster can be used to deploy multiple applications and services. To be able
to correctly filter components in services and replication controllers we
should define labels to achieve this.

This section defines a set of labels that should be used.

#### heritage (mandatory)

All manifests must specify the `{{ .Release.Service | quote }}` label in the metadata section. This enables querying a cluster to list all components deployed using Helm:

```bash
kubectl get deploy,svc -l heritage=Tiller
```

#### release (mandatory)

All manifests must specify the `{{ .Release.Name | quote }}` label in the metadata section. This enables querying a cluster to list all components deployed for a particular release:

```bash
kubectl get deploy,svc -l release=rolling-badger
```

#### chart (mandatory)

All manifests must specify the `{{ .Chart.Name }}-{{ .Chart.Version }}` label
in the metadata section. This enables identifying the chart and version of
cluster resources.

```bash
kubectl describe deploy rolling-badger-elasticsearch | grep chart
```

Note: `helm list` and `helm status <release>` give you the same?

#### app (mandatory)

All manifests must specify the app label in the metadata section.

```bash
kubectl get deploy,svc -l app=elasticsearch
```

When the manifests deploy more than one resource, the `app` label should be defined to group all the components under one label. This enables querying a cluster to list all components deployed for a particular app, while ensuring services can be created to expose just a particular component where needed:

For example, if in a GitLab deployment `gitlab-workhorse` and `sidekiq` are defined in different manifests, these manifests should both have the `app: gitlab` label grouping the components together.

#### component (optional)

Custom labels can be be defined to provide fine-grained filtering of pods in services and replication controllers. For example, a MariaDB pod can have a `component:` label to indicate if it’s a `master` or `slave` allowing the Services to correctly filter Pods.

#### Split Release, App & Component in selectors

Split release and component from app label and add release to selector.
i.e - don't use `fullname` templates for labels (do use `fullname` templates
for resource names).

Wrong

```mustache
  selector:
    app: {{ printf "%s-%s-%s" .Release.Name .Chart.Name "worker" }}
    # or
    app: {{ template "kibana.fullname" }}
```

Right

```mustache
  selector:
    app: {{ .Chart.Name }}
    release: {{ .Release.Name }}
    component: worker
```

#### Order

At Honestbee the preferred order is:

```mustache
    heritage: {{ .Release.Service | quote }}
    release: {{ .Release.Name | quote }}
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    app: {{ template "scoped.fullname" }}
    component: ...
```

### Container Image tags

Use versioned images, avoid using the latest tag.

If the latest tag is used, define `imagePullPolicy: Always`.

Provide links to image source (GitHub or DockerHub) in README.

### Ports

Always define named ports in the `podSpec`. Whenever possible name the ports with the IANA defined service name (eg. [iana?search=http](http://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=http)).

### Pod Lifecycle Management

All `containerSpecs` should define [probes](http://kubernetes.io/docs/api-reference/v1/definitions/#_v1_probe) of type `livenessProbe` and `readinessProbe`. (note: all probes are ran by kubelet in the kubelet network namespace)

Three primitives are available for setting up these probes (only 1 may be specified per probe):

- HTTP check ([`httpGet`](http://kubernetes.io/docs/api-reference/v1/definitions/#_v1_httpgetaction))

  Performs an HTTP Get against the container’s IP address on a specified port and path expecting on success that the response has a status code greater than or equal to `200` and less than `400`.

- Container Execution Check ([`exec`](http://kubernetes.io/docs/api-reference/v1/definitions/#_v1_execaction))

  Executes a specified command inside the container expecting on success that the command exits with status code `0`.

- TCP Socket Check ([`tcpSocket`](http://kubernetes.io/docs/api-reference/v1/definitions/#_v1_tcpsocketaction))

  Performs a tcp check against the container’s IP address on a specified port expecting on success that the port is open.

#### Liveness Probe

A liveness probe checks if the container is running and responding normally. If the liveness probe fails, the container is terminated and subject to the pod's [RestartPolicy](http://kubernetes.io/docs/user-guide/pod-states/#restartpolicy). The RestartPolicy is applicable to all containers in the pod (default: `Always`).

Below is an example `livenessProbe` for a mariadb podSpec.

```yaml
...
    spec:
      containers:
      - name: mariadb
        image: bitnami/mariadb:5.5.46-0-r01
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
          initialDelaySeconds: 30
          timeoutSeconds: 5
```

Here an `exec` probe is setup to check the status of the MariaDB server at 5 second intervals using the `mysqladmin ping` command. The Pod will be restarted if the command returns an error for any reason.

To allow the container to boot up before the liveness probes start, the `initialDelaySeconds` should be set high enough to allow the container to start or the container will be prematurely terminated by the probe (default `failureThreshold` before termination is 3).

To better understand the flow of container states, consider a `Running` pod with 2 containers. Assume container 1 terminates with `Failure`.

- if `restartPolicy` is:
  - `Always`: restart container, pod stays `Running`
  - `OnFailure`: restart container, pod stays `Running`
  - `Never`: Pod stays `Running`
- When container 2 exists with `Failure`...
  - if `restartPolicy` is:
    - `Always`: restart container, pod stays `Running`
    - `OnFailure`: restart container, pod stays `Running`
    - `Never`: pod becomes `Failed`
      - if running under a controller, pod will be recreated elsewhere

#### Readiness Probe

To ensure traffic is sent only to pods once a probe succeeds, ensure a `readinessProbe` is defined in the `containerSpec`.

A readiness probe indicates whether the container is ready to service requests. If the readiness probe fails, the endpoints controller will remove the pod’s IP address from the endpoints of all services that match the pod until readiness probes succeed again.

The setup of a readiness probe is similar to a liveness probe:

```yaml
...
    spec:
      containers:
      - name: mariadb
        image: bitnami/mariadb:5.5.46-0-r01
        ports:
        - name: mysql
          containerPort: 3306
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
          initialDelaySeconds: 30
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command:
            - mysqladmin
            - ping
          initialDelaySeconds: 5
          timeoutSeconds: 1
```

Because we want the Pod to start receiving traffic as soon as it's ready, a lower `initialDelaySeconds` value is specified.

A lower `timeoutSeconds` ensures that the Pod does not receive any traffic as soon as it becomes unresponsive. If the failure was temporary, the Pod would resume normal functioning after it has recovered.

Note: Allowing different probes for liveness and readiness, provides a way to define applications with fine grained traffic control for maintenance windows.

### Volumes

Volumes are required to persist container data and a method for data persistence (if applicable) should be specified.

Example (works for AWS, GCE, minikube):
```mustache
{{- if .Values.persistence.enabled -}}
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: {{ template "scoped.fullname" . }}
  labels:
    app: {{ template "scoped.fullname" . }}
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"
  annotations:
    volume.alpha.kubernetes.io/storage-class: {{ .Values.persistence.storageClass | quote }}
spec:
  accessModes:
    - {{ .Values.persistence.accessMode | quote }}
  resources:
    requests:
      storage: {{ .Values.persistence.size | quote }}
{{- end -}}
```

with `values.yaml`:

```yaml
## Enable persistence using Persistent Volume Claims
## ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
##
persistence:
  enabled: true
  storageClass: generic
  accessMode: ReadWriteOnce
  size: 8Gi
```

## Services

To expose the web applications running in a Kubernetes cluster, a `Service` resource type needs to be created. Service objects are used for discovering and routing traffic to each `Pod` resource. The set of Pods targeted by a Service is determined by label selectors. The following is a sample of a simple `Service.yaml` manifest.

Note that the `chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"` label is not used in the label selectors as its inclusion will not allow for rolling updates.

Also note that if multiple deployments of the same chart within the same namespace is required, include Release information in the `fullname` template and use the `fullname` template in the application selector.

Lastly, named ports should be used to specify the service `targetPort`, allowing us to change the port numbers in the `podSpec` without the need to change the Service manifests.

### Secrets and ConfigMaps

Objects of type ConfigMap are intended to hold configuration values. Putting this information in a ConfigMap is more flexible than adding it to the Manifest template or baking it into a docker image.

Objects of type Secret are intended to hold sensitive information, such as passwords, OAuth tokens, and ssh keys. Secrets allow the use of specialized controllers to store data encrypted.

Another difference between configMaps and Secrets is that volumeMounts for configMaps update (with inotify) in existing pods, while Secrets do not change by design and require pods to be re-created. (See [chart tips & tricks](https://github.com/kubernetes/helm/blob/master/docs/charts_tips_and_tricks.md#automatically-roll-deployments-when-configmaps-or-secrets-change))

Example:
```mustache
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ template "scoped.component-configmap" . }}
  labels:
    heritage: {{ .Release.Service | quote }}
    release: {{ .Release.Name | quote }}
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    app: {{ template "scoped.fullname" . }}
data:
{{ (.Files.Glob "files/component/*").AsConfig | indent 2 }}
```

Note `AsSecrets` and `AsConfig` [utility functions](https://github.com/kubernetes/helm/blob/master/docs/chart_template_guide/accessing_files.md#configmap-and-secrets-utility-functions).

### Documenting Parameters

The README for each chart should document all parameters. A helpful one-liner to `grep` all `Values` from a chart templates directory:

```bash
grep -Rohe '.Values.[a-zA-z0-9.]*' ./templates | cut -d. -f3- | sort -u
```

(remove `-h` flag to show file where parameter is used)