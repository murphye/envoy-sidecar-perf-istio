# Demo 3 (Achieving 28K RPS in a Saturated Cluster)

The goal of this demo is not to compare/contrast performance of the application with and without the Envoy sidecars. That was done best in [Demo 2](DEMO_2.md). With this demo, we want to demonstrate scalability with an application running in a service mesh.

For this demo, we are increasing the replication factor of the services by 2.67 (from 3 to 8) to achieve a RPS increase of by a factor of 2.8 (from 10K to 28K), with an increase of latency by a factor of approximately 2.8 (from 10 ms to 28ms). In this scenario, the cluster is saturated with up to 70% CPU and memory utilization across all nodes. Because the cluster is busier, there is more of an impact on latency at this RPS at this scale. Given the new performance requirements, the increase latency is acceptable.

## Scale Up the Demo Application with a Replication of 8

Because this is a 9-node cluster, and 1 node is dedicated to running PostgreSQL, we can scale up to 8 pods per microservice. Because of the `podAntiAffinity` rules, each node will have 1 pod instance for each microservice.

```
kubectl scale deployments/customer --replicas=8
kubectl scale deployments/person --replicas=8
kubectl scale deployments/name --replicas=8
```
Check status with `kubectl get pods -owide` until all pods are `Running` on each node.

```

```