# Submillisecond Istio Sidecars

This repository demonstrates how to achieve high throughput and low latencies with the Istio sidecar. With this demo, you deploy 3 Java microservices and a PostgreSQL database in Kubernetes (GKE). You can then run a series of load tests against the demo application both with and without Istio sidecars enabled.

At scale, you will see the delta of latency for sidecar ingress/egress measure to be less than 1 millisecond. This enables the demo application to have an end-to-end p95 latency of only 10 ms, at 10,000 RPS, while passing through 7 Envoy ingress or egress proxies across 3 microservices and a database.

Optionally, you may scale this demo up to nearly 30,000 RPS with an end-to-end p95 latency of ~25 ms. This demonstrates a more constrained cluster that is consuming a higher level of CPU and memory resources, but is still able to maintain acceptable performance.

Ultimately, this demo shows that with best practices for deploying applications at scale on Kubernetes, Istio/Envoy sidecars are up to the task for low latency and high throughput applications.
