# Kubefed Demo

This demo goes through deploying a federated minikube environment and deploying and managing an application between the two.

We will deploy two Minikube profiles (`cluster1` and `cluster2`) with `cluster1` being the primary cluster. After federating the two clusters, we will deploy an federated application to `cluster1` and propegate it to `cluster2`.

## Install

### Minikube

Install [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)

### Kubefedctl

Install the [Kubefedctl CLI](https://github.com/kubernetes-sigs/kubefed/blob/master/docs/userguide.md#kubefedctl-cli).

### Helm CLI

Install the [Helm CLI](https://helm.sh/docs/using_helm/#installing-the-helm-client)

## Infra Setup

The minikube version used in this demo is

```bash
$ minikube version
minikube version: v1.2.0
```

Create two minikube instances

```bash
minikube start -p cluster1 --kubernetes-version v1.15.0
minikube start -p cluster2 --kubernetes-version v1.15.0
```

## Deploy KubeFed

Since `cluster1` will run the KubeFed Control plane, we need to run the following commands against `cluster1`.

### Install And Configure Tiller

```bash
kubectl --cluster=cluster1 apply -f helm-rbac.yaml
helm --kube-context=cluster1 init --service-account tiller
```

### Deploy Kubefed Chart

Using the same version variable from above, we want to install the corresponding chart

```bash
$ # Add the kubefed chart repo
$ helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
$ #Ensure the repo is present
$ helm repo list | grep kubefed-charts
kubefed-charts	https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
# Update the repo, incase it was already present, or a new version was out since it was installed
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "kubefed-charts" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
# Deploy the chart to cluster1
$ helm --kube-context cluster1 install kubefed-charts/kubefed --name kubefed --version=0.1.0-rc3 --namespace kube-federation-system
NAME:   kubefed
LAST DEPLOYED: Wed Jun 26 07:54:41 2019
NAMESPACE: kube-federation-system
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME                                AGE
kubefed-role                        1s
system:kubefed:admission-requester  1s

==> v1/ClusterRoleBinding
NAME                                      AGE
kubefed-admission-webhook:anonymous-auth  1s
kubefed-admission-webhook:auth-delegator  1s
kubefed-rolebinding                       1s

==> v1/Deployment
NAME                        READY  UP-TO-DATE  AVAILABLE  AGE
kubefed-admission-webhook   0/1    1           0          1s
kubefed-controller-manager  0/2    2           0          1s

==> v1/Pod(related)
NAME                                        READY  STATUS             RESTARTS  AGE
kubefed-admission-webhook-55d77f5d8b-btw9x  0/1    ContainerCreating  0         1s
kubefed-controller-manager-c858df58-rwjvx   0/1    ContainerCreating  0         1s
kubefed-controller-manager-c858df58-wbckk   0/1    ContainerCreating  0         1s

==> v1/Role
NAME                            AGE
kubefed-admission-webhook-role  1s
kubefed-config-role             1s

==> v1/RoleBinding
NAME                                           AGE
kubefed-admission-webhook-rolebinding          1s
kubefed-admission-webhook:apiextension-viewer  1s
kubefed-config-rolebinding                     1s

==> v1/Secret
NAME                                    TYPE               DATA  AGE
kubefed-admission-webhook-serving-cert  kubernetes.io/tls  2     1s

==> v1/Service
NAME                       TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)  AGE
kubefed-admission-webhook  ClusterIP  10.99.10.191  <none>       443/TCP  1s

==> v1/ServiceAccount
NAME                       SECRETS  AGE
kubefed-admission-webhook  1        1s
kubefed-controller         1        1s

==> v1beta1/FederatedTypeConfig
NAME                                    AGE
clusterroles.rbac.authorization.k8s.io  1s
configmaps                              1s
deployments.apps                        1s
ingresses.extensions                    1s
jobs.batch                              1s
namespaces                              1s
replicasets.apps                        1s
secrets                                 1s
serviceaccounts                         1s
services                                1s

==> v1beta1/KubeFedConfig
NAME     AGE
kubefed  1s

==> v1beta1/MutatingWebhookConfiguration
NAME                          AGE
mutation.core.kubefed.k8s.io  1s

==> v1beta1/ValidatingWebhookConfiguration
NAME                             AGE
validations.core.kubefed.k8s.io  1s
```

### Cluster Registration

Cluster 1 needs to know that it's part of the federation that's controlled by itself:

```bash
$ kubefedctl join cluster1 --cluster-context cluster1 --host-cluster-context cluster1 --v=2
I0626 07:56:47.686465   41157 join.go:159] Args and flags: name cluster1, host: cluster1, host-system-namespace: kube-federation-system, kubeconfig: , cluster-context: cluster1, secret-name: , dry-run: false
I0626 07:56:47.794308   41157 join.go:219] Performing preflight checks.
I0626 07:56:47.796868   41157 join.go:225] Creating kube-federation-system namespace in joining cluster
I0626 07:56:47.798672   41157 join.go:352] Already existing kube-federation-system namespace
I0626 07:56:47.798683   41157 join.go:233] Created kube-federation-system namespace in joining cluster
I0626 07:56:47.798690   41157 join.go:236] Creating cluster credentials secret
I0626 07:56:47.798979   41157 join.go:372] Creating service account in joining cluster: cluster1
I0626 07:56:47.804848   41157 join.go:382] Created service account: cluster1-cluster1 in joining cluster: cluster1
I0626 07:56:47.804871   41157 join.go:410] Creating cluster role and binding for service account: cluster1-cluster1 in joining cluster: cluster1
I0626 07:56:47.822188   41157 join.go:419] Created cluster role and binding for service account: cluster1-cluster1 in joining cluster: cluster1
I0626 07:56:47.822209   41157 join.go:423] Creating secret in host cluster: cluster1
I0626 07:56:47.828748   41157 join.go:812] Using secret named: cluster1-cluster1-token-hz6bw
I0626 07:56:47.832736   41157 join.go:855] Created secret in host cluster named: cluster1-hm4vv
I0626 07:56:47.832754   41157 join.go:432] Created secret in host cluster: cluster1
I0626 07:56:47.832763   41157 join.go:246] Cluster credentials secret created
I0626 07:56:47.832769   41157 join.go:248] Creating federated cluster resource
I0626 07:56:47.852460   41157 join.go:257] Created federated cluster resource
```

Cluster 1 Also needs to know about Cluster 2. Provide the connection to cluster 2 via the local kube config `cluster2`:

```bash
$kubefedctl join cluster2 --cluster-context cluster2 --host-cluster-context cluster1 --v=2
I0626 07:56:47.892542   41158 join.go:159] Args and flags: name cluster2, host: cluster1, host-system-namespace: kube-federation-system, kubeconfig: , cluster-context: cluster2, secret-name: , dry-run: false
I0626 07:56:47.993572   41158 join.go:219] Performing preflight checks.
I0626 07:56:48.002908   41158 join.go:225] Creating kube-federation-system namespace in joining cluster
I0626 07:56:48.012887   41158 join.go:233] Created kube-federation-system namespace in joining cluster
I0626 07:56:48.012921   41158 join.go:236] Creating cluster credentials secret
I0626 07:56:48.012931   41158 join.go:372] Creating service account in joining cluster: cluster2
I0626 07:56:48.018477   41158 join.go:382] Created service account: cluster2-cluster1 in joining cluster: cluster2
I0626 07:56:48.018494   41158 join.go:410] Creating cluster role and binding for service account: cluster2-cluster1 in joining cluster: cluster2
I0626 07:56:48.045135   41158 join.go:419] Created cluster role and binding for service account: cluster2-cluster1 in joining cluster: cluster2
I0626 07:56:48.045163   41158 join.go:423] Creating secret in host cluster: cluster1
I0626 07:56:48.052081   41158 join.go:812] Using secret named: cluster2-cluster1-token-dh4ht
I0626 07:56:48.055595   41158 join.go:855] Created secret in host cluster named: cluster2-djjdh
I0626 07:56:48.055636   41158 join.go:432] Created secret in host cluster: cluster1
I0626 07:56:48.055650   41158 join.go:246] Cluster credentials secret created
I0626 07:56:48.055658   41158 join.go:248] Creating federated cluster resource
I0626 07:56:48.074932   41158 join.go:257] Created federated cluster resource
```

Upon running these commands, we can see the configured federated clusters here:

```bash
$ kubectl --cluster=cluster1 -n kube-federation-system get kubefedclusters
NAME       READY   AGE
cluster1   True    2m19s
cluster2   True    2m18s
```

## Deploy a Federated Application

### Setup Variables

```bash
IP1=`minikube -p cluster1 ip`
IP2=`minikube -p cluster2 ip`
```

Create an application and deploy a sample application:

```bash
$ kubectl --cluster=cluster1 create ns request-log
$ kubectl --cluster=cluster1 apply -n request-log -f https://raw.githubusercontent.com/runyontr/request-log/master/kubernetes/deployment.yaml
deployment.apps/request-log created
service/request-log created
$ kubectl --cluster=cluster1 set env -n request-log deployment/request-log HEADER=cluster1
```

Verify the application is up and running via:

```bash
$ PORT1=`kubectl --cluster=cluster1 get svc -n request-log request-log -o jsonpath="{ .spec.ports[0].nodePort }"`
$ http http://${IP1}:${PORT1} -b | head -n 1
cluster1
```

### Working

You can federate the namespace by either applying the explicit federatednamespace object:

```bash
kubectl --cluster=cluster1 apply -f federation/federatednamespace.yaml
```

or federating with `kubefedctl` via:

```bash
TODO
```

Now the namespace is up in cluster2, but the objects were not propegated over:

```bash
$ kubectl --cluster=cluster2 get ns
NAME                     STATUS   AGE
default                  Active   5d
kube-node-lease          Active   5d
kube-public              Active   5d
kube-system              Active   5d
request-log              Active   45s
$ kubectl --cluster=cluster2 get all -n request-log
No resources found.
```

To move the deployment, we have to

### Federate the deployment

The object types that are federated can be shown via

```bash
$ kubefedctl federate deployments.apps request-log -n request-log --host-cluster-context=cluster1
I0701 07:23:58.835548   12539 federate.go:480] Successfully created FederatedDeployment "request-log/request-log" from Deployment
kubectl --cluster=cluster1 get federateddeployments.types.kubefed.k8s.io -n request-log
NAME          AGE
request-log   72s
```

We also need to deploy the service object:

```bash
$ kubefedctl federate service request-log -n request-log --host-cluster-context=cluster1
```

Now we can see the application running:

```bash
$ PORT2=`kubectl --cluster=cluster2 get svc -n request-log request-log -o jsonpath="{ .spec.ports[0].nodePort }"`
$ http http://${IP2}:${PORT2} -b | head -n 1
cluster1
```

### Adjusting the deployment

To scale the deployment, its no longer possible to control the specs from the initial deployment object:

```bash
$ kubectl --cluster=cluster1 scale deployment -n request-log request-log --replicas=1
deployment.extensions/request-log scaled
$ kubectl --cluster=cluster1 get deployments -n request-log
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
request-log   3/3     3            3           4d22h
```

To adjust, we need to modify the `federateddeployment` object:

```bash
$ kubectl --cluster=cluster1 -n request-log patch federateddeployments.types.kubefed.k8s.io request-log -p '{"spec": {"template": {"spec": {"replicas":1}}}}' --type merge
federateddeployment.types.kubefed.k8s.io/request-log patched
$ kubectl --cluster=cluster2 get all -n request-log
NAME                               READY   STATUS    RESTARTS   AGE
pod/request-log-7f4548d78d-tljm6   1/1     Running   0          25m

NAME                  TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/request-log   LoadBalancer   10.98.239.77   <pending>     80:30674/TCP   19m

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/request-log   1/1     1            1           25m

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/request-log-7f4548d78d   1         1         1       25m
```

# Cluster specific overrides

In order to update a value for a specific cluster, we can use the `overrides` section of the federated deployment. This section is defined on a per cluster basis. We will update the environment variable for `cluster2`:

```bash
kind: FederatedDeployment
...
spec:
  overrides:
  - clusterName: cluster2
    clusterOverrides:
    - path: /spec/replicas
      value: 3
    - path: /spec/template/spec/containers/0/env/0/value
      value: cluster2
```

And we can see the differences:

```bash
$ kubectl --cluster=cluster1 get deployments -n request-log request-log -o jsonpath="{ .spec.replicas }"
1
$ kubectl --cluster=cluster2 get deployments -n request-log request-log -o jsonpath="{ .spec.replicas }"
3
$ kubectl --cluster=cluster1 get deployments -n request-log request-log -o jsonpath="{ .spec.template.spec.containers[0].env[0].value }"
cluster1
$ kubectl --cluster=cluster2 get deployments -n request-log request-log -o jsonpath="{ .spec.template.spec.containers[0].env[0].value }"
cluster2
$ http http://${IP1}:${PORT1} -b | head -n 1
cluster1
$ http http://${IP2}:${PORT2} -b | head -n 1
cluster2
```
