# Smoke Test

In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.

## Data Encryption

In this section you will verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

```
 lxc exec master-0 -- etcdctl get /registry/secrets/default/kubernetes-the-hard-way | hexdump -C
```

> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 87 6b f7 42 c6 04 87  |:v1:key1:.k.B...|
00000050  2c cd 2e b9 f4 8d 85 95  b9 cb 8f 14 5c 16 17 c2  |,...........\...|
00000060  06 77 db 86 75 c2 04 ca  29 74 51 f0 d5 bb 1e 9a  |.w..u...)tQ.....|
00000070  20 45 13 0d c7 c6 e4 94  db e9 f6 d7 b2 45 a9 9e  | E...........E..|
00000080  62 23 6e 1c 84 b5 03 f7  74 86 a1 2b 7f bb d3 b5  |b#n.....t..+....|
00000090  19 c5 cf 65 4d 8e a3 2f  95 7d 38 e9 8a 39 af ff  |...eM../.}8..9..|
000000a0  83 e1 b2 51 5c 10 02 a4  61 32 72 3e 57 ae 1d 83  |...Q\...a2r>W...|
000000b0  8e 41 f8 e5 df 95 d2 3f  6b ee 98 a6 5f d2 17 b0  |.A.....?k..._...|
000000c0  ea 35 10 eb 70 7c 4f 8c  97 11 7d 61 12 47 b0 31  |.5..p|O...}a.G.1|
000000d0  f5 65 0f 58 2f e6 df 7e  99 94 be 36 f0 83 01 a9  |.e.X/..~...6....|
000000e0  0f a8 43 2b d3 ff 9b 7b  ec 4a c0 c9 11 dc 7d cb  |..C+...{.J....}.|
000000f0  b1 ac c5 50 1d c6 ce 98  ca 15 29 10 0d e2 ab a7  |...P......).....|
00000100  84 37 93 d5 7c 50 aa df  39 ea fe 9a 30 ce c7 38  |.7..|P..9...0..8|
00000110  5c 8b 93 57 f8 0b b1 7b  22 c7 bd e7 7c 7b 19 07  |\..W...{"...|{..|
00000120  1f 5a f2 32 10 fa d8 d3  8e 2f 0e f7 a5 ce e7 8f  |.Z.2...../......|
00000130  ea 94 13 c1 c4 08 63 ae  5b 6d a0 c4 c4 cd fb f2  |......c.[m......|
00000140  22 0c df 24 40 79 10 3a  20 5f b4 6c 16 61 6b 95  |"..$@y.: _.l.ak.|
00000150  95 d6 a3 e1 2a 05 fe 8e  60 0a                    |....*...`.|
0000015a
```

The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

## Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```
kubectl run nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```
kubectl get pods -l run=nginx -A
```

> output

```
NAME                    READY   STATUS    RESTARTS   AGE
nginx-dbddb74b8-6lxg2   1/1     Running   0          10s
```

### Port Forwarding

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```
POD_NAME=$(kubectl get pods -l app=nginx \
  -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```
kubectl port-forward $POD_NAME 8080:80
```

> output

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```
curl --head http://127.0.0.1:8080
```


Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```
kubectl logs $POD_NAME
```

> output

```
127.0.0.1 - - [30/Sep/2018:19:23:10 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.58.0" "-"
```

### Exec

In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```
nginx version: nginx/1.15.4
```

## Services

In this section you will verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```
kubectl expose deployment nginx --port 80 --type NodePort
```

> The LoadBalancer service type can not be used because your cluster is not configured with [cloud provider integration](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider). Setting up cloud provider integration is out of scope for this tutorial.

Retrieve the node port assigned to the `nginx` service:

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Retrieve the external IP address of a worker instance:

```
EXTERNAL_IP=$(lxc info worker-0 | grep --only-matching  '10.0.1.[0-9]*')
```

Make an HTTP request using the external IP address and the `nginx` node port:

```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```


Next: [Cleaning Up](13-cleanup.md)
