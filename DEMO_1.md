# Demo 1 (Degraded Performance with Naive Performance Testing)

The goal of this demo is to show how the Envoy sidecar can negatively impact performance in an extreme scenario where naive load testing is performed against the demo applications with each microservice deployed with a replication of 1.

Impact on the CPU and memory utilization of the microservice pods will also be assessed, and then reassessed in Demo 2.

After running this Demo you may have the assumption that the Envoy sidecar is a killer on performance, and it may not be a good solution. Rest assured, this is the worst-case scenario, and you will see a very different performance outcome in Demo 2.

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

Configure the Istio Ingress Gateway and VirtualService:
```
kubectl apply -f Demo-1/default-gateway.yaml
kubectl apply -f Demo-1/customer-virtualservice.yaml
```

Check the pods. If the `name` pods have an error, there might have been a database connectivity issue when the pods was started. You can run this to correct the error:
```
kubectl rollout restart deployment name
```

You should now be able to `curl` the Istio Ingress endpoint to retrieve a list of customers. This list of customers is coming from the PostgreSQL database deployment.

```
curl http://EXTERNAL-IP/api/customer
```

**Note:** Even though the Customer 360 application deployment is not yet added to the service mesh (and running the Envoy sidecars), it's still accessable via the Istio Ingress Gateway.

## 1st Run: Naive Performance Test without Sidecars

For the 1st run with `hey`, we are going to use the default workers (50), and no rate limiting, over 4 minutes.

**Warning: Conduct the 'hey` run twice, and discount the first result!**

```
hey -z 240s http://EXTERNAL-IP/api/customer
Summary:
  Total:        240.0053 secs
  Slowest:      0.0617 secs
  Fastest:      0.0034 secs
  Average:      0.0120 secs
  Requests/sec: 6660.1817
 
Response time histogram:
  0.003 [1]     |
  0.009 [893800]        |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.015 [104904]        |■■■■■
  0.021 [1259]  |
  0.027 [23]    |
  0.033 [1]     |
  0.038 [0]     |
  0.044 [0]     |
  0.050 [0]     |
  0.056 [0]     |
  0.062 [12]    |
Latency distribution:
  10% in 0.0060 secs
  25% in 0.0066 secs
  50% in 0.0074 secs
  75% in 0.0083 secs
  90% in 0.0093 secs
  95% in 0.0101 secs
  99% in 0.0120 secs
Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0034 secs, 0.0617 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0000 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0025 secs
  resp wait:    0.0115 secs, 0.0034 secs, 0.0617 secs
  resp read:    0.0004 secs, 0.0000 secs, 0.0084 secs
Status code distribution:
  [200] 1000000 responses
```

6660 RPS with a p95 of 10 ms with only 1 pod deployed per microservice is a very respectable number out of the box, especially for 3 microservices in a chain of call, plus a database. There is opportunity for loss of performance at each service or database call. Obviously, Spring WebFlux/Netty is up to the task.

TODO `kubectl top nodes` `kubectl top pods`


## Add the Microservices and Database to the Service Mesh

Ok, now it's time to add the microservices to the mesh, and give them their Envoy sidecar. By default, Istio will enable mTLS in PERMISSIVE mode, so the HTTP calls between the microservices will be automatically secured. Traffic between Istio Ingress (which also uses Envoy) and the Customer service will also be secured with mTLS.

```
TODO
```

Next, check to make sure the application is still available via Istio Ingress:

```
curl http://EXTERNAL-IP/api/customer
```

## 2nd Run: Naive Performance Test with Sidecars

Now, lets do the same load testing with the Envoy sidecars in place, as the application is now part of the service mesh.

```
hey -z 240s http://34.122.170.225/api/customer
Summary:
  Total:        240.0154 secs
  Slowest:      0.1002 secs
  Fastest:      0.0046 secs
  Average:      0.0120 secs
  Requests/sec: 4651.4016
 
Response time histogram:
  0.005 [1]     |
  0.014 [927360]        |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.024 [67248] |■■■
  0.033 [2826]  |
  0.043 [1564]  |
  0.052 [656]   |
  0.062 [240]   |
  0.071 [71]    |
  0.081 [30]    |
  0.091 [2]     |
  0.100 [2]     |
Latency distribution:
  10% in 0.0080 secs
  25% in 0.0090 secs
  50% in 0.0103 secs
  75% in 0.0119 secs
  90% in 0.0136 secs
  95% in 0.0148 secs
  99% in 0.0186 secs
Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0046 secs, 0.1002 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0000 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0077 secs
  resp wait:    0.0119 secs, 0.0045 secs, 0.1001 secs
  resp read:    0.0001 secs, 0.0000 secs, 0.0169 secs
Status code distribution:
  [200] 1000000 responses
```

The p95 value has increased from 10 ms to 15ms, a massive 50% increase. That being said, given that there are 4 sidecars in place, with ingress and egress between service calls and database lookups, so the increase in latency seems understandable, and is still under 1ms per sidecar in this scenario.

The RPS has dropped from 6660 to 4651, a large 30% decrease. This drop may give cause for concern, especially of there are equivalent hits on RPS with larger scale deployments.

So, with this first round of performance testing, the sidecars definitely have an impact, and depending on your perspective, the performance impact may be reasonable, or it may be terrible.

## Sidecar's Impact on CPU and Memory for the Microservice Pods

