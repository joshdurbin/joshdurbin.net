+++
title = "Helm, Kubernetes 'package management', in a nutshell..."
date = "2018-04-09"
tags = ["helm", "kubernetes", "containers", "docker"]
aliases = [ "/blog/helm-kubernetes-package-management-in-a-nutshell.../", "/blurbs/helm-kubernetes-package-management-in-a-nutshell.../"]
+++

### What is Helm?

Helm is a Kubernetes package manager managed by CNCF, and started by Google and [Deis](https://deis.com).
It packages multiple Kubernetes resources into a single logical deployment -- a chart.

### Core Objectives

- Installing resources in Kubernetes should be easy like apt/yum/homebrew
- Teams should be able to easily collaborate
- Releases should be reproducible
- Packages should be sharable

### Building Blocks

Helm consists of the following four resources:

1. A chart is a package, a bundle of Kubernetes resources
2. A release is an instance of a chart that has been loaded into a Kubernetes cluster (tiller)
3. A repository is a collection of public charts
4. A template is a kubernetes resource / configuration file consisting of Go/Sprig templating

### Client and Server

A user (you, jenkins, etc...) uses the Helm CLI to communicate with Tiller, the Helm server.
Tiller runs in its own Kubernetes Pod and functions as the privileged entity that manages
deployments within your Kubernetes cluster. Tiller communicates with the Kubernetes API
to orchestrate the creation, management, and tracking of Kubernetes resources. Tiller uses
a service account to interact with the Kubernetes API which is very likely different than
the account used by the end user, the user driving the Helm CLI. This is important distinction
one should be aware of, particularly in cases where roles or permissioning ([RBAC](https://kubernetes.io/docs/admin/authorization/rbac/))
is suspected as the root cause of errors.

### Project Structure

```
redis-sample
├── Chart.yaml          # contains basic information about chart
├── charts              # contains fetched dependencies -- via ‘helm dependency update’
│   └── dep-x-0.1.0.tgz # version 0.1.0 of dependency ‘x’
├── requirements.lock   # contains hash, signatures, and metadata for fetched dependencies
├── requirements.yaml   # contains dependency definitions
├── templates           # templates that are combined with values to create K8S manifest files
│   ├── _helpers.tpl
│   ├── deployment.yaml # deployment manifest
│   ├── pvc.yaml        # pvc manifest
│   └── service.yaml    # svc manifest
└── values.yaml         # default configuration values for this chart
```

### Project Structure: chart.yaml

The `chart.yaml` describes (limited set):

- The official chart name, its version
- Author, sources, home and other data for search indexes

Ex:

```
name: kubeotron
version: 0.6.0
appVersion: 0.6.1
description: Kubeotron lords over all other pods
home: https://github.com/kubeo/trontron
sources:
- https://github.com/kubeo/trontron
maintainers:
- name: Kube Frankenstein
  email: ooopsdid@creatamonster.io
engine: deadness
```

### Project Structure: Dependencies

Dependencies can exist as:

- Embedded dependencies, charts, in the requirements directory
- In requirements.yaml file in which developers can declare chart dependencies,
along with the location of the chart and the desired version

Ex:

```
dependencies:
  - name: redis
    version: 0.10.1
    repository: https://kubernetes-charts.storage.googleapis.com/
```

### Project Structure: Templates

- The Go Template language {{.foo | quote }}
- Variables, simple control structures (eg: looping, conditionals, nesting)
- Pipelines allow for output and input of functions to be chained
- 50+ functions from Go/Sprig Template libraries:
  - date, string, conversions, encoding, reflection, data structures, math, crypto, semver

### Project Structure: Values

- Parameters that can be injected into templates
- Simple YAML with namespaces
- Each subchart can have its own values.yaml
- A chart can leverage multiple values.yaml files
- Can override individual value for an installation / update operation

Ex yaml:

```
anotherYamlFileQuestionMark: true
favoritePlaces:
  - name: Garbage Patch
    value: false
  - name: Pumpkin Patch
    value: true
desiredPumpkins: 2020
```

### Installing a Chart

- Tiller installation -- `helm init` (leverages the default namespace in your
kubernetes config) to install tiller in a pod and service (tiller-deploy) in your namespace
- Configure repositories -- helm comes with a stable and local repo configured;
`helm repo list`. Any additional repos must be added; `helm repo add <web-endpoint>`.
- (optional) Search for charts across the configured repos; `helm search <term>`.
- Install a chart and its dependencies to create a release; `helm install <chart-name>`.
Use `--dry-run` and `--debug` to see what the produced combined output before it’s
communicated to Tiller for instruction application against the Kubernetes APIs.

![image](/img/2018-04-kubernetes-helm-nutshell/helm_workflow.jpg)

### Release and Chart operations

- List releases tracked by the Helm server (tiller); `helm list`.
- Upgrade or rollback a release; `helm upgrade|rollback <release-name> <chart-name>`.
- Remove a release (`--purge` removes all state tracking from tiller); `helm delete <release-name>`.

### Build a Chart

- Pull any dependencies for your Chart; `helm dependency update`. Remember dependencies
are other Charts that are subject to the same release cycles as your Chart.
- Package your chart; `helm package`.

### Plugins

Plugins are client-side tools written in any language that integrate seamlessly
with the Helm CLI. Plugins do not impact the core Helm tool or the Tiller server.

The Helm plugin model is partially modeled on Git's plugin model. This allows
Helm to provide the target user experience and top level processing logic, while
the plugins do the work of performing a desired action.

### Hooks

- Allow for the execution of operations at specific points in a release cycle
- Operations can be any Kubernetes resource: job, config-map, secret, pod, etc...
- The resources that are created as part of a hook are not tracked as part of
the Helm Chart release

### Tips

1. Learn and practice [Go Template](https://godoc.org/text/template) language
(and [Sprig template library](https://godoc.org/github.com/Masterminds/sprig))
2. Use helm test to validate releases
3. Manage environments with multiple Values ﬁles
4. Use helm template plugin to debug Helm Charts; or use --dry-run ﬂag

### Supplemental Resources

- [Stable Charts on GH](https://github.com/kubernetes/charts)
- [Chart Template Guide(s)](https://github.com/kubernetes/helm/tree/master/docs/chart_template_guide)
- [Kubernetes Helm Plugin Guide](https://github.com/kubernetes/helm/blob/master/docs/plugins.md)
