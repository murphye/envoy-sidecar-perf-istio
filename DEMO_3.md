# Demo 3 (Achieving 28K RPS in a Saturated Cluster)

If you initially created a 5 node GKE cluster, rather than 9, you will not be able to continue with Demo 3. You can increase your node pool size for the GKE cluster to be 9 should you wish to continue.


## Scale Up the Demo Application with a Replication of 8

Because this is a 9-node cluster, and 1 node is dedicated to running PostgreSQL, we can scale up to 8 pods per microservice. Because of the `podAffinity` and `podAntiAffinity` rules, each node will have 1 pod instance for each microservice. By having all of the pods co-located, it will further improve performance at scale and balance out usage of CPU and memory across the nodes.

```
kubectl scale deployments/customer --replicas=8
kubectl scale deployments/person --replicas=8
kubectl scale deployments/name --replicas=8
```
Check status with `kubectl get pods -owide` until all pods are `Running` on each node.