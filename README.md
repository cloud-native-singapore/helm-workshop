# Cloud Native Singapore - Helm workshop

This repository contains the guidelines for a Workshop on writing Helm Charts.

This is meant to be a follow up on the [Kubernetes Workshop](https://github.com/cloud-native-singapore/kubernetes-workshop)

Slides are [available on google slides](https://docs.google.com/presentation/d/1GM6L8XWEHzeI4QUaJ0k7pJSgXcRS9UugPTSO64Mgp4A/edit?usp=sharing)


## Demo Script

### Refresher

In the previous workshop we became familiar with creating & exposing Deployments:

```bash
# Create the nginx Deployment
kubectl run nginx --image=nginx:1.10-alpine --port=80

# Create the nginx service routing traffic to nginx containers
kubectl expose deploy nginx --target-port=80 --type=NodePort

# Integrate with GCP to route traffic to k8s nodes into nginx service
kubectl create -f code/basic/ingress.yaml
```

The Ingress integration ultimately provisions a Backend Service resource in GCloud...
```bash
gcloud compute backend-services list
```

Allowing us to access nginx externally as soon as the k8s nodes are recognized
as healthy by the load balancer:
```bash
$ gcloud compute backend-services get-health <backend-service>
kind: compute#backendServiceGroupHealth
healthStatus:
  - healthState: HEALTHY
    instance: https://www.googleapis.com/compute/v1/...
    port: 30780
  - healthState: HEALTHY
    instance: https://www.googleapis.com/compute/v1/...
    port: 30780
  - healthState: HEALTHY
    instance: https://www.googleapis.com/compute/v1/...
    port: 30780
```

### Infrastructure as Code

The above commands were all imperative commands, ideally we store the desired
state and collaborate on it using source control.

Let's extract the Manifests for the Kubernetes resources we created:

```bash
kubectl get deploy nginx -o yaml > nginx-deploy.yaml
kubectl get svc nginx -o yaml > nginx-svc.yaml

# use :n and :p to navigate files
less nginx-*
```

It is now a good idea to store these manifests under source control (and handcraft them to our liking).

If somebody accidentally deletes everything:
```bash
kubectl delete deploy,svc nginx
```

We can quickly re-create the resources from source control as well as
collaborate on changing the setup.

Let's say we want to use a ConfigMap to serve `index.html`:
```bash
less code/basic/nginx-cm.yaml code/basic/nginx-deploy.yaml
```

> **Note**: This is just an example, not a guideline on how to deploy static sites on Kubernetes!

re-creating the resources in Kubernetes is quick:
```bash
kubectl create -f code/basic/nginx-cm.yaml,code/basic/nginx-svc.yaml,code/basic/nginx-deploy.yaml
```

> **Note**: We could also concatenate all yaml documents into a single document stream (1 file)
or create resources based on all manifests contained within 1 directory:

>```bash
>kubectl create -f code/basic/
>```

This is good, but it can be better...

### Templating and Release Management

The first thing we may want to do is provide the ability to template parts of the manifests.


For example to:
- Change the tag for the image during resource and deploy different versions using the same manifest files. Or, ...
- Modify the contents of the ConfigMap without having to worry about wrapping it in
  yaml...

[Helm](https://github.com/kubernetes/helm) comes with a very powerfull templating
engine (which is just one of it's features).

First, follow the [Helm Installation Guide](https://github.com/kubernetes/helm/blob/master/docs/install.md)

1. Download your [desired version](https://github.com/kubernetes/helm/releases)
1. Unpack it (`tar -zxvf helm-v2.0.0-linux-amd64.tgz`)
1. Find the helm binary in the unpacked directory, and move it to its desired destination
   (`mv linux-amd64/helm /usr/local/bin/helm`)
1. Initialize Helm
   ```bash
   helm init
   ```

> Note: or use Homebrew `brew install kubernetes-helm` at your own risk

Now, use the [Simple Nginx](code/simple-nginx) Chart to re-deploy
our nginx service:

```bash
$ helm install -n simple-nginx-1-10 code/simple-nginx/
```

List all installed charts
```bash
$ helm ls
```

Installation was fast and simple. We can also remove all resources much easier
with a single command:

```bash
$ helm delete simple-nginx-1-10
```

> **Note**: Release names can not be re-used unless they are `--purge` deleted

> ```bash
> $ helm ls --all
> NAME              REVISION   UPDATED                    STATUS  CHART
> simple-nginx-1-10 1          Thu Mar  2 14:21:44 2017   DELETED simple-nginx-0.0.1
> ```

In this simple chart we have:

- A `Chart.yaml` at the root

  This provides information about the chart which can be used for searching.

- A folder full of `templates`

  Which are based on golang templates with 50+ addon functions to render based
  user provided on values

- A `values.yaml` file to provide values for the templates

  Currently only 1 parameter is provided and the default value is documented
  in the [values.yaml](code/simple-nginx/values.yaml) file at the root of the directory.

This chart also demonstrate the ability to include files allowing us to
modify the [html ConfigMap](code/simple-nginx/html/) for illustration purposes.

We can demonstrate these changes as follows:
```
vim code/simple-nginx/html/index.html
helm install -n simple-nginx-1-11 --set tag=1.11-alpine code/simple-nginx/
```

### Advanced Templating and User Notes

With the current state of the chart, if we try to install the simple-nginx chart multiple times,
we will get errors due to name conflicts between the Kubernetes resources.

We can use the Templating features to template out the names as follows:

```yaml
# Templates/deploy.yaml
kind: Deployment
metadata:
  name: {{ printf "%s-%s" .Release.Name .Values.name | trunc 32 }}
...

# Templates/svc.yaml
kind: Service
metadata:
  name: {{ printf "%s-%s" .Release.Name .Values.name | trunc 32 }}
```

But this is very verbose, and if we have to change anything (support longer names, for example), the
change is spread throughout the repository and errors may be made. (not very DRY)

To help with this, template helpers can be defined:

```yaml
# _helpers.tpl
{{- define "fullname" -}}
{{- printf "%s-%s" .Release.Name .Values.name | trunc 63 -}}
{{- end -}}
```

And used to avoid repitition:
```yaml
# Templates/deploy.yaml
kind: Deployment
metadata:
  name: {{ template "fullname" . }}
...
```

The `values.yaml` can contain Map data structures:

```yaml
name: inspector
image:
  repository: so0k/kuar-inspector
  tag: 1.0.0
```

Which can be used easily:

```yaml
spec:
  containers:
  - name: {{ .Values.name }}
    image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

The Go templating language even allows us to iterate over the key value pairs in the map:

```yaml
# values.yaml
config:
  kelsey-rating: "pretty dope"
  env: gcpug
  asset-pipeline: false

# manifest template fragment
env:
{{- range $key, $value :=  .Values.config }}
- name: {{ $key | upper | replace "-" "_" }}
  value: {{ $value | quote }}
{{- end }}

# rendered template fragment:
env:
- name: ASSET_PIPELINE
  value: "false"
- name: ENV
  value: "gcpug"
- name: KELSEY_RATING
  value: "pretty dope"
```
> *Note*: not only looping, but also conditionals and nesting is supported

The final result is a very powerfull and easy to install Chart:
```bash
helm install -n inspector-v1 code/inspector/
```

Use the instructions printed by the Helm chart to verify the environment was set correctly:

```bash
# port forward
kubectl get po
kubectl port-forward <pod> 8080:80
open http://localhost:8080
```

```bash
helm install -n inspector-v2 --set image.tag=2.0.0,config.kelsey-rating="extra dope" code/inspector/
```

### Debuging and Troubleshooting

As charts get more complicated, we often want to see the rendered manifests without
sending them to the Kubernetes API server:

```bash
helm install --dry-run --debug .
```

Helm Chart CI/CD systems may also want to ensure code quality is up to expectations using:

```bash
helm lint
```

### Kickstart your Chart

As demonstrated above, after writing a few Charts, there is a lot of boilerplate we will often re-use.

To facilitate Chart creation, starter scaffolding can be used (your own `--starter` can be provided).

The `helm create` command creates a chart directory along with the common files and
directories used in a chart.


```bash
# using the default scaffolding
helm create my-chart
```

### Other features

Just to highlight a few:

- Helm plugins: A plugin is a tool that can be accessed through the helm CLI, but which is
  not part of the built-in Helm codebase.

  See: [Samples](https://github.com/technosophos/helm-plugins)

- Central Repositories: [kubernetes/charts](https://github.com/kubernetes/charts)

   ```
   # Using alpine tag which does not password protect redis
   helm install kubernetes-charts/redis -n atlas-cache \
      --set persistence.enabled=false,image=redis:3.2-alpine
   ```

- Dependency Management: define `requirements.yaml`

  ```
  helm dep up
  ```

- Release Upgrades

  ```
  helm upgrade <release> .
  ```

## References

For more details on Chart writing:

- [Introduction to Charts](docs/01_INTRODUCTION.md)
- [Writing Charts](docs/02_WRITING_CHARTS.md)
- [Installing Charts](docs/04_INSTALLING_CHARTS.md)
