
## Naive Performance Testing with Unthrottled Load Tests

* Running unthrottled load tests against a microservices application can skew performance of a high throughput, low latency application
* For a non-sidecar/with-sidecar comparison:
* Replication of 1: p95 latency increased by 90%, RPS was cut by 44%
* Replication of 3: p95 latency increased by 38%, RPS was cut by 38%
* Replication of 1: CPU usage increased 15%, Memory increased 2%
* Replication of 3: CPU usage increased 18%, Memory decreased 5% (likely due to lower RPS)

### Takeaways

* Never a good idea to do unthrottled load tests to measure sidecar performance; numbers not realistic to real world use
* Unthrottled load tests do not trigger proper vertical CPU scaling for the pod, therefore degrading overall performance

## Optimized Performance Testing with Throttled Load Tests

* Throttled load tests should increase # of client connections while rate limiting each connection
* Running throttled load tests against a microservices application will demonstrate much better performance while also allowing vertical scaling of CPU to accomodate the sidecar
* For a non-sidecar/with-sidecar comparison:
* Replication of 3: p95 latency increased by 20%, RPS was cut by 3%
* Replication of 3: CPU usage increased 99%, Memory increased 2% (likely due to lower RPS)

### Analysis

1. For the p95 latency of 10 ms, the split between microservice/sidecar for a pod is 5ms/5ms given the lower bound of p95 latency for the microservice is 5 ms without the sidecar
2. Per #1, the vertical CPU scaling benefits both the microservice and the sidecar
3. Per #2, it can be inferred that the sidecar adds about 1 CPU per pod (1 CPU out of 2.9 CPU total per pod)
4. Per #3, this lines up with Istio docs stating the [Envoy proxy consumes 0.35 CPU per 1000 requests](https://istio.io/latest/docs/ops/deployment/performance-and-scalability/). In this case, each microservice is processing 3333 RPS.
5. Per #1, with 5ms additional p95 latency due to the sidecars, and each sidecar ingress or egress adds approx. 0.7 ms for the p95 latency
6. Per #5, **for a microservice that has both sidecar ingress and egress traffic, the sidecar adds approx 1.4 ms of p95 latency**
7. Per #6, the Istio docs state that the ["Envoy proxy added 2.65ms to the p90"](https://istio.io/latest/docs/ops/deployment/performance-and-scalability/), which is almost double my observed numbers

## Takeaways (Sidecar/Envoy Proxy)

1. For best performance of a pod with the sidecar, the goal should be to "spike" the CPU usage as much as possible for the microservice pod; this CPU spike will benefit both the microservice and the Envoy proxy
1. CPU spikes can only be triggered by proper load testing with throttled connections; unthrottled load tests do not result in CPU spikes and therefore poor performance
1. Expect to require an additional 1 CPU per sidecar to maintain high RPS and low latencies for 3333 RPS per pod
1. Performance of the Envoy proxy in the Istio sidecar can be twice as fast as [documented](https://istio.io/latest/docs/ops/deployment/performance-and-scalability/) (~1.4/2.7)

## Takeaways (Java Microservice with a Sidecar)

1. For applications requiring high throughput and low latency, target approx. 3333 RPS per pod with a sidecar
2. Per #1, this will give a predicable 1 CPU per Envoy proxy CPU overhead requires to maintain high RPS and low latencies 
3. Per #1, for a Netty-based Java microservice, target ~3 CPU per pod with a sidecar for maximum performance
4. Per #3, if less than ~3 CPU is consumed per pod, adjust # of client connections and throttling of load test
