# Simple Helm chart for nginx

This is a very Simple helm chart which is trying to be as minimal as possible to only
demonstrate 2 Helm features.

This chart does not follow Helm Best Practices for the sake of simplicity.

## Usage

```
helm install .
```

## Directory Sructure:

```
code/simple-nginx/
├── Chart.yaml             Chart specs
├── README.md              This README
├── html                   HTML files for ConfigMap
│   └── index.html
├── templates              Kubernetes Manifests
│   ├── nginx-cm.yaml
│   ├── nginx-deploy.yaml
│   └── nginx-svc.yaml
└── values.yaml            A default Values file
```

## Purpose

Show templating features of helm:

1. Template Image Tags:
   ```
   spec:
      containers:
      - image: nginx:{{ .Values.tag }}
   ```

1. Template ConfigMaps:
   ```
   kind: ConfigMap
   metadata:
      name: nginx-index
   data:
   {{ (.Files.Glob "html/*").AsConfig | indent 2 }}
   ```
