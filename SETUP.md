# Setup Guide

## Kubernetes Cluster and Load Test VM Setup

### Kubernetes Cluster for Demo #1, #2, and #3

Setup environment:

```
gcloud config set project PROJECT_ID
gcloud config set compute/zone us-central1-a
gcloud config set compute/region us-central-1
```

To run all three demos, you need to create a 9-node cluster with a machine type of `e2-highcpu-16`. This will create a 144 core cluster, so don't forget to shut down your cluster after creating it! For Demo #2, a 5-node cluster would suffice, but for Demo #3 a 9-node cluster is required.

To minimize latency, the high-CPU, 16 core machine type was used because CPU is more important than memory for running the load tests. As such, each machine only has 16 GB of memory.

```
gcloud container clusters create load-test-envoy-sidecars-cluster --num-nodes=9 --machine-type=e2-highcpu-16
```

After the cluster is running, you can then obtain the credentials so you can connect to the cluster with `kubectl`.

```
gcloud container clusters get-credentials load-test-envoy-sidecars-cluster
kubectl get nodes
```

### Virtual Machine for Load Testing

To minimize latency from a client to the Kubernetes cluster where the demo application is running, running a virtual machine in the same zone as the GKE cluster is recommended. This will give realistic numbers for a internal client that is running inside your network.

```
gcloud compute instances create load-test-envoy-sidecars --zone=us-central1-a --image ubuntu-2104-hirsute-v20210720
```

After the VM is running, you can connect to it. You will want to open a separate terminal window/tab, as you will want to connect with `kubectl` from your local machine in a different terminal.

```
gcloud beta compute ssh load-test-envoy-sidecars
```

## Installing `hey` onto the Virtual Machine

Hey is a small load testing tool that is simple to use and generates an easy to understand report after each run. There is no configuration required to run `hey` as opposed to other load test tools like `k6`. That being said, feel free to use your favorite load testing tool for this exercise provided that you are able to understand how these settings translate.

* Virtual Users: In `hey` there are workers that open 2 connections each. So 50 workers == 100 connections == 100 virtual users. The default is 50 workers in `hey`.
* Rate Limiting: In `hey` the queries per second is on a per-worker basis. By default, there is no limit.
* Duration or # of Requests: You can use either, but it's recommended to achieve at least 1 million requests per run.

Download `hey`:

```
curl https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64 --output hey
```

Run `hey`:
```
chmod 775 hey
hey
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
