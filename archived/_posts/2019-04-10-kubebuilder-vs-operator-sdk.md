---
layout: post
title: "kubebuilder vs operator-sdk"
tagline : ""
description: ""
category: "cloud"
tags: ["kubernetes", "kubebuilder", "operator"]
---
{% include JB/setup %}
# kubebuilder vs operator-sdk

## Abstract

[kubebuilder](https://github.com/kubernetes-sigs/kubebuilder) and [operator-sdk](https://github.com/operator-framework/operator-sdk) are two major tools used to set up
a controller/operator code base for kubernetes CRDs. Both of them under the hood using [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) to
adding a manager, api scheme and creates an structs implements a reconciler interface, thus the major difference are around the tools for integration, testing and etc.

## Backgroud

Kubernetes APIs provide consistent and well defined endpoints for objects adhering to a consistent and rich structure.
A resource is an endpoint in the Kubernetes API that stores a collection of API objects of a certain kind. For example, the built-in pods resource contains a collection of Pod objects.
A custom resource (CR) is an extension of the Kubernetes API that is not necessarily available in a default Kubernetes installation. It represents a customization of a particular Kubernetes installation. A CR is dynamically registered by Custom Resource Definition (CRD).

Building services as Kubernetes APIs provides many advantages to plain old REST, including:

* Hosted API endpoints, storage, and validation.
* Rich tooling and clis such as kubectl.
* Support for Authn and granular Authz.
* Support for API evolution through API versioning and conversion.
* Facilitation of adaptive / self-healing APIs that continuously respond to changes in the system state without user intervention.
* Kubernetes as a hosting environment

Developers may build and publish their own Kubernetes APIs for installation into running Kubernetes clusters.

In order to build and publish Kubernetes APIs in Go, there are two SDKs that provides abstractions and scaffolds to reduce developers' time spent on setting up project, so the developers could focus on features velocity. These two SDKs are kubebuilder and operator-sdk.

### kubebuilder

kubebuilder is one of the kubernetes-sigs (Special Interest Groups), part of the [API Machinery](https://github.com/kubernetes/community/blob/master/sig-api-machinery/README.md) group, both owners are from google, most commits from google, redhat.

kubebuilder v1.0.0 was released at July 19, 2018, the latest release is v1.0.8, the document could be found at https://book.kubebuilder.io/. On github there are 1129 stars, 205 forks, 61 contributors at the time this is written.

### operator-sdk

operator-sdk is part of operator-framework, part of CNCF landscape (https://landscape.cncf.io/selected=operator-framework), owners are from redhat, most commits from redhat.

operator-sdk first release v0.1.0 was at Oct 31, 2018, the latest release is v0.7.0, the document could be found at https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md. On github there are 1606 stars, 353 forks, 75 contributors at the time this is written.

### Summary

By comparing the backgrouds of two projects, they are pretty similar, we call it even on this round.

## Install

Installing kubebuilder and operator-sdk are pretty straight-forward, belows are steps for each of them.

### Install kubebuilder

To install kubebuilder, there is a script from kubebuilder repo that helps to install the tools quickly.

```
curl -L https://raw.githubusercontent.com/kubernetes-sigs/kubebuilder/master/scripts/install.sh | KUBEBUILDER_VERSION=${KUBEBUILDER_VERSION} bash -
```

Meanwhile, since kubebuilder using [kustomize](https://github.com/kubernetes-sigs/kustomize) for testing, we also need to install it, we will look into it later.

```
# on OSX
brew install kustomize
```

Since there is no go dependency on kubebuilder in the controller artefact, we don't need to use a fixed version of kubebuilder, in another word changing kubebuilder
version only impacts the scaffold codes.

### Install operator-sdk

To install operator-sdk, user can easily run

```
brew install operator-sdk
```

Of course in order to run testing with helm, tools like `helm` needs to be installed.

operator-sdk is imported as part of the artefact it builds (though only a tiny portion), a fixed version that matches vendored go package is preferred.


### Summary

By comparing the installing process, both of these two SDKs provide simple way to install. Though the kubebuilder decouples the tool setting up the scaffolds and the actual codes, wins a little.

## Development

Development workflow for both tools are well-documented, the following sections compare them in steps.

### Create new project

kubebuilder creates a project by run `kubebuilder init --domain k8s.io --license apache2 --owner "The Kubernetes Authors"`, after it completes, the project will look like below

```
.
├── Dockerfile
├── Gopkg.lock
├── Gopkg.toml
├── Makefile
├── PROJECT
├── README.md
├── bin
├── cmd
│   └── manager
├── config
│   ├── crds
│   ├── default
│   ├── manager
│   ├── rbac
│   └── samples
├── hack
├── pkg
│   ├── apis
│   ├── controller
│   └── webhook
└── vendor
```

operator-sdk creates a project by run `operator-sdk new app-operator`, after it completes, the project will look like below

```
.
├── Gopkg.lock
├── Gopkg.toml
├── build
│   ├── Dockerfile
│   └── bin
├── cmd
│   └── manager
├── deploy
│   ├── operator.yaml
│   ├── role.yaml
│   ├── role_binding.yaml
│   └── service_account.yaml
├── pkg
│   ├── apis
│   └── controller
├── vendor
└── version
```

### Create New API (Resource)

kubebuilder creates a new API by run `kubebuilder create api --group mygroup --version v1beta1 --kind MyKind`.
It will create a resource type under `apis/mygroup/v1beta1/` directory and a controller (by answer `y` when it asks if need to create controller) under `controler/mykind` directory.

operator-sdk creates a new API by run `operator-sdk add api --api-version=app.example.com/v1alpha1 --kind=AppService` and creates a controller
for it by run `operator-sdk add controller --api-version=app.example.com/v1alpha1 --kind=AppService`.
The first step will create an API under `apis/app/v1alpha1/` and the second will create a controller under `controller/appservice/`


Difference on created resources:

* Controller created by kubebuilder by default is cluster-scoped, while created by operator-sdk is namespace-scoped
* Controller code scaffolds examples are different, kubebuilder using example interacting with a deployment, where  operator-sdk uses pod as example
* The vendored packages are different, when kubebuilder only imports `sigs.k8s.io` and `k8s.io` and testing framework, the operator-sdk requires `operator-sdk` in its imported packages

Other than above, there is really no much difference between the codes generated by the SDKs. All the codes generated for API are an implemetation of `runtime.Object` interface and all the codes
for controller are an implemetation of `reconcile.Reconciler`.

### Generate manifests

In the project set up by kubebuilder, user could run `make generate` and `make manifests` to generate the codes and manifests. It uses `go generate` and `go build` hooks to generate
the codes and manifests (includes CRDs). Whenever user makes a change on the API spec, add a validation for a field, managing subresources (e.g. status, scale), printcolumn, they can
just update the codes and re-run `make manifests`, the CRDs and role-binding will be generated with proper configuration.

e.g.

- A comment on a spec field, will end up in the CRD

```
# in code
type IPAddressSpec struct {
    // IPAllocatorName is the IPAllocator's name used to allocate this ip.
    // +kubebuilder:validation:MinLength=1
    IPAllocatorName string `json:"ipallocatorName"`
}

# in CRD yaml
ipallocatorName:
  description: IPAllocatorName is the IPAllocator's name used to allocate
    this ip.
  minLength: 1
  type: string
```

- A comment on a resource, will generate the subresource and printcolumn
```
# in code
// IPAllocator is the Schema for the ipallocators API
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +kubebuilder:printcolumn:name="source",type="string",JSONPath=".spec.source"
// +kubebuilder:printcolumn:name="state",type="string",JSONPath=".status.state"
// +kubebuilder:printcolumn:name="free",type="string",JSONPath=".status.capacity.free"
// +kubebuilder:subresource:status
// +k8s:openapi-gen=true
type IPAllocator struct { ... }

# in CRD yaml
  additionalPrinterColumns:
  - JSONPath: .spec.source
    name: source
    type: string
  - JSONPath: .status.state
    name: state
    type: string
  - JSONPath: .status.capacity.free
    name: free
    type: string
...
  subresources:
    status: {}
```

- A comment on a reconciler will genreate the RBAC

```
// +kubebuilder:rbac:groups=net.ccp.cisco.com,resources=ipallocators,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=net.ccp.cisco.com,resources=ipallocators/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=net.ccp.cisco.com,resources=ipaddresses,verbs=get;list
// +kubebuilder:rbac:groups=net.ccp.cisco.com,resources=ipaddresses/status,verbs=get
func (r *ReconcileIPAllocator) Reconcile(request reconcile.Request) (reconcile.Result, error) { ... }
```

In the project setup by operator-sdk, user could run `operator-sdk generate k8s` to generate codes, but it doesn't provides
updating of CRD and roles through codes.


### Testing

kubebuilder will create testing scaffolds for all API and controller it creates, named as `_test.go`, it uses `gomega` as testing
framework, and uses `controller-runtime/pkg/envtest` as to bring up a testing environment. Since kubebuilder is part of kubernetes-sigs,
it has better set up to work with the existing kubernetes-sigs projects. During kubebuilder installation, it will put the kubebuilder
binary with a `etcd`, `kube-apiserver` and `kubectl` under `/usr/local/kubebuilder/bin/`, which is the default binary path for the `envtest`.
During testing of controller, it will bring up an actual `kube-apiserver` and `etcd` (done by envtest), and doing the testing. To run
tests, there is a make target for it, user could run `make test` to start testing.

On the other hand, operator-sdk doesn't generates testing scaffolds, instead it only provdes a [document](https://github.com/operator-framework/operator-sdk/blob/master/doc/test-framework/writing-e2e-tests.md).
User follows the document could run `operator-sdk test local` to test the controller.

### Deploy & Run

To deploy CRDs, kubebuilder provides a make target to deploy it easiy, while operator-sdk is a manual process (write your own makefile).

```
# kubebuilder
make install

# operator-sdk
kubectl create -f deploy/crds/
```

Both kubebuilder and operator-sdk provide two options to run the artefact, outside and inside a k8s cluster.

With kubebuilder, to run the controller outside k8s cluster as binary, run `make run`, to run the controller within a cluster, need to run `make docker-build deploy` which builds
out the docker container, and generates deploy YAML files by `kustomize`, then get applied on a k8s cluster.

With operator-sdk, to run the controller outside k8s cluster as bianry, run `operator-sdk up local`. to run the
controller within a cluster as a pod, after run `operator-sdk build` to build the docker image, then deploy
it either using its helm watcher, or manually with YAML files. Meanwhile, operator-sdk provides a watcher helpers
for helm and ansible, that automatically reload containers/configs when there is change made.

### Misc

kubebuilder provides a webhook service, metric service and upstream is working on document generating for a project.

operator-sdk provides metric service and integration with helm and ansible.

### Summary

Looking through the developing steps for both tools, the kubebuilder is the winner, the best part of it is it generates CRDs and roles
from codes, so we don't have to manually manages it by editing YAML files. The operator-sdk also has some extra features like helm watcher that
helps developing.

## Conclusion

Both kubebuilder and operator-sdk are good tools to start writing k8s controllers, they are well documented, having good communities and provide
helper commands to start a new project or adding new controller. The operator-sdk has better integration with helm, and having better integration
with the operator-framework ecosystem, while the kuebuilder integrates with existing kubernetes-sigs projects naturally. This post goes through background
, installing and development process for both tools, and I think the kuebuilder could provides better management over codes, so I would suggest to use
kuebuilder to start a new projects.

---
[Wei T.](wtie@cisco.com)
2019-04-10
