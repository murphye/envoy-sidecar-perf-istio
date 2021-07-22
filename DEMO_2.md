# Demo #2 (Achieving Low Latency at 10K RPS)

As shown in [Demo #1](DEMO_1.md), there is a cost to the Envoy sidecars, and in certain scenarios, the cost can be very steep. Yet, running a simple application, with each microservice running as a single replication, with no well-defined performance requirements, may result in a worst-case scenario set of results.

Here for Demo 2, we are going to take a step back and reexamine how the application is deployed, and how the performance requirements impact performance testing. With a fresh approach, the numbers may come into better alignment with expectations.

## Performance Requirements

1. At least 10,000 requests per second (RPS)
1. At most 10 milliseconds (ms) end-to-end latency using a client within the same datacenter/zone as the cluster
1. No HTTP errors for 1,000,000 requests (after proper warm up run)

## Scaling Out the Application

For an application that requires high throughput and low latency, using `podAffinity` and `podAntiAffinity` rules can help you maximize performance. You can carefully place pods onto nodes to balance the load equally, and potentially reduce node-to-node communication by co-locating pods as well.

As such, we can now deploy updated deployment configuration, with `podAffinity` and `podAntiAffinity` rules already set.

```
kubectl scale deployments/postgres --replicas=0
kubectl scale deployments/customer --replicas=0
kubectl scale deployments/person --replicas=0
kubectl scale deployments/name --replicas=0

kubectl apply -f Demo-1/deploy-postgres.yaml
kubectl apply -f Demo-2/deploy-customer.yaml
kubectl apply -f Demo-2/deploy-person.yaml
kubectl apply -f Demo-2/deploy-name.yaml
```

Check the pods. You should see that they are all `Running` and each microservice is co-located with a corresponding microservice on the `NODE`.

```
kubectl get pods -owide
```

## 1st Run: Unthrottled Load Test without Envoy Sidecar

```
hey -n 1000000 http://$GATEWAY_IP/api/customer
... Warm up!...
hey -n 1000000 http://$GATEWAY_IP/api/customer
... Warm up!...
hey -n 1000000 http://$GATEWAY_IP/api/customer

Summary:
  Total:        81.8178 secs
  Slowest:      0.0207 secs
  Fastest:      0.0023 secs
  Average:      0.0039 secs
  Requests/sec: 12222.2837

Latency distribution:
  10% in 0.0033 secs
  25% in 0.0035 secs
  50% in 0.0038 secs
  75% in 0.0042 secs
  90% in 0.0047 secs
  95% in 0.0050 secs
  99% in 0.0059 secs

Status code distribution:
  [200] 1000000 responses
```

Compared to the 1st run in Demo 1, RPS has increased from 7448 to 12222.

### Examine CPU and Memory Usage

```
kubectl top pods
NAME                        CPU(cores)   MEMORY(bytes)   
customer-6468d675d7-2csmw   2070m        3073Mi          
customer-6468d675d7-9zrz7   2003m        3084Mi          
customer-6468d675d7-smgnt   2072m        3079Mi          
name-757d8fcb9f-5z5gv       2178m        3059Mi          
name-757d8fcb9f-m7kgw       1505m        3056Mi          
name-757d8fcb9f-tvqgm       2573m        3066Mi          
person-5ccb469b7f-fsjkm     1699m        3053Mi          
person-5ccb469b7f-mhkvf     1819m        3074Mi          
person-5ccb469b7f-pr7xq     1751m        3076Mi          
postgres-6f84fcc68b-697b2   1141m        82Mi     
```

CPU Total: 18811m
Mem Total: 27702Mi

## 2nd Run: Throttled Load Test without Envoy Sidecar

```
hey -c 82 -q 130 -n 1000000 http://$GATEWAY_IP/api/customer

Summary:
  Total:        93.9136 secs
  Slowest:      0.0596 secs
  Fastest:      0.0026 secs
  Average:      0.0051 secs
  Requests/sec: 10647.9830

Latency distribution:
  10% in 0.0039 secs
  25% in 0.0043 secs
  50% in 0.0049 secs
  75% in 0.0056 secs
  90% in 0.0065 secs
  95% in 0.0080 secs
  99% in 0.0092 secs

Status code distribution:
  [200] 999990 responses
```

```
kubectl top pods
NAME                        CPU(cores)   MEMORY(bytes)   
customer-6468d675d7-2csmw   1659m        3079Mi          
customer-6468d675d7-9zrz7   1542m        3090Mi          
customer-6468d675d7-smgnt   1664m        3088Mi          
name-757d8fcb9f-5z5gv       1563m        3055Mi          
name-757d8fcb9f-m7kgw       1203m        3054Mi          
name-757d8fcb9f-tvqgm       1665m        3062Mi          
person-5ccb469b7f-fsjkm     1525m        3053Mi          
person-5ccb469b7f-mhkvf     1341m        3065Mi          
person-5ccb469b7f-pr7xq     1262m        3080Mi          
postgres-6f84fcc68b-697b2   996m         83Mi           
```

CPU Total: 14420m
Mem Total: 27626Mi

## Add the Microservices and Database to the Service Mesh

Ok, now it's time to add the microservices to the mesh (again), and give them their Envoy sidecar (again).

```
istioctl kube-inject -f Demo-1/deploy-postgres.yaml | kubectl apply -f -
istioctl kube-inject -f Demo-2/deploy-customer.yaml | kubectl apply -f -
istioctl kube-inject -f Demo-2/deploy-person.yaml | kubectl apply -f -
istioctl kube-inject -f Demo-2/deploy-name.yaml | kubectl apply -f -
```

Next, check to make sure the application is still available via Istio Ingress:

```
curl http://$GATEWAY_IP/api/customer
```

## 3rd Run: Unthrottled Load Test with Envoy Sidecar

```
hey -n 1000000 http://$GATEWAY_IP/api/customer
... Warm up!...
hey -n 1000000 http://$GATEWAY_IP/api/customer
... Warm up!...
hey -n 1000000 http://$GATEWAY_IP/api/customer

Summary:
  Total:        132.7737 secs
  Slowest:      0.0241 secs
  Fastest:      0.0040 secs
  Average:      0.0065 secs
  Requests/sec: 7531.6106

Latency distribution:
  10% in 0.0054 secs
  25% in 0.0058 secs
  50% in 0.0063 secs
  75% in 0.0070 secs
  90% in 0.0077 secs
  95% in 0.0082 secs
  99% in 0.0094 secs

Status code distribution:
  [200] 1000000 responses
```

### Examine CPU and Memory Usage

```
kubectl top pods
NAME                        CPU(cores)   MEMORY(bytes)   
customer-678d56c89f-5l7b7   2440m        3141Mi          
customer-678d56c89f-nrg24   2777m        2537Mi          
customer-678d56c89f-tfvlr   2729m        3128Mi          
name-768959d54d-4nsl2       2205m        2513Mi          
name-768959d54d-f8ksc       2509m        3096Mi          
name-768959d54d-zd825       2129m        3100Mi          
person-7cf74f846b-5jv8b     1893m        3012Mi          
person-7cf74f846b-qtsxk     2289m        3092Mi          
person-7cf74f846b-zk555     2378m        2648Mi          
postgres-5d7bf4dc95-8mncc   1649m        126Mi   
```

CPU Total: 22998m
Mem Total: 26393Mi

## 4th Run: Throttled Load Test with Envoy Sidecar

```
hey -c 82 -q 130 -n 1000000 http://$GATEWAY_IP/api/customer

Summary:
  Total:        96.5790 secs
  Slowest:      0.0247 secs
  Fastest:      0.0041 secs
  Average:      0.0078 secs
  Requests/sec: 10354.1091

Latency distribution:
  10% in 0.0062 secs
  25% in 0.0068 secs
  50% in 0.0076 secs
  75% in 0.0085 secs
  90% in 0.0096 secs
  95% in 0.0104 secs
  99% in 0.0121 secs

Status code distribution:
  [200] 999990 responses
```

### Examine CPU and Memory Usage

```
kubectl top pods
NAME                        CPU(cores)   MEMORY(bytes)   
customer-678d56c89f-5l7b7   3112m        3147Mi          
customer-678d56c89f-nrg24   3128m        3133Mi          
customer-678d56c89f-tfvlr   3176m        3154Mi          
name-768959d54d-4nsl2       2848m        3110Mi          
name-768959d54d-f8ksc       3144m        3109Mi          
name-768959d54d-zd825       2733m        3107Mi          
person-7cf74f846b-5jv8b     2512m        3125Mi          
person-7cf74f846b-qtsxk     2820m        3143Mi          
person-7cf74f846b-zk555     3136m        3126Mi          
postgres-5d7bf4dc95-8mncc   2166m        126Mi         
```

CPU Total: 28775m
Mem Total: 28280Mi

## Performance Summary

### Unthrottled Load Test

#### No Sidecar 
* RPS: 12222
* p95: 5 ms
* CPU Total: 18811m
* Mem Total: 27702Mi

#### With Sidecar
* RPS: 7532 (-4690)
* p95: 8 ms (+3)
* CPU Total: 22998m (+4187)
* Mem Total: 26393Mi (-1309; lower memory utlization is likely a side effect of much lower RPS and not directly due to the sidecar itself)

### Throttled Load Test

The throttling of the load test was intended to keep the RPS about the same. This results in a more apples-to-apples comparison as opposed to an unthrottled load test that doesn't offer baseline to compare against. We can then identify what the "cost" is to maintain RPS and latency when adding the Envoy sidecars.

#### No Sidecar 
* RPS: 10647
* p95: 8 ms
* CPU Total: 14420m
* Mem Total: 27626Mi

#### With Sidecar
* RPS: 10354 (-293)
* p95: 10 ms (+2)
* CPU Total: 28775m (+14355)
* Mem Total: 28280Mi (+654)

### Performance Takeaway

The big takeaway here is the CPU total for the throttled load test. For 3 microservices (and a replication of 3 each for 9 pods total), and 1 database pod, all runnning with sidecars, double the CPU is required to meet the performance requirements as opposed to without the sidecar. 

With the Envoy sidecars, 28.3 CPU cores are required to meet the performance requirements, but without the Envoy sidecars, 14.5 CPU cores are required. Given that there are 10 pods in total, each Envoy sidecar needs appromimately 1.4 CPU to maintain this level of performance.

In this scenario, the Envoy sidecars only added 2 ms to the total p95 latency. Given that there are 3 service calls, and a database query for each request, that equates to 0.3 ms added for each Envoy ingress or egress from the sidecar containers. However, the calculation isn't that simple, as some of the additional CPU cycles consumed may have further improved the microservice application performance therfore helping to balance out the additonal latency added by the Envoy sidecar. Collecting addtional metrics showing how much CPU Envoy is consuming vs. the microservice would determine exactly how much latency the sidecars are adding.

The key to maintaining performance when employing the Envoy sidecars is maximizing CPU utilization with proper deployment scaling, tuning, and testing. **Using more CPU  is not a bad thing**, as the benefits offered by Envoy sidecars and a service mesh are often worth the extra CPU required while maintaining performance.