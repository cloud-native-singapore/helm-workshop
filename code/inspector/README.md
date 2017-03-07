# Helm chart for Kelsey Hightower's Inspector

This helm chart tries to show off the powerfull configuration features `values.yaml` provides to Helm Charts.

This chart does not follow Helm Best Practices for the sake of simplicity and uses Kelsey Hightower's [Inspector](https://github.com/kelseyhightower/inspector) service to demonstrate its functionality.

## Usage

```
# install
helm install -n inspector-v1 .

# port forward
kubectl get po
kubectl port-forward <pod> 8080:80
open http://localhost:8080
```

## Directory Sructure:

```
code/inspector/
├── Chart.yaml
├── README.md
├── templates
│   ├── _helpers.tpl
│   ├── deploy.yaml
│   └── svc.yaml
└── values.yaml
```

## Purpose

Show templating features of helm:

1. Nested Configuration values:
   ```yaml
   name: inspector
   image:
     repository: so0k/kuar-inspector
     tag: 1.0.0
   config:
     kelsey-rating: "pretty dope"
     env: gcpug
     asset-pipeline: false
   ```
   Note: [the Hightower scale](https://twitter.com/kelseyhightower/status/801102768232480769?lang=en)

1. Template helpers:
   ```yaml
   {{- define "fullname" -}}
   {{- printf "%s-%s" .Release.Name .Values.name | trunc 63 -}}
   {{- end -}}
   ```

1. Iterating over nested config maps:
    ```yaml
    # manifest template fragment
    env:
    {{- range $key, $value :=  .Values.config }}
    * name: {{ $key | upper | replace "-" "_" }}
      value: {{ $value | quote }}
    {{- end }}

    # rendered template fragment:
    env:
    * name: ASSET_PIPELINE
      value: "false"
    * name: ENV
      value: "gcpug"
    * name: KELSEY_RATING
      value: "pretty dope"
    ```
