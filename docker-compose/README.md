# Making a pull-through cache for the docker hub using Docker Compose

Head to [storage.new](https://storage.new) and make a new bucket. You can name it whatever you want, but this tutorial will call it `pullthru-docker-hub`.

Click on the Access Keys tab in the sidebar and then click on Create New Access Key. Give it a descriptive name and grant it editor access to the bucket you just created.

## Configuring the registry

Copy the `.env.example` file to `.env`. Fill in the following fields:

- `REGISTRY_STORAGE_S3_BUCKET`: Set it to your bucket name, such as `pullthru-docker-hub`.
- `REGISTRY_STORAGE_S3_ACCESSKEY`: Set it to the Tigris keypair key ID starting with `tid_`.
- `REGISTRY_STORAGE_S3_SECRETKEY`: Set it to the Tigris keypair secret key starting with `tsec_`.
- `REGISTRY_PROXY_USERNAME`: Set it to the username of a user on the Docker Hub
- `REGISTRY_PROXY_PASSWORD`: Set it to the password of that user on the Docker Hub

Once you've gotten that all configured, start the Docker registry with `docker compose`:

```text
docker compose up -d
```

This will start the registry listening on port 5000 of your machine.

## Testing

Now that the registry is up, you can test it using the `docker run` command:

```text
docker run --rm -it localhost:5000/library/hello-world
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

Now that you have your pull-through cache set up, there's a few things you can do next to productionalize this. You should set this up with a HTTPS certificate using a service like nginx or traefik so that your infrastructure can reliably reach this cache.

Keep in mind that if you are not careful, you can create a circular dependency. Try not to do that.

When you've done that, you can configure your system to use this pull-through cache by default instead of using the docker hub. The exact configuration will vary by program, but here's some examples to get you started.

### The docker daemon (dockerd)

The docker daemon is configured with a file at `/etc/docker/daemon.json`. Configure your pull-through cache with the `registry-mirrors` key:

```json
{
  "registry-mirrors": ["https://pt-dh.your-domain.example"]
}
```

If you are running your pull-through cache over plain HTTP, you can use this instead:

```json
{
  "registry-mirrors": ["http://localhost:5000"]
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

If you are running your pull-through cache over plain HTTP, you can use this instead:

```yaml
mirrors:
  docker.io:
    endpoint:
      - "http://localhost:5000"
```

Restart k3s:

```text
sudo systemctl restart k3s
```

Then any container images pulled from the Docker Hub will automatically go through your cache.
