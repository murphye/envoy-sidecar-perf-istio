# Demo 1 (Degraded Performance with Naive Performance Testing)

The goal of this demo is to show how the Envoy sidecar can negatively impact performance in an extreme scenario where naive load testing is performed against the demo applications with each microservice deployed with a replication of 1. Deploying an application, especially for production, with a replication of 1 is not a best practice, but makes for an interesting test scenario.

Impact on the CPU and memory utilization of the microservice pods will also be assessed, and then reassessed in Demo 2.

After running this first Demo you may have the assumption that the Envoy sidecar is a killer on performance, and it may not be a good solution. Rest assured, this is an unrealistic scenario, and you will see a very different performance outcome in Demo 2.

## The Demo Application (Customer 360)

Customer 360 is a simplistic application to retrieve information about a customer, including their name. There are 3 simple microservices involved with a PostgreSQL database backing a Name service.

**TODO: Image**

The microservices are written in Spring Boot using the reactive WebFlux API. The Name service also uses the reactive R2JDBC database driver. The entire execution flow for the microservices is non-blocking and highly performant. WebFlux is a Netty-based library, and similar results could be achieve with Quarkus, Micronaut, and Vert.x, which also use Netty.

**Because the microservices use a JVM, after deployment, JVM warm up is required. When you run `hey` you should do it twice, and only count the 2nd result, as the 1st result will be impacted by the JVM warmup period.** That being said, Java is a very performant runtime for this type of microservice application.

## Deploy the Demo Application (Customer 360)

```
kubectl apply -f Demo-1/deploy-postgres.yaml
kubectl apply -f Demo-1/deploy-customer.yaml
kubectl apply -f Demo-1/deploy-person.yaml
kubectl apply -f Demo-1/deploy-name.yaml
```

Check the pods. You should see that they are all `Running` and each is deployed on a separate `NODE`.

```
kubectl get pods -owide
```

If the `name` pods have an error, there might have been a database connectivity issue when the pods was started. You can run this to correct the error:
```
kubectl rollout restart deployment name
```

Configure the Istio Ingress Gateway and VirtualService:
```
kubectl apply -f Demo-1/default-gateway.yaml
kubectl apply -f Demo-1/customer-virtualservice.yaml
```

You should now be able to `curl` the Istio Ingress endpoint to retrieve a list of customers. This list of customers is coming from the PostgreSQL database deployment.

```
curl http://$GATEWAY_IP/api/customer
```

**Note:** Even though the Customer 360 application deployment is not yet added to the service mesh (and running the Envoy sidecars), it's still accessable via the Istio Ingress Gateway.

## 1st Run: Naive Performance Test without Sidecars

For the 1st run on the VM with `hey`, we are going to use the default workers (50), and no rate limiting, for 1 million requests.

**Warning: The JVMs need to be warmed up. Conduct the 'hey` run twice, and discount the first result!**

```
hey -n 1000000 http://$GATEWAY_IP/api/customer
... Warm up!...
hey -n 1000000 http://$GATEWAY_IP/api/customer

Summary:
  Total:        134.2580 secs
  Slowest:      0.0352 secs
  Fastest:      0.0024 secs
  Average:      0.0063 secs
  Requests/sec: 7448.3475
  

Response time histogram:
  0.002 [1]             |
  0.006 [266961]        |■■■■■■■■■■■■■■■
  0.009 [715236]        |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.012 [16004]         |■
  0.015 [1573]          |
  0.019 [182]           |
  0.022 [3]             |
  0.025 [1]             |
  0.029 [7]             |
  0.032 [15]            |
  0.035 [17]            |

Latency distribution:
  10% in 0.0047 secs
  25% in 0.0056 secs
  50% in 0.0064 secs
  75% in 0.0071 secs
  90% in 0.0078 secs
  95% in 0.0082 secs
  99% in 0.0093 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0024 secs, 0.0352 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0000 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0016 secs
  resp wait:    0.0062 secs, 0.0023 secs, 0.0336 secs
  resp read:    0.0001 secs, 0.0000 secs, 0.0197 secs

Status code distribution:
  [200] 1000000 responses
```

7500 RPS with a p95 of 8 ms with only 1 pod deployed per microservice is a very respectable number out of the box, especially for 3 microservices in a chain of call, plus a database. There is opportunity for loss of performance at each service or database call. Obviously, Spring WebFlux/Netty is up to the task.


### Examine CPU and Memory Usage

```
kubectl top pods
NAME                        CPU(cores)   MEMORY(bytes)   
customer-55965c46fc-p2hhz   3011m        3081Mi          
name-5565bb7d5-95pbn        3087m        3066Mi          
person-7cff6f6945-fdkjc     2848m        3064Mi          
postgres-6f84fcc68b-h8v9j   751m         55Mi     
```

