## Installing Charts

Ideally, Helm is used in combination with a Repository server to support the ability to remotely search charts and install them into the Kubernetes cluster of choice.

Currently, Honestbee is not using a repository server. Therefore, to install these Charts please clone this repository locally.

Going forward, a repository server may be created either by:

- using GitHub pages, or
- using Object storage (i.e. s3, gcloud)

### GitHub pages

1. [Create gh-pages branch](https://github.com/kubernetes/helm/blob/master/docs/chart_repository.md#github-pages-example)
1. [Use GitHub Helm plugin](https://github.com/technosophos/helm-plugins/blob/master/github/github.sh)

### Object storage

1. [Google Cloud Storage Example](https://github.com/kubernetes/helm/blob/master/docs/chart_repository.md#google-cloud-storage)
1. [Sync script](https://github.com/kubernetes/helm/blob/master/scripts/sync-repo.sh)