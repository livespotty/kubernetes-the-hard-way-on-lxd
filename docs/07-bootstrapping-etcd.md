# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/coreos/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

The commands in this lab must be run on each master instance: `master-0`, `master-1`, and `master-2`.
We will download all files in the main server, push all files to the masters, and run commands on each master.

## Bootstrapping an etcd Cluster Member


Extract and install the `etcd` server and the `etcdctl` command line utility:

```
{
for instance in master-0 master-1 master-2; do
  lxc file push downloads/controller/etcd ${instance}/home/ubuntu/
  lxc file push downloads/client/etcdctl ${instance}/home/ubuntu/
  lxc file push units/etcd.service ${instance}/home/ubuntu/
  lxc exec ${instance} -- mv /home/ubuntu/etcd /usr/local/bin/
  lxc exec ${instance} -- mv /home/ubuntu/etcdctl /usr/local/bin/
done
}
```

### Configure the etcd Server

```
{
for instance in master-0 master-1 master-2; do
  lxc exec ${instance} -- mkdir -p /etc/etcd /var/lib/etcd
  lxc exec ${instance} -- cp /home/ubuntu/ca.crt /etc/etcd/
  lxc exec ${instance} -- cp /home/ubuntu/kube-api-server.key /etc/etcd/
  lxc exec ${instance} -- cp /home/ubuntu/kube-api-server.crt /etc/etcd/
done
}
```

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address for the current compute instance

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance

Create the `etcd.service` systemd unit file for each of the master nodes:

```
{
for instance in 0 1 2; do

  INTERNAL_IP=10.0.2.1${instance}

  ETCD_NAME=master-${instance}

cat <<EOF | tee etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos
[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --peer-cert-file=/etc/etcd/kube-api-server.crt \
  --peer-key-file=/etc/etcd/kube-api-server.key \
  --cert-file=/etc/etcd/kube-api-server.crt \
  --key-file=/etc/etcd/kube-api-server.key \
  --trusted-ca-file=/etc/etcd/ca.crt \
  --peer-trusted-ca-file=/etc/etcd/ca.crt \
  --peer-client-cert-auth \
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://${INTERNAL_IP}:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster master-0=https://10.0.2.10:2380,master-1=https://10.0.2.11:2380,master-2=https://10.0.2.12:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF

  lxc file push etcd.service ${ETCD_NAME}/etc/systemd/system/

done
}
```

Copy the certs and key

```
{
for instance in master-0 master-1 master-2; do
  lxc exec ${instance} -- mkdir -p /etc/etcd /var/lib/etcd
  lxc exec ${instance} -- cp /home/ubuntu/ca.crt /etc/etcd/
  lxc exec ${instance} -- cp /home/ubuntu/kube-api-server.key /etc/etcd/
  lxc exec ${instance} -- cp /home/ubuntu/kube-api-server.crt /etc/etcd/
done
}
```

### Start the etcd Server

```
{
for instance in master-0 master-1 master-2; do
  lxc exec ${instance} -- systemctl daemon-reload
  lxc exec ${instance} -- systemctl enable etcd
  lxc exec ${instance} -- systemctl start etcd
done
}
```

> Remember to run the above commands on each master node: `master-0`, `master-1`, and `master-2`.

## Verification

Login to one of the master nodes, or you can check in all of them:

```
lxc exec master-0 -- sudo /bin/bash
```

Execute the verification command:

```
etcdctl member list --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/kube-api-server.crt --key=/etc/etcd/kube-api-server.key
```
> output

```
8ec27d324d7508b8, started, master-0, https://10.0.2.10:2380, https://10.0.2.10:2379, false
c18be024472ccb33, started, master-2, https://10.0.2.12:2380, https://10.0.2.12:2379, false
df64d128890b1397, started, master-1, https://10.0.2.11:2380, https://10.0.2.11:2379, false
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