As you can see, for the 1st run, each microservice is consuming about 3 CPU cores, and 3 Gi of memory. This is in line with what we would expect from a Java microservice.

For all running pods, here are the totals:
* CPU: 9697m
* MEMORY: 9248Mi

## Add the Microservices and Database to the Service Mesh

Ok, now it's time to add the microservices to the mesh, and give them their Envoy sidecar. By default, Istio will enable mTLS in PERMISSIVE mode, so the HTTP calls between the microservices will be automatically secured. Traffic between Istio Ingress (which also uses Envoy) and the Customer service will also be secured with mTLS.

```
istioctl kube-inject -f Demo-1/deploy-postgres.yaml | kubectl apply -f -
istioctl kube-inject -f Demo-1/deploy-customer.yaml | kubectl apply -f -
istioctl kube-inject -f Demo-1/deploy-person.yaml | kubectl apply -f -
istioctl kube-inject -f Demo-1/deploy-name.yaml | kubectl apply -f -
```

Next, check to make sure the application is still available via Istio Ingress:

```
curl http://$GATEWAY_IP/api/customer
```

## 2nd Run: Naive Performance Test with Sidecars

Now, lets do the same load testing with the Envoy sidecars in place, as the application is now part of the service mesh.

```
hey -n 1000000 http://$GATEWAY_IP/api/customer
... Warm up!...
hey -n 1000000 http://$GATEWAY_IP/api/customer

Summary:
  Total:        238.8059 secs
  Slowest:      0.0967 secs
  Fastest:      0.0038 secs
  Average:      0.0105 secs
  Requests/sec: 4187.5014
  

Response time histogram:
  0.004 [1]             |
  0.013 [807515]        |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.022 [192232]        |■■■■■■■■■■
  0.032 [159]           |
  0.041 [37]            |
  0.050 [21]            |
  0.060 [3]             |
  0.069 [3]             |
  0.078 [4]             |
  0.087 [6]             |
  0.097 [19]            |


Latency distribution:
  10% in 0.0066 secs
  25% in 0.0081 secs
  50% in 0.0108 secs
  75% in 0.0126 secs
  90% in 0.0141 secs
  95% in 0.0152 secs
  99% in 0.0174 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0038 secs, 0.0967 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0000 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0017 secs
  resp wait:    0.0105 secs, 0.0037 secs, 0.0919 secs
  resp read:    0.0000 secs, 0.0000 secs, 0.0089 secs

Status code distribution:
  [200] 1000000 responses
```

### Examine CPU and Memory Usage

```
kubectl top pods
NAME                        CPU(cores)   MEMORY(bytes)   
customer-85b786649-5vz5m    3579m        3108Mi          
name-6656775848-s5rbc       3362m        3112Mi          
person-6b6b455b6-xc7bf      3178m        3107Mi          
postgres-5d7bf4dc95-jffrj   1030m        95Mi            
```

For all running pods, here are the totals when running with the Envoy sidecar:
* CPU: 11149m
* MEMORY: 9422Mi

## Performance Impact of the Sidecars on Latency and Throughput

**The p95 value has increased from 8 ms to 15ms, a very large 90% increase.** That being said, given that there are 4 sidecars in place, this implies about 1.75 ms of latency added by the sidecars.

**The RPS has dropped from 7448 to 4187, a very large 44% decrease.** This drop may give cause for concern, especially of there are equivalent hits on RPS with larger scale deployments.

So, with this first round of performance testing, the sidecars definitely have an impact, and depending on your perspective, the performance impact may be reasonable, or it may be terrible. It may also be cause for concern on how throughput and latency would be affected at scale with a service mesh with many replicas running.

## Sidecar's Impact on CPU and Memory

Now, let's take a look at the impact on CPU and memory, in aggregate, created by the sidecars.

Total Before Sidecar:
* CPU: 9697m
* MEMORY: 9248Mi

Total After Sidecar:
* CPU: 11149m
* MEMORY: 9422Mi

Change in % After Sidecar:
* CPU: +15%
* MEMORY: +2%

The Envoy sidecar adds an average of 363m more CPU requirements per pod, so if you have many pods deployed, the potential CPU overhead of the sidecar should be taken into account based on these numbers.

## Takeaways

In this scenario, the Envoy sidecars take a major toll on performance. They basically double the p95 latency, halve the RPS, and add a 15% CPU overhead. 

If you're goal is to have high throughput, low latency microservices, you might already be ready to walk away from the Envoy sidecar. Yet, there is still [Demo 2](DEMO_2.md) to look at how the Envoy sidecar performs at a higher scale and with more concrete performance requirements.