# Setup Guide

## Kubernetes Cluster and Load Test VM Setup

### Kubernetes Cluster for Demo #1, #2, and #3

Setup environment:

```
gcloud config set project PROJECT_ID
gcloud config set compute/zone us-central1-a
gcloud config set compute/region us-central-1
```

For all 3 demos, you need to create a 9-node cluster with a machine type of `e2-highcpu-16`. This will create a 144 core cluster, so don't forget to shut down your cluster after creating it! For Demo #2, a 5-node cluster would suffice, but for Demo #3 a 9-node cluster is required. 

```
gcloud container clusters create load-test-envoy-sidecars-cluster --num-nodes=9 --machine-type=e2-highcpu-16
```

### Virtual Machine for Load Testing

```
gcloud compute instances create load-test-envoy-sidecars --zone=us-central1-a --image ubuntu-2104-hirsute-v20210720
```

## Installing Istio

Only a basic installion of Istio with the `default` profile that installs the `istiod` and `istio-ingressgateway`. All load testing is conducted through Istio Ingress from the VM where the load test client resides. No specific configuration or tuning of Istio is required to run the demos.

To install Istio, simply use the `istioctl` command line tool and run:

```
istioctl install
```

After installation is completed, you can then run:

```
istioctl verify-install
```

You can then find the `EXTERNAL-IP` for Istio Ingress by running:

```
kubectl get svc -n istio-system istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress[0].ip}"
```

This `EXTERNAL-IP` will be used for your load testing client.
