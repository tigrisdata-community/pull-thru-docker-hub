# Setting up your own pull-through Docker Hub cache with Tigris

This repo contains a few examples for setting up your own pull-through docker hub cache on top of [Tigris](https://tigrisdata.com). They contain production-worthy example configuration as well as full instructions on how to set this up.

To get started, you will need the following:

- An account on Tigris: [https://storage.new](https://storage.new).
- A server or Kubernetes cluster to run this image on.
- Optionally, a domain name pointed to that server or Kubernetes cluster for TLS termination.

Here's the tutorials, broken down by the stack:

- [Docker Compose](./docker-compose/)
- [Kubernetes](./k8s/)
