# Vertical Pod Autoscaler

# Intro

Vertical Pod Autoscaler (VPA) frees the users from necessity of setting
up-to-date resource requests for the containers in their pods.
When configured, it will set the requests automatically based on usage and
thus allow proper scheduling onto nodes so that appropriate resource amount is
available for each pod.

Autoscaling is configured with a
[Custom Resource Definition object](https://kubernetes.io/docs/concepts/api-extension/custom-resources/)
called [VerticalPodAutoscaler](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/apis/poc.autoscaling.k8s.io/v1alpha1/types.go).
It allows to specify which pods should be under vertically autoscaled as well as if/how the
resource recommendations are applied.

To enable vertical pod autoscaling on your cluster please follow the installation
procedure described below.


# Installation

### Notice on the backwards compatibility

During alpha the VPA CRD object may evolve in a way that is not compatible between releases.
If you install a new release of VPA it is safest to delete the existing VPA CRD objects.
Note that this will happen automatically if you just use the `vpa-down.sh` script to tear down
the old installation of VPA.

### Prerequisites

* It is strongly recommended to use Kubernetes 1.9 or greater.
  Your cluster must support MutatingAdmissionWebhooks, which are enabled by default
  since 1.9 ([#58255](https://github.com/kubernetes/kubernetes/pull/58255)).
  Read more about [VPA Admission Webhook](./admission-controller/README.md#running).
* `kubectl` should be connected to the cluster you want to install VPA in.
* The metrics server must be deployed in your cluster. Read more about [Metrics Server](https://github.com/kubernetes-incubator/metrics-server).
* If you are using a GKE Kubernetes cluster, you will need to grant your current Google
  identity `cluster-admin` role. Otherwise you won't be authorized to grant extra
  privileges to the VPA system components.
  ```console
  $ gcloud info | grep Account    # get current google identity
  Account: [myname@example.org]

  $ kubectl create clusterrolebinding myname-cluster-admin-binding --clusterrole=cluster-admin --user=myname@example.org
  Clusterrolebinding "myname-cluster-admin-binding" created
  ```
* If you already have another version of VPA installed in your cluster, you have to tear down
  the existing installation first with:
  ```
  ./hack/vpa-down.sh
  ```

### Install command

To install VPA, please download the source code of VPA (for example with `git clone https://github.com/kubernetes/autoscaler.git`)
and run the following command inside the `vertical-pod-autoscaler` directory:

```
./hack/vpa-up.sh
```

Note: the script currently reads environment variables: `$REGISTRY` and `$TAG`.
Make sure you leave them unset unless you want to use a non-default version of VPA.

The script issues multiple `kubectl` commands to the
cluster that insert the configuration and start all needed pods (see
[architecture](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/autoscaling/vertical-pod-autoscaler.md#architecture-overview))
in the `kube-system` namespace. It also generates
and uploads a secret (a CA cert) used by VPA Admission Controller when communicating
with the API server.

### Quick start

After [installation](#installation) the system is ready to recommend and set
resource requests for your pods.
In order to use it you need to insert a *Vertical Pod Autoscaler* resource for
each logical group of pods that have similar resource requirements.
We recommend to insert a *VPA* per each *Deployment* you want to control
automatically and use the same label selector as the *Deployment* uses.
There are three modes in which *VPAs* operate:

* `"Auto"`: VPA assigns resource requests on pod creation as well as updates
  them on existing pods using the preferred update mechanism. Currently this is
  equivalent to `"Recreate"` (see below). Once restart free ("in-place") update
  of pod requests is available, it may be used as the preferred update mechanism by
  the `"Auto"` mode.
* `"Recreate"`: VPA assigns resource requests on pod creation as well as updates
  them on existing pods by evicting them when the requested resources differ significantly
  from the new recommendation (respecting the Pod Distruption Budget, if defined).
  This mode should be used rarely, only if you need to ensure that the pods are restarted
  whenever the resource request changes. Otherwise prefer the `"Auto"` mode which may take
  advantage of restart free updates once they are available.
* `"Initial"`: VPA only assigns resource requests on pod creation and never changes them
  later.
* `"Off"`: VPA does not automatically change resource requirements of the pods.
  The recommendations are calculated and can be inspected in the VPA object.

### Test your installation

A simple way to check if Vertical Pod Autoscaler is fully operational in your
cluster is to create a sample deployment and a corresponding VPA config:
```
kubectl create -f examples/hamster.yaml
```

The above command creates a deployment with 2 pods, each running a single container
that requests 100 millicores and tries to utilize slightly above 500 millicores.
The command also creates a VPA config with selector that matches the pods in the deployment.
VPA will observe the behavior of the pods and after about 5 minutes they should get
updated with a higher CPU request
(note that VPA does not modify the template in the deployment, but the actual requests
of the pods are updated). To see VPA config and current recommended resource requests run:
```
kubectl describe vpa
```


*Note: if your cluster has little free capacity these pods may be unable to schedule.
You may need to add more nodes or adjust examples/hamster.yaml to use less CPU.*

### Example VPA configuration

```
apiVersion: poc.autoscaling.k8s.io/v1alpha1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  selector:
    matchLabels:
      app: my-app
  updatePolicy:
    updateMode: "Auto"
```

### Troubleshooting

To diagnose problems with a VPA installation, perform the following steps:

* Check if all system components are running:
```
kubectl --namespace=kube-system get pods|grep vpa
```
The above command should list 3 pods (recommender, updater and admission-controller)
all in state Running.

* Check if the system components log any errors.
For each of the pods returned by the previous command do:
```
kubectl --namespace=kube-system logs [pod name]| grep -e '^E[0-9]\{4\}'
```

* Check that the VPA Custom Resource Definition was created:
```
kubectl get customresourcedefinition|grep verticalpodautoscalers
```

### Components of VPA

The project consists of 3 components:

* Recommender - it monitors the current and past resource consumption and, based on it,
provides recommended values containers' cpu and memory requests.

* Updater - it checks which of the managed pods have correct resources set and, if not,
kills them so that they can be recreated by their controllers with the updated requests.

* Admission Plugin - it sets the correct resource requests on new pods (either just created
or recreated by their controller due to Updater's activity).

More on the architecture can be found [HERE](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/autoscaling/vertical-pod-autoscaler.md).

### Tear down

Note that if you stop running VPA in your cluster, the resource requests
for the pods already modified by VPA will not change, but any new pods
will get resources as defined in your controllers (i.e. deployment or
replicaset) and not according to previous recommendations made by VPA.

To stop using Vertical Pod Autoscaling in your cluster:
* If runnning on GKE, clean up role bindings created in [Prerequisites](#prerequisites):
```
kubectl delete clusterrolebinding myname-cluster-admin-binding
```
* Tear down VPA components:
```
./hack/vpa-down.sh
```

# Known limitations of the alpha version

* Whenever VPA updates the pod resources the pod is recreated, which causes all
  running containers to be restarted. The pod may be recreated on a different node.
* Vertical Pod Autoscaler **should not be used with the [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) (HPA) on CPU or memory** at this moment. 
  However, you can use VPA with [HPA on custom and external metrics](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-custom-metrics).
* VPA in `Auto` mode can only be used on pods that run under a controller
  (such as Deployment), which is responsible for restarting deleted pods.
  **Using VPA in `Auto` mode with a pod not running under any controller will
  cause the pod to be deleted and not recreated**.
* The VPA admission controller is an admission webhook. If you add other admission webhooks
  to you cluster, it is important to analyze how they interact and whether they may conflict
  with each other. The order of admission controllers is defined by a flag on APIserver.
* VPA reacts to some out-of-memory events, but not in all situations.
* VPA performance has not been tested in large clusters.
* VPA recommendation might exceed available resources (e.g. Node size, available
  size, available quota) and cause **pods to go pending**. This can be addressed by
  using VPA together with [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#basics).
* Multiple VPA resources matching the same pod have undefined behavior.

# Related links

* [Design
  proposal](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/autoscaling/vertical-pod-autoscaler.md)
* [API
  definition](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/pkg/apis/poc.autoscaling.k8s.io/v1alpha1/types.go)
