# Bootstrapping the Kubernetes Control Plane

In this lab you will bootstrap the Kubernetes control plane across three compute instances and configure it for high availability. You will also create an external load balancer that exposes the Kubernetes API Servers to remote clients. The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

Create the Kubernetes configuration directory:

```
{
for instance in master-0 master-1 master-2; do
  lxc exec ${instance} -- mkdir -p /etc/kubernetes/config
done
}
```
Install the Kubernetes binaries:

```
{
  for instance in master-0 master-1 master-2; do
    lxc file push downloads/controller/kube-apiserver ${instance}/usr/local/bin/
    lxc file push downloads/controller/kube-controller-manager ${instance}/usr/local/bin/
    lxc file push downloads/controller/kube-scheduler ${instance}/usr/local/bin/
    lxc file push downloads/client/kubectl ${instance}/usr/local/bin/
    lxc file push units/kube-apiserver.service ${instance}/home/ubuntu/
    lxc file push units/kube-controller-manager.service ${instance}/home/ubuntu/
    lxc file push units/kube-scheduler.service ${instance}/home/ubuntu/
    lxc file push configs/kube-scheduler.yaml ${instance}/home/ubuntu/
    lxc file push configs/kube-apiserver-to-kubelet.yaml ${instance}/home/ubuntu/
  done
}
```

### Configure the Kubernetes API Server

```
{
for instance in master-0 master-1 master-2; do
  lxc exec ${instance} -- mkdir -p /var/lib/kubernetes/
  lxc file push ca.crt ${instance}/var/lib/kubernetes/
  lxc file push ca.key ${instance}/var/lib/kubernetes/
  lxc file push kube-api-server.key ${instance}/var/lib/kubernetes/
  lxc file push kube-api-server.crt ${instance}/var/lib/kubernetes/
  lxc file push service-accounts.key ${instance}/var/lib/kubernetes/
  lxc file push service-accounts.crt ${instance}/var/lib/kubernetes/
  lxc file push encryption-config.yaml ${instance}/var/lib/kubernetes/
 done
}
```

The instance internal IP address will be used to advertise the API Server to members of the cluster. Retrieve the internal IP address for the current compute instance:

```
{
for instance in master-0 master-1 master-2; do
  lxc exec ${instance} -- cp /home/ubuntu/kube-apiserver.service /etc/systemd/system/
done
}
```

### Configure the Kubernetes Controller Manager

Move the `kube-controller-manager` kubeconfig into place and create the `kube-controller-manager.service` systemd unit file::

```
for instance in master-0 master-1 master-2; do
  lxc file push kube-controller-manager.kubeconfig ${instance}/var/lib/kubernetes/
  lxc file push units/kube-controller-manager.service ${instance}/etc/systemd/system/

done
```

### Configure the Kubernetes Scheduler

Create the `kube-scheduler.yaml` and the `kube-scheduler.service` configuration files:

```
for instance in master-0 master-1 master-2; do
  lxc file push kube-scheduler.kubeconfig ${instance}/var/lib/kubernetes/
  lxc file push units/kube-scheduler.service ${instance}/etc/systemd/system/
  lxc file push configs/kube-scheduler.yaml ${instance}/etc/kubernetes/config/
done
```

### Start the Master Services

```
{
for instance in master-0 master-1 master-2; do
  lxc exec ${instance} -- systemctl daemon-reload
  lxc exec ${instance} -- systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  lxc exec ${instance} -- systemctl start kube-apiserver kube-controller-manager kube-scheduler
done
}
```

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.

### Verification

For this part of the lab, we will send the commands to haproxy, which will loadbalance to the master nodes, for that, update the admin.kubeconfig to the haproxy address on the server you used to create the lxc containers, do not change the file on any of the master nodes:

```
 vi admin.kubeconfig
```

Change the field server to 10.0.1.100:

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data:<CERTIFICATE DATA NOT REPRODUCED HERE>
    server: https://10.0.1.100:6443
  name: kubernetes-the-hard-way
contexts:
- context:
    cluster: kubernetes-the-hard-way
    user: admin
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data:<CERTIFICATE DATA NOT REPRODUCED HERE>
    client-key-data:<CERTIFICATE DATA NOT REPRODUCED HERE>
```

Now check the status of cluster:

```
kubectl cluster-info --kubeconfig admin.kubeconfig
```

You will have to move the `/home/ubuntu/` folder to run this command if you wanted to check this command inside master nodes

```
NAME                 STATUS    MESSAGE              ERROR
Kubernetes control plane is running at https://10.0.1.100:6443
```

## RBAC for Kubelet Authorization

In this section you will configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.

> This tutorial sets the Kubelet `--authorization-mode` flag to `Webhook`. Webhook mode uses the [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API to determine authorization.

Create the `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) with permissions to access the Kubelet API and perform most common tasks associated with managing pods:

```
kubectl apply -f configs/kube-apiserver-to-kubelet.yaml \
  --kubeconfig admin.kubeconfig

```


### Verification

Make a HTTP request for the Kubernetes version info on the haproxy:

```
curl --cacert ca.crt https://10.0.1.100:6443/version
```

> output

```
{
  "major": "1",
  "minor": "34",
  "emulationMajor": "1",
  "emulationMinor": "34",
  "minCompatibilityMajor": "1",
  "minCompatibilityMinor": "33",
  "gitVersion": "v1.34.3",
  "gitCommit": "df11db1c0f08fab3c0baee1e5ce6efbf816af7f1",
  "gitTreeState": "clean",
  "buildDate": "2025-12-09T14:59:13Z",
  "goVersion": "go1.24.11",
  "compiler": "gc",
  "platform": "linux/arm64"
}
```

Next: [Bootstrapping the Kubernetes Worker Nodes](09-bootstrapping-kubernetes-workers.md)
