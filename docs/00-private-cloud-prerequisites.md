# Using Multipass to create private cloud environment local to your workstation.

After you install Multipass on your Mac/Windows
see instructions here [multipass](https://canonical.com/multipass)

# Launching VM using multipass

```
multipass launch --name k8s-hardway-26 --memory 5G --disk 20G

multipass exec k8s-hardway-26 /bin/bash

```

You will get the following prompt

```
ubuntu@k8s-hardway-26:~\$

```

Setting up the above host as Jump server

Sync GitHub Repository

Now it's time to download a copy of this tutorial which contains the configuration files and templates that will be used build your Kubernetes cluster from the ground up. Clone the Kubernetes The Hard Way git repository using the git command:

```
git clone https://github.com/livespotty/kubernetes-the-hard-way-on-lxd.git
```

Change into the `kubernetes-the-hard-way` directory:

```bash
cd kubernetes-the-hard-way
```

This will be the working directory for the rest of the tutorial. If you ever get lost run the `pwd` command to verify you are in the right directory when running commands on the `jumpbox`:


### Download Binaries

In this section you will download the binaries for the various Kubernetes components. The binaries will be stored in the `downloads` directory on the `jumpbox`, which will reduce the amount of internet bandwidth required to complete this tutorial as we avoid downloading the binaries multiple times for each machine in our Kubernetes cluster.

The binaries that will be downloaded are listed in either the `downloads-amd64.txt` or `downloads-arm64.txt` file depending on your hardware architecture, which you can review using the `cat` command:

```bash
cat downloads-$(dpkg --print-architecture).txt
```

Download the binaries into a directory called `downloads` using the `wget` command:

```bash
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads-$(dpkg --print-architecture).txt
```

Depending on your internet connection speed it may take a while to download over `500` megabytes of binaries, and once the download is complete, you can list them using the `ls` command:

```bash
ls -oh downloads
```

Extract the component binaries from the release archives and organize them under the `downloads` directory.

```bash
{
  ARCH=$(dpkg --print-architecture)
  mkdir -p downloads/{client,cni-plugins,controller,worker}
  tar -xvf downloads/crictl-v1.35.0-linux-${ARCH}.tar.gz \
    -C downloads/worker/
  tar -xvf downloads/containerd-2.2.0-linux-${ARCH}.tar.gz \
    --strip-components 1 \
    -C downloads/worker/
  tar -xvf downloads/cni-plugins-linux-${ARCH}-v1.8.0.tgz \
    -C downloads/cni-plugins/
  tar -xvf downloads/etcd-v3.6.6-linux-${ARCH}.tar.gz \
    -C downloads/ \
    --strip-components 1 \
    etcd-v3.6.6-linux-${ARCH}/etcdctl \
    etcd-v3.6.6-linux-${ARCH}/etcd
  mv downloads/{etcdctl,kubectl} downloads/client/
  mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} \
    downloads/controller/
  mv downloads/{kubelet,kube-proxy} downloads/worker/
  mv downloads/runc.${ARCH} downloads/worker/runc
}
```

```bash
rm -rf downloads/*gz
```

Make the binaries executable.

```bash
{
  chmod +x downloads/{client,cni-plugins,controller,worker}/*
}
```

### Install kubectl

In this section you will install the `kubectl`, the official Kubernetes client command line tool, on the `jumpbox` machine. `kubectl` will be used to interact with the Kubernetes control plane once your cluster is provisioned later in this tutorial.

Use the `chmod` command to make the `kubectl` binary executable and move it to the `/usr/local/bin/` directory:

```bash
{
  sudo cp downloads/client/kubectl /usr/local/bin/
}
```

At this point `kubectl` is installed and can be verified by running the `kubectl` command:

```bash
kubectl version --client
```

```text
Client Version: v1.34.3
Kustomize Version: v5.7.1
```

At this point the `jumpbox` / `host` machine has been set up with all the command line tools and utilities necessary to complete the labs in this tutorial.

Note: In this host, we are going to create 3 master nodes (ie., 3 etcd), 3 worker nodes and haproxy node to route traffic to 3 master nodes.

Next: [prerequisites local cloud setup](01-prerequisites.md)
