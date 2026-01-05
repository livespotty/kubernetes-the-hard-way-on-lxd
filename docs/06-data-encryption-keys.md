# Generating the Data Encryption Config and Key

Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) cluster data at rest.

In this lab you will generate an encryption key and an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

## The Encryption Key

Generate an encryption key:

```
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

```

## The Encryption Config File

Create the `encryption-config.yaml` encryption config file:

```
envsubst < configs/encryption-config.yaml \
  > encryption-config.yaml
```

Copy the `encryption-config.yaml` encryption config file to each master instance:

```
for instance in master-0 master-1 master-2; do
  lxc file push encryption-config.yaml ${instance}/home/ubuntu/  
done
```

Next: [Bootstrapping the etcd Cluster](07-bootstrapping-etcd.md)
