This repo contains demo for my talk about service meshes. It is based on [microservices-demo application](https://github.com/microservices-demo/microservices-demo) with some minor modifications to make it play nicely with istio. 

This demo is deployed and tested with `kubernetes 1.16` and `istio 1.5`

### 0. Install istio
1. Refer to [istio docs](https://istio.io/docs/setup/install/) for different methods on how install istio. Istio will be installed in a deferent namespace called `istio-system`
2. Create a namespace for our application and add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies during deployment of sock-shop app. 
```bash
$ kubectl apply -f 1-deploy-app/manifests/sock-shop-ns.yaml 
$ kubectl label namespace sock-shop istio-injection=enabled
```

### 1. Deploy application
No changes we're made to the original k8s manifests from [microservices-demo application](https://github.com/microservices-demo/microservices-demo) except:

+ updating `Deployment` resources to use the sable api `apps/v1` required since [k8s 1.16](https://kubernetes.io/blog/2019/09/18/kubernetes-1-16-release-announcement/)
+ added `version: v1` label to all Kubernetes deployments. We need it for Istio `Destination Rules` to work properly

1. deploy the application
```bash
$ kubectl apply -f 1-deploy-app/manifests
```

2. Configure Istio virtual services & Distination rules
```bash
$ kubectl apply -f 1-deploy-app/sockshop-virtual-services.yaml
```
Along with virtual services, destination rules are a key part of Istio’s traffic routing functionality. You can think of virtual services as how you route your traffic to a given destination, and then you use destination rules to configure what happens to traffic for that destination.

3. Configure Istio ingress gateway
```bash
$ kubectl apply -f 1-deploy-app/sockshop-gateway.yaml
```
An ingress Gateway describes a load balancer operating at the edge of the mesh that receives incoming HTTP/TCP connections. It configures exposed ports, protocols, etc. but, unlike Kubernetes Ingress Resources, does not include any traffic routing configuration.

4. Verifying our config
```bash
$ istioctl proxy-status
```
Using the `istioctl proxy-status` command allows us to get an overview of our mesh. If you suspect one of your sidecars isn’t receiving configuration or is not synchronized, proxy-status will let you know. 

If everything is fine, run the below command to open sock-shop app in your browser
```bash
$ open "http://$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'):$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')"
```
If everything is fine you should see the app up and running along with some socks :) 
5. User accounts
|Username |	Password|
|---------|:--------|
|user	  | password|
|user1	  | password|

### 2. Traffic Management
Istio’s traffic routing rules let you easily control the flow of traffic and API calls between services. Istio simplifies configuration of service-level properties like circuit breakers, timeouts, and retries, and makes it easy to set up important tasks like A/B testing, canary rollouts, and staged rollouts with percentage-based traffic splits.

We start by rolling a new Deployment
```bash
$ kubectl apply -f 2-traffic-management/canary/front-end-dep-v2.yaml  
```
now we have 2 versions of the front-end app running side by side. However if you hit the browser you'll have only the v1 (blue)

#### 1. Blue/Green Deployment
Blue-green deployment is a technique that reduces downtime and risk by running two identical production environments called Blue and Green.
Now let's swicth to v1 (red) as live environment serving all production traffic. 
```bash
$ kubectl apply -f 2-traffic-management/blue-green/frontv2-virtual-service.yaml
```
Now if you check the bapp, you'll see only the v2 (red version) of our application. Same way, you can rollback at any moment to the old version

#### 2. Canary deployment
Istio’s routing rules provides important advantages; you can easily control fine-grained traffic percentages (e.g., route 1% of traffic without requiring 100 pods) and you can control traffic using other criteria (e.g., route traffic for specific users to the canary version). To illustrate, let’s look at deploying the `front-end` service and see how simple to achieve canary deployment using istio.

For that, we need to set a routing rule to control the traffic distribution by sending 20% of the traffic to the canary (v2). execute the following command
```bash
$ kubectl apply -f 2-traffic-management/canary/canary-virtual-service.yaml 
```
and refresh the page a couple of times. The majority of pages return v1 (blue), with some v2 (red) from time to time.
#### 3. Route based on some criteria
With istio, we can easily route requests when they met some desired criteria. 
For now we have v1 and v2 deployed in our clusters, we can forward all forward all users using `Firefox` to v2, and serve v1 to all other clients:
```bash
$ kubectl apply -f 2-traffic-management/route-headers/frontv2-virtual-service-firefox.yaml
```
#### 4. Mirroring
Traffic mirroring, also called shadowing, is a powerful concept that allows feature teams to bring changes to production with as little risk as possible. Mirroring sends a copy of live traffic to a mirrored service. You can then send the traffic to out-of-band of the critical request path for the primary service (Content inspection, Threat monitoring, Troubleshooting)

```bash
$ kubectl apply -f 2-traffic-management/mirorring/mirror-v2.yaml 
```
This new rule sends 100% of the traffic to v1 while mirroring the same traffic to v2. you can check the logs of v1 and v2 pods to verify that logs created in v2 are the mirrored requests that are actually going to v1.

### 3. Resiliency
#### 1. Fault injection
Fault injection is an incredibly powerful way to test and build reliable distributed applications.
Istio allows you to configure faults for HTTP traffic, injecting arbitrary delays or returning specific response codes (e.g., 500) for some percentage of traffic.
##### Delay fault
In this example. we gonna inject a five-second delay for all traffic calling the `catalogue` service. This is a great way to reliably test how our frontend app behaves on a bad network.
```bash
$ kubectl apply -f 3-resiliency/fault-injection/delay-faults/delay-fault-injection-virtual-service.yaml
```
Open the application and you can see that it takes now longer to render catalogs
##### Abort fault
Replying to clients with specific response codes, like a 429 or a 500, is also great for testing. For example, it can be challenging to programmatically test how your application behaves when a third-party service that it depends on begins to fail. Using Istio, you can write a set of reliable end-to-end tests of your application’s behavior in the presence of failures of its dependencies.

For example, we can simulate 10% of requests to `catalogue` service is failing at runtime with a 500 response code.
```bash
$ kubectl apply -f 3-resiliency/fault-injection/delay-faults/abort-fault-injection-virtual-service.yaml 
$ open "http://$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'):$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')/catalogue"
```
Refresh the page a couple of times. You'll notice that sometimes it doesn't return the json list of catalogs

#### 2. Load-Balancing Strategy
Client-side load balancing is an incredibly valuable tool for building resilient systems. By allowing clients to communicate directly with servers without going through reverse proxies, we remove points of failure while still keeping a well-behaved system.
By default, Istio uses a round-robin load balancing policy, where each service instance in the instance pool gets a request in turn. Istio supports other options that you can check [here](https://istio.io/docs/concepts/traffic-management/#load-balancing-options)

More complex load-balancing strategies such as consistent hash-based load balancing are also supported. In this example we set up sticky sessions for `catalogue` based on source IP address as the hash key.
```bash
$ kubectl apply -f 3-resiliency/load-balancing/load-balancing-consistent-hash.yaml
```
#### 3. Circuit Breaking
Circuit breaking is a pattern of protecting calls (e.g., network calls to a remote service) behind a “circuit breaker.” If the protected call returns too many errors, we “trip” the circuit breaker and return errors to the caller without executing the protected call. 

For example, we can configure circuit breaking rules for `catalogue` service and test the configuration by intentionally “tripping” the circuit breaker. To achieve this, we gonna use a simple load-testing client called [fortio](https://github.com/fortio/fortio). Fortio lets us control the number of connections, concurrency, and delays for outgoing HTTP calls. 
```bash
$ kubectl apply -f 3-resiliency/circuit-breaking/circuit-breaking.yaml 
$ kubectl apply -f 3-resiliency/circuit-breaking/fortio.yaml 
$ FORTIO_POD=$(kubectl get pod -n sock-shop| grep fortio | awk '{ print $1 }')  
$ kubectl -n sock-shop exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -c 4 -qps 0 -n 40 -loglevel Warning http://catalogue/tags
```
In the `DestinationRule` settings, we specified `maxConnections: 1` and `http1MaxPendingRequests: 1`. They indicate that if we exceed more than one connection and request concurrently, you should see some failures when the istio-proxy opens the circuit for further requests and connections. You should see something similar to this in the output:
```
Sockets used: 28 (for perfect keepalive, would be 4)
Code 200 : 14 (35.0 %)
Code 503 : 26 (65.0 %)
```
#### 4. Retries
Every system has transient failures: network buffers overflow, a server shutting down drops a request, a downstream system fails, and so on.

Istio gives you the ability to configure retries globally for all services in your mesh. More significant, it allows you to control those retry strategies at runtime via configuration, so you can change client behavior on the fly.

The following example configures a maximum of 3 retries to connect to `catalogue` service subset after an initial call failure, each with a 1s timeout.
```bash
$ kubectl apply -f 3-resiliency/retry/retry-virtual-service.yaml
```
Worth nothing to mention that retry policy defined in a `VirtualServic`e works in concert with the connection pool settings defined in the destination’s `DestinationRule` to control the total number of concurrent outstanding retries to the destination.

#### 5. Timeouts
Timeouts are important for building systems with consistent behavior. By attaching deadlines to requests, we’re able to abandon requests taking too long and free server resources.

Here we configure a virtual service that specifies a 5 second timeout for calls to the `v1` subset of the `catalogue` service:
```bash
$ kubectl apply -f 3-resiliency/timeout/timeout-virtual-service.yaml
```

We combined in the example the use of retry and timeout. The timeout represents then the total time that the client will spend waiting for a server to return a result.

### 4. Policy
#### 1. Rate limiting
Rate limiting is generally put in place as a defensive measure for services. Shared services need to protect themselves from excessive use—whether intended or unintended—to maintain service availability.
Till very soon, the recomanded way to set up rate limiting in Istio was to use [mixer policy](https://istio.io/docs/tasks/policy-enforcement/rate-limiting/). Since version 1.5 the mixer policy is deprecated and not recommended for production usage, and the  preferred way is using [Envoy native rate limiting](https://www.envoyproxy.io/docs/envoy/v1.13.0/intro/arch_overview/other_features/global_rate_limiting) instead of mixer rate limiting. 
There is no native support yet for rate limiting API with Istio. To overcome this, we'll be using the [rate limit service](https://github.com/envoyproxy/ratelimit), which is is a Go/gRPC service designed to enable generic rate limit scenarios from different types of applications.
To mimic a real world example, we suppose that we have 2 plans: 
+ Basic: 5 requests pe minute
+ Plus: 20 requests per minute

We configured Envoy rate limiting actions to look for `x-plan` and `x-account` in request headers. We also configured the descriptor match any request with the account and plan keys, such that (`'account', '<unique value>')`, `('plan', 'BASIC | PLUS')`. The `account` key doesn't specify any value, it uses each unique value passed into the rate limiting service to match. The `plan` descriptor key has two values specified and depending on which one matches (BASIC or PLUS) determines the rate limit, either 5 request per minute for `BASIC` or 20 requests per minute for `PLUS`.
```
$ kubectl apply -f 4-policy/rate-limiting/rate-limit-service.yaml
$ kubectl apply -f 4-policy/rate-limiting/date-limit-envoy-filter.yaml  
``` 
Testing the above scenarios prove that the rate limiting is working
```bash
####  BASIC PLAN
$ kubectl -n sock-shop exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -c 1 -qps 0 -n 10 -loglevel Warning -H "x-plan: BASIC" -H "x-account: user" $INGRESS_IP/catalogue
...
Sockets used: 5 (for perfect keepalive, would be 1)
Code 200 : 5 (50.0 %)
Code 429 : 5 (50.0 %)
### PLUS PLAN
$ kubectl -n sock-shop exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -c 1 -qps 0 -n 25 -loglevel Warning -H "x-plan: PLUS" -H "x-account: user2" $INGRESS_IP/catalogue
...
Sockets used: 5 (for perfect keepalive, would be 1)
Code 200 : 20 (80.0 %)
Code 429 : 5 (20.0 %)
```
#### 2. CORS
Cross-Origin Resource Sharing (CORS) is a method of enforcing client-side access controls on resources by specifying external domains that are able to access certain or all routes of your domain. Browsers use the presence of HTTP headers to determine if a response from a different origin is allowed.

For example, the following rule restricts cross origin requests to those originating from `aboullaite.me` domain using HTTP POST/GET, and sets the `Access-Control-Allow-Credentials` header to false. In addition, it only exposes `X-Foo-bar` header and sets an expiry period of 1 day.

```bash 
$ kubectl apply -f 4-policy/cors/cors-virtual-service.yaml  
```
Checking now CORS options to confirm that the config effectively took place
```bash
curl -I -X OPTIONS -H 'access-control-request-method: PUT' -H 'origin: http://aboullaite.me' http://$INGRESS_IP/catalogue
#### should return output below
HTTP/1.1 200 OK
access-control-allow-origin: http://aboullaite.me
access-control-allow-methods: POST,GET
access-control-allow-headers: X-Foo-Bar
access-control-max-age: 86400
date: Sat, 23 May 2020 15:40:05 GMT
server: istio-envoy
content-length: 0
```
### 5. Security
#### 1. mutual TLS authentication
With all of the identity certificates (SVIDs) distributed to workloads across the system, how do we actually use them to verify the identity of the servers with which we’re communicating and perform authentication and authorization? This is where mTLS comes into play.
mTLS is TLS in which both parties, client and server, present certificates to each other. This allows the client to verify the identity of the server, like normal TLS, but it also allows the server to verify the identity of the client attempting to establish the connection. 
In this example, we will migrate the existing Istio services traffic from plaintext to mutual TLS without breaking live traffic.

Istio 1.5 brings the concept of PeerAuthentication, which is a CRD that allows us to enable and configure mTLS at both the cluster level and namespace level. First we start by enabling mTLS:
```bash
$ kubectl apply -f 5-security/mtls/peer-auth-mtls.yaml
$ kubectl apply -f 5-security/mtls/destination-rule-tls.yml  
```

To confirm that plain-text requests fail as TLS is required to talk to any service in the mesh, we redeploy fortlio by disabling sidecare injection this time. and run some requests
```bash
$ kubectl apply -f 5-security/fortio.yaml
$ kubectl -n sock-shop exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -curl -k http://catalogue/tags  
```
You should notice that fortio fails to make calls to `catalogue` service, and we get a `Connection reset by peer` which is what we expected.
Now how do we get a successful connection? In order to have applications communicate over mutual TLS, they need to be onboarded onto the mesh. Or we can disable mTLS for `catalogue` service
```bash
$ kubectl apply -f 5-security//disable-mtls-catalogue.yaml
$ kubectl -n sock-shop exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -curl -k http://catalogue/tags  
```
You can see that the request was successful! But if you go to our app, you should notice that no catalogues are returned, we should re-enable mtls again in order to work:
```bash
$ kubectl apply -f 5-security/mtls/peer-auth-mtls.yaml
$ kubectl apply -f 5-security/mtls/destination-rule-tls.yml  
```

--- 
Ressources:
+ [Istio documentation](https://istio.io/docs/t)
+ [Microservices demo](https://microservices-demo.github.io/)
+ [istio up and running](https://www.oreilly.com/library/view/istio-up-and/9781492043775/)