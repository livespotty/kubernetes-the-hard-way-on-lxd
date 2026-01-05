# Prerequisites

## Initializing LXD

If you never used LXD on your host, you need to initialize it:
Ensure you have lxc version 5.x and above

### Note:

By default, if you have used multipass, lxd and lxc is already installed.

```
lxc --version
Installing LXD snap, please be patient.
5.21.4 LTS

```

Create a new storage pool, and select the backend to be `dir`, this is the only supported backend for this tutorial.

```
lxc storage create lxd-storage dir

```

You can now check the lxd storage by running:

```
ubuntu@k8s-hardway-18:~$ lxc storage list
+-------------+--------+----------------------------------------------------+-------------+---------+---------+
|    NAME     | DRIVER |                       SOURCE                       | DESCRIPTION | USED BY |  STATE  |
+-------------+--------+----------------------------------------------------+-------------+---------+---------+
| lxd-storage | dir    | /var/snap/lxd/common/lxd/storage-pools/lxd-storage |             | 0       | CREATED |
+-------------+--------+----------------------------------------------------+-------------+---------+---------+
```

You should see no containers created at this point.

## Creating containers profiles

We will use a special profile to run our containers, since some components require special access to modules to run. This is not safe for a production environment, and should be used only for this lab.
More info [here](https://ubuntu.com/kubernetes/charmed-k8s/docs/install-local).

create the profile configuration yaml with the following content:

```
cat <<EOF |tee kube-profile.yaml
config:
  limits.cpu: "2"
  limits.memory.swap: "false"
  boot.autostart: "false"
  linux.kernel_modules: ip_tables,ip6_tables,netlink_diag,nf_nat,overlay,br_netfilter
  raw.lxc: |
    lxc.apparmor.profile=unconfined
    lxc.mount.auto=proc:rw sys:rw cgroup:rw
    lxc.cgroup.devices.allow=a
    lxc.cap.drop=
  security.nesting: "true"
  security.privileged: "true"
description: ""
devices:
  aadisable:
    path: /sys/module/nf_conntrack/parameters/hashsize
    source: /dev/null
    type: disk
  aadisable1:
    path: /sys/module/apparmor/parameters/enabled
    source: /dev/null
    type: disk
EOF
```
Now create the profile:

```
 lxc profile create kube-profile
```

Set the profile with the properties from the yaml file:

```
cat kube-profile.yaml | lxc profile edit kube-profile
```

Check the profile content with:

```
lxc profile show kube-profile
```

Disable swap on your host:

```
sudo swapoff -a
```

Next: [Installing the Client Tools](02-client-tools.md)
