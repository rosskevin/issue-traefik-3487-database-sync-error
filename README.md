# Reproduction of traefik `Database sync error`

## Summary

This setup is:

- GKE (Kubernetes)
- Consul (3 replicas)
- Traefik (1 replica) + Let's encrypt ACME using `gcloud` dns + dashboard
- Hello world app
- Ingresses for hello and traefik https urls

Traefik is almost but not quite HA in this setup - I started reproducing the `Database sync error` noted in https://github.com/containous/traefik/issues/3487 so I stopped to reproduce.

## Prerequisites

1. Setup a GCP account
1. `gcloud auth login`
1. Setup a domain in gcloud dns with your domain configured
1. Edit `environments/foo/.env` and set the `DOMAIN` and `GOOGLE_CLOUD_PROJECT`
1. Setup a terminal window with `. ./use foo`
1. Create the cluster with

```bash
./scripts/cluster/create
./scripts/cluster/create-namespace
./scripts/cluster/switch
```

## Manually created cluster resources

### Consul secret

1. install consul locally on the PATH from https://www.consul.io/downloads.html
2. `consul keygen`
3. add `CONSUL_GOSSIP_KEY=xx` to `~/.gcp/${GOOGLE_CLOUD_PROJECT}/production.env`
4. run `./components/consul/scripts/secret`

### Traefik resources

1. run `./components/traefik/scripts/service-account`
1. run `./components/traefik/scripts/key`
1. run `./components/traefik/scripts/secret`
1. run `./components/traefik/scripts/secret-dashboard`
1. run `./components/traefik/scripts/reserve-ip`
1. Get reserved ip and replace `manifest.yml` `11.11.11.11` with your ip

## Apply the manifest

```
kubectl apply -f manifest.yml
```

## Post-setup Traefik notes

When bootstrapping, a number of things will happen and traefik will not be running properly.

1. `consul` must be up and running before job can complete
1. `traefik-storeconfig` job must complete before traefik can run successfully
1. `traefik` will start, but it won't be in a good state. Delete the traefik pod to let it restart
1. Try some urls including hello and traefik. This is the part where you are going to see:

```bash
time="2018-08-21T19:02:24Z" level=error msg="Datastore sync error: Object lock value: expected 7ff59879-849a-436b-bb9d-cfdc36658806, got 83e91690-cf4d-40c8-b3ff-6275da8b4ab8, retrying in 364.9962ms"
```

1. Try deleting the traefik pod - you will likely see the same error.
1. After an _extended_ period of time, you will stop getting a default cert error and get a _valid_ (we are using the staging server):

```
NET::ERR_CERT_AUTHORITY_INVALID
Subject: hello.foo.com

Issuer: Fake LE Intermediate X1
```

## TL;DR

We are getting `Datastore sync error` consistently when bootstrapping a new Traefik with a key value store. Deleting the pod will only increase the backoff in the log message from 300s to 500s (it seems).

## Structure

We use ksonnet, so it was easiest to setup this reproduction similarly. This reproduction does not use ksonnet, but the `manifest.yml` was generated locally from my real ksonnet resources for consul, traefik, and hello world.
