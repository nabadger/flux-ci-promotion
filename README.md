# Overview

Testing image promotion from one environment to another using flux.

Uses: https://github.com/kubernetes-sigs/kind for local cluster development.

The goal is to see how we can promote releases from dev -> staging -> prod using flux and kustomize.


# Usage

## Create `kind` clusters for each environment

```
kind create cluster --name dev
kind create cluster --name staging
kind create cluster --name prod
```

```
export KUBECONFIG=$(kind get kubeconfig-path --name dev)
export KUBECONFIG=$(kind get kubeconfig-path --name staging)
export KUBECONFIG=$(kind get kubeconfig-path --name prod)
```

## Create and tag echoserver

Create some example tags and upload them to dockerhub (or whereever).

```
docker pull k8s.gcr.io/echoserver:1.4
docker tag k8s.gcr.io/echoserver:1.4 nabadger/echoserver:dev-test-1
docker tag k8s.gcr.io/echoserver:1.4 nabadger/echoserver:dev-test-2
docker tag k8s.gcr.io/echoserver:1.4 nabadger/echoserver:dev-test-3
docker push nabadger/echoserver:dev-test-1
docker push nabadger/echoserver:dev-test-2
docker push nabadger/echoserver:dev-test-3
```

**NOTE**: These have already been pushed to dockerhub, so can skip this step, unless you want add more tags.

## Deploying Flux

Deploy each flux into its own cluster using the same environment. 

Dev Cluster:

```
export KUBECONFIG=$(kind get kubeconfig-path --name dev)
kubectl apply -f ./bootstrap
kubectl apply -f ./flux/
kubectl apply -f ./flux/clusters/dev.yaml
```

Staging Cluster:

```
export KUBECONFIG=$(kind get kubeconfig-path --name staging)
kubectl apply -f ./bootstrap
kubectl apply -f ./flux/
kubectl apply -f ./flux/clusters/staging.yaml
```
## Confirm cluster workloads

```
NAME                                 READY     STATUS    RESTARTS   AGE
flux-59d479788d-7dhnr                1/1       Running   0          7m
memcached-76964fb66f-ms6bb           1/1       Running   0          7m
team-a-echoserver-6dcd6cb759-8d42q   1/1       Running   0          7s
team-b-echoserver-68d8bbf5d-gmhtx    1/1       Running   0          7s
```

## Setup Git Deploy Key

```
fluxctl identity
```

Add public-key from above command to the Git repo Deploy Key. Requires `write` access.

## Monitor workloads 

```
fluxctl list-workloads
```

```
WORKLOAD                              CONTAINER     IMAGE                                                  RELEASE  POLICY
default:deployment/flux               flux          docker.io/2opremio/flux:generators-releasers-8baf8bd0  ready    
default:deployment/memcached          memcached     memcached:1.4.25                                       ready    
default:deployment/team-a-echoserver  my-container  nabadger/echoserver:dev-test-1                         ready    
default:deployment/team-b-echoserver  my-container  nabadger/echoserver:dev-test-1                         ready    
```

## Release a new image

```
fluxctl release -n default --workload=default:deployment/team-a-echoserver --update-image=nabadger/echoserver:dev-test-2
```
