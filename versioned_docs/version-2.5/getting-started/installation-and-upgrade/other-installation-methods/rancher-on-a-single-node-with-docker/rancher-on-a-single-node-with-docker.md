---
title: Installing Rancher on a Single Node Using Docker
description: For development and testing environments only, use a Docker install. Install Docker on a single Linux host, and deploy Rancher with a single Docker container.
---

<head>
  <link rel="canonical" href="https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/other-installation-methods/rancher-on-a-single-node-with-docker"/>
</head>

Rancher can be installed by running a single Docker container.

In this installation scenario, you'll install Docker on a single Linux host, and then deploy Rancher on your host using a single Docker container.

> **Want to use an external load balancer?**
> See [Docker Install with an External Load Balancer](../../advanced-options/advanced-use-cases/configure-layer-7-nginx-load-balancer.md) instead.

A Docker installation of Rancher is recommended only for development and testing purposes. The ability to migrate Rancher to a high-availability cluster depends on the Rancher version:

The Rancher backup operator can be used to migrate Rancher from the single Docker container install to an installation on a high-availability Kubernetes cluster. For details, refer to the documentation on [migrating Rancher to a new cluster.](../../../../how-to-guides/new-user-guides/backup-restore-and-disaster-recovery/migrate-rancher-to-new-cluster.md)

### Privileged Access for Rancher v2.5+

When the Rancher server is deployed in the Docker container, a local Kubernetes cluster is installed within the container for Rancher to use. Because many features of Rancher run as deployments, and privileged mode is required to run containers within containers, you will need to install Rancher with the `--privileged` option.

## Requirements for OS, Docker, Hardware, and Networking

Make sure that your node fulfills the general [installation requirements.](../../installation-requirements/installation-requirements.md)

## 1. Provision Linux Host

Provision a single Linux host according to our [Requirements](../../installation-requirements/installation-requirements.md) to launch your Rancher server.

## 2. Choose an SSL Option and Install Rancher

For security purposes, SSL (Secure Sockets Layer) is required when using Rancher. SSL secures all Rancher network communication, like when you login or interact with a cluster.

