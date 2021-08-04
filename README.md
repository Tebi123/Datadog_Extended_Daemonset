# ExtendedDaemonSet

![badge](https://action-badges.now.sh/datadog/extendeddaemonset)
[![Go Report Card](https://goreportcard.com/badge/github.com/DataDog/extendeddaemonset)](https://goreportcard.com/report/github.com/DataDog/extendeddaemonset)
[![codecov](https://codecov.io/gh/datadog/extendeddaemonset/branch/main/graph/badge.svg)](https://codecov.io/gh/datadog/extendeddaemonset)

**ExtendedDaemonSet** aims to provide a new implementation of the Kubernetes `DaemonSet` resource with key features:

* Canary Deployment: Deploy a new DaemonSet version with only a few nodes.
* Custom Rolling Update: Improve the default rolling update logic available in Kubernetes `batch/v1 Daemonset`.

## How to use it

### Deployment

To use the ExtendedDaemonSet controller in your Kubernetes cluster, only two commands are required:

First, deploy the Custom Resources Definitions:

```console
$ make install
```

Then deploy default manifest (uses Kustomize)

```console
$ make deploy
```

By default, the controller only watches the ExtendedDaemonSet resources that are present in its own namespace. If you want to deploy the controller cluster wide, add a Kustomization to the `config/manager`

```yaml
            env:
            - name: WATCH_NAMESPACE
              value: ""
```

Alternatively, you can use this [helm chart](
https://github.com/DataDog/helm-charts/tree/master/charts/extended-daemon-set) to deploy:

```bash
helm repo add datadog https://helm.datadoghq.com
helm repo update
helm install eds datadog/extendeddaemonset
```

### Demo application

If you want to test and compare the advantages of the ExtendedDaemonSet over the the standard DaemonSet, you can use the demo application available in the `/example` folder. Follow the below scenario:

First, you need a Kubernetes cluster with several nodes; we recommend using three nodes. If you want, you can use [kind.sigs.k8s.io](https://kind.sigs.k8s.io/) to create a three node cluster with the following command: `kind create cluster --config examples/kind-cluster-configuration.yaml`.

This creates a three node cluster with one control-plane and two worker nodes:

```console
$ kind create cluster --config examples/kind-cluster-configuration.yaml
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.15.3) 🖼
 ✓ Preparing nodes 📦📦📦
 ✓ Creating kubeadm config 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Cluster creation complete. You can now use the cluster with:

$ export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"

```

#### ExtendedDaemonSet controller deployment

```console
# deploy the controller needed crds
$ make install

# deploy the controller pod
$ make deploy

# you should see the extendeddaemonset controller pod running
$ kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
extendeddaemonset-855cd7c679-gpmql   1/1     Running   0          2m11s

```

#### `foo` ExtendedDaemonSet deployment

Create the `foo` app with the ExtendedDaemonSet. For demo purposes, we'll use the `k8s.gcr.io/pause` Docker image, which is only awaiting a terminating signal. You can look at the `foo` application definition in the file `examples/foo-eds_v1.yaml`.

```console
$ kubectl apply -f examples/foo-eds_v1.yaml
extendeddaemonset.datadoghq.com/foo created
```

You can see the state of the ExtendedDaemonSet `foo` with:

```console
$ kubectl get eds
NAME   DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   STATUS    ACTIVE RS   CANARY RS   AGE
foo    3         3         3       3            3           Running   foo-8z7lr               44s

# Also the `extendeddaemonsetreplicaset` resource generated by the controller from the `foo` EDS instance:
$ kubectl get ers
NAME        STATUS   DESIRED   CURRENT   READY   AVAILABLE   NODE SELECTOR   AGE
foo-8z7lr   active   3         3         3       3                           61s

```

#### `foo` ExtendedDaemonSet deployment update with canary strategy

Now we can try to update the ExtendedDaemonSet `foo`. The only difference between the two versions is the Docker image used in the pod template.

```console
$ diff examples/foo-eds_v1.yaml examples/foo-eds_v2.yaml
17c17
<         image: k8s.gcr.io/pause:3.0
---
>         image: k8s.gcr.io/pause:3.1

$ kubectl apply -f examples/foo-eds_v2.yaml
extendeddaemonset.datadoghq.com/foo configured
```

As you can see with the following command, a canary ReplicaSet is now configured for the `foo` ExtendedDaemonSet. Additionally, a new ExtendedReplicaSet has been created to handle the new `foo` pod template version.

```console
$ kubectl get eds
NAME   DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   STATUS   ACTIVE RS   CANARY RS   AGE
foo    3         3         3       3            3           Canary   foo-8z7lr   foo-xdj4b   85s

$ kubectl get ers
NAME        STATUS   DESIRED   CURRENT   READY   AVAILABLE   NODE SELECTOR   AGE
foo-8z7lr   active   2         2         2       2                           2m
foo-xdj4b   canary   1         1         1       1                           40s

$ kubectl get pod -l extendeddaemonset.datadoghq.com/name=foo
NAME                                 READY   STATUS    RESTARTS   AGE
foo-8z7lr-bp9w8                      1/1     Running   0          108s
foo-8z7lr-jlvrq                      1/1     Running   0          88s
foo-xdj4b-zvss2                      1/1     Running   0          8s
```

Only one pod is running with the ExtendedReplicaSet `foo-xdj4b` pod template version. This corresponds to the setting `spec.canary.replicas` in the ExtendedDaemonSet `foo`.

#### Rolling update after the canary deployment validation period ended

After 5 minutes, which corresponds to `spec.canary.duration`, the controller will set as valid and activate the `foo-xdj4b` ExtendedReplicaSet. It will trigger the full `foo-xdj4b` ExtendedReplicaSet deployment.

```console
$ kubectl get eds
NAME   DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   STATUS    ACTIVE RS   CANARY RS   AGE
foo    3         3         3       3            3           Running   foo-xdj4b               9m21s

$ kubectl get ers
NAME        STATUS   DESIRED   CURRENT   READY   AVAILABLE   NODE SELECTOR   AGE
foo-xdj4b   active   3         3         3       3                           8m21s

$ kubectl get pod -l extendeddaemonset.datadoghq.com/name=foo
NAME              READY   STATUS    RESTARTS   AGE
foo-xdj4b-hh6d8   1/1     Running   0          5m11s
foo-xdj4b-rgtk9   1/1     Running   0          5m31s
foo-xdj4b-zvss2   1/1     Running   0          10m
```

#### Overwrite container's Pod resources for a specific Node

The ExtendedDaemonset controller allows to overwrite the container's pod managed by an ExtendedDaemonset for a specific Node, thanks to an annotation that you can set on the Node: `resources.extendeddaemonset.datadoghq.com/<eds-namespace>.<eds-name>.<container-name>={...}`. the value corresponds to the Resources definition in JSON.

For example, for the ExtendedDaemonset named `foo` in the `bar` namespace. The container `myapp` resources specification can be overwriten by adding the following annotation on a Node:

```console
$ kubectl annotate node <node-name> `resources.extendeddaemonset.datadoghq.com/bar.foo.myapp={"requests":{"cpu":"2.0","memory":"2G"}}`
node/<node-name> annotated
```

#### Overwrite container's Pod resources for a set of Nodes with `ExtendedDaemonsetSettings`

In some cases (for example with different nodes type), it can be useful to have different resource configurations for a Daemonset to handle the Node's workload specificity.

To do so you can create an instance of `ExtendedDaemonsetSetting` resource that aims to overwrite the resources
definition of the container(s) present in ExtendedDaemonset Pods.

the information needed is:

* `spec.nodeSelector`: a NodeLabels selector that matches with the nodes where it must trigger the usage of this resource.
* `spec.reference`: contains enough information to let you identify the referred resource.
* `spec.containers`: contains a list of Container spec overwrites.

```yaml
apiVersion: datadoghq.com/v1alpha1
kind: ExtendedDaemonsetSetting
metadata:
  name: foo-xxl-node
spec:
  nodeSelector:
    matchLabels:
      node-type: xxl
  reference:
    kind: ExtendedDaemonset
    name: foo
  containers:
   - name: daemon
    resources:
      requests:
        cpu: "0.5"
        memory: "300m"
```

#### Remove a pod on a given node using `nodeAffinity`

In some cases, it could be useful to remove a daemon pod on a given node. This can be done using the `podTemplate.spec.affinity.nodeAffinity` field.

First set a new `requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms` field

```yaml
apiVersion: datadoghq.com/v1alpha1
kind: ExtendedDaemonSet
metadata:
  name: foo
spec:
  template:
    spec:
      //...
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: extendeddaemonset.datadoghq.com/exclude
                operator: NotIn
                values:
                - foo
```

Then add the label `extendeddaemonset.datadoghq.com/exclude=foo` to the node in question

`kubectl label nodes <your-node-name> extendeddaemonset.datadoghq.com/exclude=foo`


#### Canary settings

The Canary deployment can be customized in a few ways.

- `replicas`: The number of replica pods to participate in the Canary deployment
- `duration`: The duration of the Canary deployment, after which the Canary deployment will end and the active ExtendedReplicaSet will update
- `autoPause.enabled`: Activation of the Canary deployment auto pausing feature (default is `true`)
- `autoPause.maxRestarts`: The maximum number of restarts tolerable before the Canary deployment is automatically paused (default is `2`)
- `validationMode`: Used to configure how a canary deployment is validated. Possible values are `auto` (default) and `manual`. 
  In manual mode canary will be validated only after `kubectl-eds canary validate` command. You can control default value by setting `EDS_VALIDATION_MODE` environment variable for deployment.
  When set to `manual` `duration` and `noRestartsDuration` will have no effect and will not be defaulted. Setting them to some value will result in validation error.

Example configuration of the spec canary strategy:

```
spec:
  strategy:
    canary:
      replicas: 1
      duration: 5m
      autoPause:
        enabled: true
        maxRestarts: 5
```


### Kubectl plugin

To build the the kubectl ExtendedDaemonSet plugin, you can run the command: `make build-plugin`. This will create the `kubectl-eds` Go binary, corresponding to your local OS and architecture.
Then, add or move this binary to the `PATH` and run the command `kubectl eds`:

```console
$ kubectl eds
Usage:
  ExtendedDaemonset [command]

Available Commands:
  canary      control ExtendedDaemonset canary deployment
  get         get ExtendedDaemonSet deployment(s)
  get-ers     get-ers ExtendedDaemonSetReplicaset deployment(s)
  help        Help about any command
  pods        print the list pods managed by the EDS
```

#### List the not ready pods managed by the ExtendedDaemonSet

`kubectl-eds pods <ExtendedDaemonSet name> --select=not-ready`

#### List the active Canary pods

Print the canary pods and their corresponding status and restart counts.

`kubectl-eds canary pods <ExtendedDaemonSet name>`

OR

`kubectl-eds pods <ExtendedDaemonSet name> --select=canary`

#### Validate Canary deployment

As an alternative to waiting for the Canary duration to end, the deployment can be manually validated.

`kubectl-eds canary validate <ExtendedDaemonSet name>`

#### Pause Canary deployment

The Canary deployment can be paused to investigate an issue.

`kubectl-eds canary pause <ExtendedDaemonSet name>`

#### Unpause Canary deployment

The Canary deployment can be unpaused, and the Canary duration will continue.

`kubectl-eds canary unpause <ExtendedDaemonSet name>`

#### Fail Canary deployment

The Canary deployment can be manually failed. This command will restore the currently active ExtendedReplicaSet on the Canary pods.

`kubectl-eds canary fail <ExtendedDaemonSet name>`

### How to migrate from a DaemonSet

If you already have an application running in your cluster with a DaemonSet, it is possible to migrate to an ExtendedDaemonSet with a `smooth` migration path.

* Update your `DaemonSet` specification to set a toleration that does not correspond to your node's taints. As a result, the `DaemonSet` pods that are already running will not be deleted, and the `DaemonSet` controller will not take any new actions on it.

* In the ExtendedDaemonSet definition, add a specific annotation to inform the `extendeddaemonset-controller` which `DaemonSet` needs to be migrated. The controller will recognize the pods from the "old" DaemonSet as a previous version and will do a proper rolling update.

```yaml
apiVersion: datadoghq.com/v1alpha1
kind: ExtendedDaemonSet
metadata:
  name: foo
  annotations:
    extendeddaemonset.datadoghq.com/old-daemonset: foo
spec:
    # ...
```

## Developers section

### How to build it

This project uses ```go module```. Ensure you have it activated: ```export GO111MODULE=on```.

Run ```make install-tools``` to install mandatory tooling, like the `kubebuilder` or the `golangci` linter.

```console
$ make build
CGO_ENABLED=0 go build -i -installsuffix cgo -ldflags '-w' -o controller ./cmd/manager/main.go
```

### Implementation documentation

* [Reconcile loops interactions](./docs/canary-worflows.md)

### How to test it

### Custom image

You can create (and deploy) a custom image easily through the `IMG` environment variable:

```
IMG=<your_dockerhub_repo>/extendeddaemonset:test make docker-build docker-push deploy
```

#### Unit tests

```console
$ make test
ok      github.com/DataDog/extendeddaemonset/controllers/extendeddaemonset   1.107s  coverage: 77.0% of statements
ok      github.com/DataDog/extendeddaemonset/controllers/extendeddaemonsetreplicaset 1.098s  coverage: 63.9% of statements
ok      github.com/DataDog/extendeddaemonset/controllers/extendeddaemonsetreplicaset/strategy        1.036s  coverage: 5.3% of statements
ok      github.com/DataDog/extendeddaemonset/controllers/extendeddaemonsetreplicaset/strategy/limits 1.016s  coverage: 83.3% of statements
ok      github.com/DataDog/extendeddaemonset/pkg/controller/utils       1.015s  coverage: 100.0% of statements
```

##### controller-runtime envtest

This project is using the `controller-runtime` [envtest](https://book.kubebuilder.io/reference/envtest.html) to test reconcile controllers loop against an API-Server dynamically started to run the tests.
One advantage of using the controller-runtime `envtest` is that tests run faster compared to tests that run against a real k8s cluster. The downside is: only the `API-Server` is running but not the other controllers, so resources such as Pods are not updated by a Kubelet. You can find more information about the `envtest` limitation [here](https://book.kubebuilder.io/reference/envtest.html#testing-considerations).

these test are located in `/controllers/extendeddaemonset_test.go`

#### end2end test

End2end tests are also present. Unlike the tests that use the `envtest`, the e2e tests need to have access to a running kubernetes cluster.
The envvar KUBECONFIG should be set in the terminal where the `make e2e` is executed.

[Kind](https://kind.sh/) is a great solution to start a multi-nodes cluster locally.

```console
$ kind create cluster --config examples/kind-cluster-configuration.yaml
cluster created
$ make e2e
Ran 12 of 12 Specs in 242.249 seconds
SUCCESS! -- 12 Passed | 0 Failed | 0 Pending | 0 Skipped
--- PASS: TestAPIs (242.25s)
PASS
ok      github.com/DataDog/extendeddaemonset/controllers        242.686s
```

### Linter validation

To use the linter, run:

```console
$ make lint
./bin/golangci-lint run ./...
```

Note that it runs automatically when running the `test` or `build` targets.

### How to release it

See [RELEASING](RELEASING.md)
