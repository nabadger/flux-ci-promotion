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

## Setup Git Deploy Key

```
fluxctl identity
```

Add public-key from above command to the Git repo Deploy Key. Requires `write` access.

## Monitor workloads 

```
fluxctl list-images --workload=default:deployment/dev-echo
```

## Release a new image

```
fluxctl release --workload=default:deployment/dev-echo --update-image=nabadger/echoserver:dev-test-1
```