> **Do you want to...**
>
> - Use a proxy? See [HTTP Proxy Configuration](../../../../reference-guides/single-node-rancher-in-docker/http-proxy-configuration.md)
> - Configure custom CA root certificate to access your services? See [Custom CA root certificate](../../../../reference-guides/single-node-rancher-in-docker/advanced-options.md#custom-ca-certificate)
> - Complete an Air Gap Installation? See [Air Gap Install](../air-gapped-helm-cli-install/air-gapped-helm-cli-install.md)
> - Record all transactions with the Rancher API? See [API Auditing](../../../../reference-guides/single-node-rancher-in-docker/advanced-options.md#api-audit-log)

Choose from the following options:

- [Option A: Default Rancher-generated Self-signed Certificate](#option-a-default-rancher-generated-self-signed-certificate)
- [Option B: Bring Your Own Certificate, Self-signed](#option-b-bring-your-own-certificate-self-signed)
- [Option C: Bring Your Own Certificate, Signed by a Recognized CA](#option-c-bring-your-own-certificate-signed-by-a-recognized-ca)
- [Option D: Let's Encrypt Certificate](#option-d-lets-encrypt-certificate)

### Option A: Default Rancher-generated Self-signed Certificate

If you are installing Rancher in a development or testing environment where identity verification isn't a concern, install Rancher using the self-signed certificate that it generates. This installation option omits the hassle of generating a certificate yourself.

Log into your Linux host, and then run the minimum installation command below.

As of Rancher v2.5, privileged access is [required.](#privileged-access-for-rancher-v25)

```bash
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  --privileged \
  rancher/rancher:latest
```

### Option B: Bring Your Own Certificate, Self-signed
In development or testing environments where your team will access your Rancher server, create a self-signed certificate for use with your install so that your team can verify they're connecting to your instance of Rancher.

> **Prerequisites:**
> Create a self-signed certificate using [OpenSSL](https://www.openssl.org/) or another method of your choice.
>
> - The certificate files must be in PEM format.
> - In your certificate file, include all intermediate certificates in the chain. Order your certificates with your certificate first, followed by the intermediates. For an example, see [Certificate Troubleshooting.](certificate-troubleshooting.md)

After creating your certificate, run the Docker command below to install Rancher. Use the `-v` flag and provide the path to your certificates to mount them in your container.

| Placeholder         | Description     |
| ------------------- | --------------------- |
| `<CERT_DIRECTORY>`  | The path to the directory containing your certificate files. |
| `<FULL_CHAIN.pem>`  | The path to your full certificate chain.                     |
| `<PRIVATE_KEY.pem>` | The path to the private key for your certificate.            |
| `<CA_CERTS.pem>`        | The path to the certificate authority's certificate.         |

As of Rancher v2.5, privileged access is [required.](#privileged-access-for-rancher-v25)

```bash
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /<CERT_DIRECTORY>/<FULL_CHAIN.pem>:/etc/rancher/ssl/cert.pem \
  -v /<CERT_DIRECTORY>/<PRIVATE_KEY.pem>:/etc/rancher/ssl/key.pem \
  -v /<CERT_DIRECTORY>/<CA_CERTS.pem>:/etc/rancher/ssl/cacerts.pem \
  --privileged \
  rancher/rancher:latest
```

### Option C: Bring Your Own Certificate, Signed by a Recognized CA

In production environments where you're exposing an app publicly, use a certificate signed by a recognized CA so that your user base doesn't encounter security warnings.

> **Prerequisites:**
>
> - The certificate files must be in PEM format.
> - In your certificate file, include all intermediate certificates provided by the recognized CA. Order your certificates with your certificate first, followed by the intermediates. For an example, see [Certificate Troubleshooting.](certificate-troubleshooting.md)

After obtaining your certificate, run the Docker command below.

- Use the `-v` flag and provide the path to your certificates to mount them in your container. Because your certificate is signed by a recognized CA, mounting an additional CA certificate file is unnecessary.
- Use the `--no-cacerts` as argument to the container to disable the default CA certificate generated by Rancher.

| Placeholder         | Description    |
| ------------------- | ----------------------------- |
| `<CERT_DIRECTORY>`  | The path to the directory containing your certificate files. |
| `<FULL_CHAIN.pem>`  | The path to your full certificate chain.                     |
| `<PRIVATE_KEY.pem>` | The path to the private key for your certificate. |

As of Rancher v2.5, privileged access is [required.](#privileged-access-for-rancher-v25)

```bash
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /<CERT_DIRECTORY>/<FULL_CHAIN.pem>:/etc/rancher/ssl/cert.pem \
  -v /<CERT_DIRECTORY>/<PRIVATE_KEY.pem>:/etc/rancher/ssl/key.pem \
  --privileged \
  rancher/rancher:latest \
  --no-cacerts
```

### Option D: Let's Encrypt Certificate

> **Remember:** Let's Encrypt provides rate limits for requesting new certificates. Therefore, limit how often you create or destroy the container. For more information, see [Let's Encrypt documentation on rate limits](https://letsencrypt.org/docs/rate-limits/).

For production environments, you also have the option of using [Let's Encrypt](https://letsencrypt.org/) certificates. Let's Encrypt uses an http-01 challenge to verify that you have control over your domain. You can confirm that you control the domain by pointing the hostname that you want to use for Rancher access (for example, `rancher.mydomain.com`) to the IP of the machine it is running on. You can bind the hostname to the IP address by creating an A record in DNS.

> **Prerequisites:**
>
> - Let's Encrypt is an Internet service. Therefore, this option cannot be used in an internal/air gapped network.
> - Create a record in your DNS that binds your Linux host IP address to the hostname that you want to use for Rancher access (`rancher.mydomain.com` for example).
> - Open port `TCP/80` on your Linux host. The Let's Encrypt http-01 challenge can come from any source IP address, so port `TCP/80` must be open to all IP addresses.

After you fulfill the prerequisites, you can install Rancher using a Let's Encrypt certificate by running the following command.

| Placeholder       | Description         |
| ----------------- | ------------------- |
| `<YOUR.DNS.NAME>` | Your domain address |

As of Rancher v2.5, privileged access is [required.](#privileged-access-for-rancher-v25)

```
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  --privileged \
  rancher/rancher:latest \
  --acme-domain <YOUR.DNS.NAME>
```

## Advanced Options

When installing Rancher on a single node with Docker, there are several advanced options that can be enabled:

- Custom CA Certificate
- API Audit Log
- TLS Settings
- Air Gap
- Persistent Data
- Running `rancher/rancher` and `rancher/rancher-agent` on the Same Node

Refer to [this page](../../../../reference-guides/single-node-rancher-in-docker/advanced-options.md) for details.

## Troubleshooting

Refer to [this page](certificate-troubleshooting.md) for frequently asked questions and troubleshooting tips.

## What's Next?

- **Recommended:** Review Single Node [Backup](../../../../how-to-guides/new-user-guides/backup-restore-and-disaster-recovery/back-up-docker-installed-rancher.md) and [Restore](../../../../how-to-guides/new-user-guides/backup-restore-and-disaster-recovery/restore-docker-installed-rancher.md). Although you don't have any data you need to back up right now, we recommend creating backups after regular Rancher use.
- Create a Kubernetes cluster: [Provisioning Kubernetes Clusters](../../../../how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/kubernetes-clusters-in-rancher-setup.md).
