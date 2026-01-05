# Installing the Client Tools

In this lab you will install the command line utilities required to complete this tutorial: [openssl](https://www.openssl.org/), and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).

## Install OpenSSL

`openssl` command line utility will be used to provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) and generate TLS certificates.

Download and install `openssl` from the official release binaries:

### Linux

```
sudo apt-get install openssl
```

### Verification

Verify `openssl` version 3.x or higher is installed:

```
openssl version
```

> output

```
OpenSSL 3.6.0 1 Oct 2025 (Library: OpenSSL 3.6.0 1 Oct 2025)
```


Next: [Provisioning Compute Resources](03-compute-resources.md)
