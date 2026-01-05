# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap three Kubernetes worker nodes. The following components will be installed on each node: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

Install the OS dependencies:

```
{
for instance in worker-0 worker-1 worker-2; do
  lxc exec ${instance} -- apt-get update
  lxc exec ${instance} -- apt-get -y install socat conntrack ipset
done
}
```

> The socat binary enables support for the `kubectl port-forward` command.

Create the installation directories:

```
{
for instance in worker-0 worker-1 worker-2; do
  lxc exec ${instance} -- mkdir -p /etc/cni/net.d
  lxc exec ${instance} -- mkdir -p /opt/cni/bin
  lxc exec ${instance} -- mkdir -p /var/lib/kubelet
  lxc exec ${instance} -- mkdir -p /var/lib/kube-proxy
  lxc exec ${instance} -- mkdir -p /var/lib/kubernetes
  lxc exec ${instance} -- mkdir -p /var/run/kubernetes
  lxc exec ${instance} -- mkdir -p /etc/containerd/
done
}
```

Install the worker binaries:

```
{
  for instance in worker-0 worker-1 worker-2; do
    lxc file push downloads/client/kubectl ${instance}/usr/local/bin/
    lxc file push downloads/worker/kube-proxy ${instance}/usr/local/bin/
    lxc file push downloads/worker/kubelet ${instance}/usr/local/bin/
    lxc file push downloads/worker/runc ${instance}/usr/local/bin/
    lxc file push downloads/cni-plugins/* ${instance}/opt/cni/bin/
    lxc file push downloads/worker/containerd ${instance}/bin/
    lxc file push downloads/worker/containerd-shim-runc-v2 ${instance}/bin/
    lxc file push downloads/worker/containerd-stress ${instance}/bin/
  done
}
```

### Configure CNI Networking

Create the `bridge` network configuration file:

```
{
for instance in 0 1 2; do

POD_CIDR=10.1.1${instance}.0/24

cat <<EOF | tee 10-bridge.conf
{
    "cniVersion": "1.0.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF

lxc file push 10-bridge.conf worker-${instance}/etc/cni/net.d/

done
}
```

### Configure the Kubelet

Create the `kubelet-config.yaml` configuration file:

```
for instance in 0 1 2; do

POD_CIDR=10.1.${instance}.0/16

cat <<EOF | tee kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/worker-${instance}.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/worker-${instance}.key"
EOF

lxc file push kubelet-config.yaml worker-${instance}/var/lib/kubelet/

lxc file push worker-${instance}.key  worker-${instance}/var/lib/kubelet/
lxc file push worker-${instance}.crt worker-${instance}/var/lib/kubelet/
lxc file push ca.crt worker-${instance}/var/lib/kubernetes/

done
```

### Copy all the configuration files to all workers

```
for instance in worker-0 worker-1 worker-2; do
    lxc file push configs/99-loopback.conf ${instance}/etc/cni/net.d/
    lxc file push configs/containerd-config.toml ${instance}/etc/containerd/config.toml
    lxc file push units/containerd.service ${instance}/etc/systemd/system/
    lxc file push units/kubelet.service ${instance}/etc/systemd/system/
    lxc file push configs/kube-proxy-config.yaml ${instance}/var/lib/kube-proxy/
    lxc file push units/kube-proxy.service ${instance}/etc/systemd/system/
done
```

To ensure network traffic crossing the CNI `bridge` network is processed by `iptables`, load and configure the `br-netfilter` kernel module:

```bash
{
  modprobe br-netfilter
  echo "br-netfilter" >> /etc/modules-load.d/modules.conf
}
```

```bash
{
  echo "net.bridge.bridge-nf-call-iptables = 1" \
    >> /etc/sysctl.d/kubernetes.conf
  echo "net.bridge.bridge-nf-call-ip6tables = 1" \
    >> /etc/sysctl.d/kubernetes.conf
  sysctl -p /etc/sysctl.d/kubernetes.conf
}
```


### Start the Worker Services

```
{
for instance in worker-0 worker-1 worker-2; do
  lxc exec ${instance} -- systemctl daemon-reload
  lxc exec ${instance} -- systemctl enable containerd kubelet kube-proxy
  lxc exec ${instance} -- systemctl start containerd kubelet kube-proxy

done
}
```

## SWAP issues

If your nodes failed to start (check the journalctl in one of the workers), there is a good chance that Kubelet is failing because swap is active. One way to fix this is to disable swap in your main server, not the container, with the command:

```
sudo swapoff -a
```

You need at least 8GB of memory to run everything without Swap with some performance. This Lab was tested in a machine with 8GB of ram.

Note: There is hack that needs to be done on all worker nodes, ensure this is in place when you restart the nodes

```
ln -s /dev/console /dev/kmsg

```

## Recommendation

Have a handy shell script that you will run every time when you restart worker nodes

```
{
for instance in worker-0 worker-1 worker-2; do
  lxc exec ${instance} -- ln -s /dev/console /dev/kmsg
done
}
```

## Verification

> The compute instances created in this tutorial will not have permission to complete this section. Run the following commands from the same machine used to create the compute instances.

List the registered Kubernetes nodes:

```
kubectl get nodes --kubeconfig admin.kubeconfig
```

> output

```
NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   35s   v1.34.3
worker-1   Ready    <none>   36s   v1.34.3
worker-2   Ready    <none>   36s   v1.34.3
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
