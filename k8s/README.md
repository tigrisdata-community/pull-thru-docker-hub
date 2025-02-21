# Making a pull-through cache for the docker hub using Kubernetes

If you do not have [cert-manager](https://cert-manager.io/) installed, install it.

Head to [storage.new](https://storage.new) and make a new bucket. You can name it whatever you want, but this tutorial will call it `pullthru-docker-hub`.

Click on the Access Keys tab in the sidebar and then click on Create New Access Key. Give it a descriptive name and grant it editor access to the bucket you just created.

Copy the `.env.example` file to `.env`. Fill in the following fields:

- `REGISTRY_STORAGE_S3_BUCKET`: Set it to your bucket name, such as `pullthru-docker-hub`.
- `REGISTRY_STORAGE_S3_ACCESSKEY`: Set it to the Tigris keypair key ID starting with `tid_`.
- `REGISTRY_STORAGE_S3_SECRETKEY`: Set it to the Tigris keypair secret key starting with `tsec_`.
- `REGISTRY_PROXY_USERNAME`: Set it to the username of a user on the Docker Hub
- `REGISTRY_PROXY_PASSWORD`: Set it to the password of that user on the Docker Hub

Open `ingress.yaml` in your editor. Change the domain `pt-dh.yolo-swag.com` to a domain name that you control.

Apply the Kubernetes configs using `kubectl`:

```text
kubectl apply -k .
```

That will create the following resources:

- A Deployment named `pt-dh` that runs the Docker pull-through cache, pulling environment variables from the Secret containing the `.env` file.
- A Secret with the contents of the `.env` file with a random name.
- A Service named `pt-dh` that gives the cluster a DNS name for your pull-through cache.
- An Ingress named `pt-dh` that exposes the `pt-dh` Service to the world over HTTPS.

The registry should be up within a moment or two, you can verify with the `kubectl get` command:

```text
$ kubectl get deploy/pt-dh
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
pt-dh   3/3     3            3           46m
```

## Testing

Now that the registry is up, you can test it using the `docker run` command:

```text
docker run --rm -it pt-dh.your-domain.example/library/hello-world
```

If all goes to plan, you should get a hello world message. The docker registry should write the image data to the bucket, which you can check using the AWS CLI:

```text
aws s3 ls s3://pullthru-docker-hub/docker/registry/v2/repositories/library/hello-world/
```

You should see two folders:

```text
PRE _layers/
PRE _manifests/
```

## Next steps

Now that you have your pull-through cache set up, you can configure your infrastructure to use it. Keep in mind that if you are not careful, you can create a circular dependency. Try not to do that.

### The docker daemon (dockerd)

The docker daemon is configured with a file at `/etc/docker/daemon.json`. Configure your pull-through cache with the `registry-mirrors` key:

```json
{
  "registry-mirrors": ["https://pt-dh.your-domain.example"]
}
```

Restart the docker daemon:

```text
sudo systemctl restart docker
```

And then you will be using your pull-through cache instead of the upstream docker hub.

### k3s

[k3s](https://k3s.io) lets you configure [mirrors for other Docker registries](https://docs.k3s.io/installation/private-registry). Copy this into `/etc/rancher/k3s/registries.yaml`:

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://pt-dh.your-domain.example"
```

Restart k3s:

```text
sudo systemctl restart k3s
```

Then any container images pulled from the Docker Hub will automatically go through your cache.
