# Docker Registry

Docker provide an open source registry that allows users to store and manage
their Docker images. This can be run locally or on a remote server. See the
[Docker Registry] documentation for more information.

This repository contains manifest files for deploying a private Docker Registry
on Kubernetes. I use this on a [K3s] deployment to store my images for my
homelab.

## Authentication

HTTP basic authentication can be used to authenticate users, from the Docker
Registry documentation:

> The simplest way to achieve access restriction is through basic authentication
> (this is very similar to other web servers’ basic authentication mechanism).
> This example uses native basic authentication using htpasswd to store the
> secrets.

Note that as per the warning in that documentation basic authentication can only
be used if the registry is secured using TLS. You will see further down that I
have an ingress route that is secured with TLS using Lets Encrypt for this
purpose.

Create the password file, see the [Apache htpasswd] documentation for more
information on this command.

```sh
$ htpasswd -b -c -B htpasswd docker-registry registry-password!
Adding password for user docker-registry
```

Add the generated password file as a Kubernetes secret.

```sh
$ kubectl create secret generic basic-auth --from-file=./htpasswd -n registry
secret/basic-auth created
```

## Storage Backend

The Docker Registry can be configured with different options for storing images,
see the [storage] documentation for more information. I am storing my images
using the S3 storage backend, I am using my own hosted [MinIO] server to provide
this, not AWS for example.

The [100-registry-secrets.yaml] file contains the credentials for the S3
storage. This is the access key and secret access key of the registry user used
to connect to the S3 server for the bucket to store the images in. The values
for the secrets need to be base64 encoded.

## Running the Registry

After adding your secrets and configuring the storage backend, whatever option
you decide upon, the registry can be started.

```sh
kubectl apply -f 100-registry-secrets.yaml -f 200-registry-service.yaml -f 300-registry-deployment.yaml
```

Check the status of the deployment and logs to ensure it has started correctly.

```plain
kubectl get events -n registry -w
```

```plain
kubectl logs -f -n registry -l app=registry
```

## Ingress Route

I use Traefik as my Kubernetes Ingress Controller. See my
[k3s-traefik-v2-kubernetes-crd] GitHub repository for information on my setup.

The [400-registry-ingressroute.yaml] file contains the configuration for the
ingress route. The `tlsresolver` certificate resolver is configured within
Traefik to use the `tlsChallenge` type with Lets Encrypt to provide the
certificates to secure this registry. TLS is required to be able to use basic
authentication when logging into the registry.

## Docker Registry UI

There is a web based UI that can be deployed for the registry. See the
[joxit/docker-registry-ui] GitHub repository for more information on this image
and configuration. The below Kubernetes manifest files can be applied to deploy
this. Update the configuration in those to point to your registry.

- [500-registry-ui-service.yaml]
- [600-registry-ui-deployment.yaml]
- [700-registry-ui-ingressroute.yaml]

You can now access the registry UI at <https://registry.mydomain.io/ui/> (note
that the trailing slash in the url is required). The `/ui/` path is used so that
the ingress route rules can route traffic to this service on the same hostname
as the registry. This path will be stripped off by the `stripprefix` middleware
in the [700-registry-ui-ingressroute.yaml] file. This is just my approach,
others could be used, e.g. hosting the UI on a different hostname.

[100-registry-secrets.yaml]: ./100-registry-secrets.yaml
[200-registry-service.yaml]: ./200-registry-service.yaml
[300-registry-deployment.yaml]: ./300-registry-deployment.yaml
[400-registry-ingressroute.yaml]: ./400-registry-ingressroute.yaml
[500-registry-ui-service.yaml]: ./500-registry-ui-service.yaml
[600-registry-ui-deployment.yaml]: ./600-registry-ui-deployment.yaml
[700-registry-ui-ingressroute.yaml]: ./700-registry-ui-ingressroute.yaml
[apache htpasswd]: https://httpd.apache.org/docs/2.4/programs/htpasswd.html
[docker registry]: https://docs.docker.com/registry/
[joxit/docker-registry-ui]: https://github.com/Joxit/docker-registry-ui
[k3s]: https://k3s.io/
[k3s-traefik-v2-kubernetes-crd]:
  https://github.com/sleighzy/k3s-traefik-v2-kubernetes-crd
[minio]: https://min.io/
[storage]: https://docs.docker.com/registry/configuration/#storage
