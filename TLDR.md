# TL;DR

## Summary TL;DR

### Load Testing Findings
* Unthrottled load test will yield skewed performance benchmarks that are not comparable
* Throttled load tests against pre-defined performance requirements yield performance benchmarks that are comparable
* Throttled load tests with more client connections trigger proper vertical CPU scaling for pods

### Sidecar Proxy Performance Findings
* At 3333 RPS, 1 CPU is consumed by each sidecar Envoy proxy
* 1.4 ms p95 latency is introduced for 1 Envoy proxy egress + 1 Envoy proxy ingress
* 1.5 CPU baseline (microservice) + 0.5 CPU extra (microservice) can negate 40% of latency introduced by the sidecar proxy (about 0.6 ms in latency reduction)
* Ideal CPU usage is 1.5 CPU baseline (microservice) + 0.5 CPU extra (microservice) + 1.0 CPU sidecar Envoy proxy = 3.0 CPU per microservice pod with a sidecar

## Naive Performance Testing with Unthrottled Load Tests

* Running unthrottled load tests against a microservices application can skew performance of a high throughput, low latency application
* For a non-sidecar/with-sidecar comparison:
* Replication of 1: p95 latency increased by 90%, RPS was cut by 44%
* Replication of 3: p95 latency increased by 38%, RPS was cut by 38%
* Replication of 1: CPU usage increased 15%, Memory increased 2%
* Replication of 3: CPU usage increased 18%, Memory decreased 5% (likely due to lower RPS)

### Naive Performance Testing Takeaways

* Never a good idea to do unthrottled load tests to measure sidecar performance; numbers not realistic to real world use
* Unthrottled load tests do not trigger proper vertical CPU scaling for the pod, therefore degrading overall performance

## Optimized Performance Testing with Throttled Load Tests (10K RPS, p95 10 ms)

* Throttled load tests should increase # of client connections while rate limiting each connection
* Running throttled load tests against a microservices application will demonstrate much better performance while also allowing vertical scaling of CPU to accomodate the sidecar
* For a non-sidecar/with-sidecar comparison:
* Replication of 3: p95 latency increased by 20%, RPS was cut by 3%
* Replication of 3: CPU usage increased 99% (this is a good thing), Memory increased 2%

### Analysis (10K RPS, p95 10 ms)

1. For the p95 latency of 10 ms, the split between microservice/sidecar for a pod is approximately 5ms/5ms given the lower bound of p95 latency for the microservice is 5 ms without the sidecar
2. Per #1, the vertical CPU scaling benefits both the microservice and the sidecar
3. It can be inferred that the sidecar adds about 1 CPU per pod (1 CPU out of 2.9 CPU total per pod) by comparing non-sidecar/with-sidecar CPU performance 
   *(28775m total CPU with sidecar - 18811m total CPU without sidecar = 9964m total for sidecar / 10 pods = 996m per sidecar per pod)*
4. Per #3, this lines up with Istio docs stating the [Envoy proxy consumes 0.35 CPU per 1000 requests](https://istio.io/latest/docs/ops/deployment/performance-and-scalability/); in this case, each microservice is processing 3333 RPS *(5% improvement over Istio docs)*
5. Per #1, with 5ms additional p95 latency due to the sidecars, and each sidecar ingress or egress adds approx. 0.7 ms for the p95 latency *(5 ms / 7 ingresses and egresses)*
6. Per #5, for an equal balance of 3 sidecar ingress, and 3 sidecar egress (ignoring the database sidecar ingress), ~4.3 ms of p95 latency is introduced
7. Per #6, **1 sidecar egress with 1 sidecar ingress adds ~1.4 ms of p95 latency** *(4.3/3)*
8. Istio docs state that [with two proxies, one handling sidecar ingress, one handling sidecar ingress add 1.7 ms of p90 latency with jitter enabled](https://istio.io/latest/docs/ops/deployment/performance-and-scalability/)
9. Per #8, the Istio benchmarks roughly align ~1.4 p95 vs. ~1.7 p90 addional latency introduce by the sidecar (accounting for both ingress/egress across two Envoy proxies)

## Takeaways (With Sidecar Proxy @ 10K RPS, p95 10 ms)

1. For best performance of a pod with the sidecar, the goal should be to "spike" the CPU usage as much as possible for the microservice pod; this CPU spike will benefit both the microservice and the Envoy proxy
1. CPU spikes can only be triggered by proper load testing with throttled connections; unthrottled load tests do not result in CPU spikes and therefore poor performance
1. Expect to require an additional 1 CPU per sidecar to maintain high RPS and low latencies for 3333 RPS per pod
1. The Envoy proxy in the Istio sidecar can be ~20% faster than [documented](https://istio.io/latest/docs/ops/deployment/performance-and-scalability/) (~1.4/1.7) when using a high-performing CPU cluster

## Takeaways (Java Microservice with a Sidecar Proxy @ 10K RPS, p95 10 ms)

1. For applications requiring high throughput and low latency, target approx. 3333 RPS per pod with a sidecar
2. Per #1, this will give a predictable 1 CPU per Envoy proxy CPU overhead requires to maintain high RPS and low latencies 
3. Per #1, for a Netty-based Java microservice, target ~3 CPU per pod with a sidecar for maximum performance
4. Per #3, if less than ~3 CPU is consumed per pod, adjust # of client connections and throttling of load test
5. Apply proper `podAntiAffinity` rules to balance load across cluster nodes; pod placement is critical for maximizing performance
6. Run tests multiple times to make sure the Java Virtual Machine is warmed up appropriately
7. Use a Java framework that is reactive, non-blocking, and based on Netty for the best performance
