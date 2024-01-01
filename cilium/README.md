Kubernetes labels:   
+ the Death Star: org=empire, class=deathstar
+ the Imperial TIE fighter: org=empire, class=tiefighter
+ the Rebel X-Wing: org=alliance, class=xwing

The deployment also includes a deathstar-service, which load-balances traffic to all pods with label org=empire, class=deathstar.   

Let's install everything via the manifest http-sw-app.yaml:   

```sh
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/http-sw-app.yaml
```


Each pod will also be represented in Cilium as an Endpoint. To retrieve a list of all endpoints managed by Cilium, the Cilium Endpoint (or cep) resource can be used:
```sh
kubectl get cep --all-namespaces
```


A simple API call:
```sh
kubectl exec tiefighter -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```


This command should timeout:
```sh
kubectl exec xwing -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

The curl should not be permitted, a security policy is missing.

### Security Policy
IP addresses are no longer relevant for Cloud Native workloads. Security policies need something else.   
Cilium provides this: >Cilium uses the labels assigned to pods to define security policies.   

A basic policy to restrict deathstar landing requests to the ships that have label org=empire only.   

Cilium Network Policy:
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: rule1
spec:
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
    - fromEndpoints:
        - matchLabels:
            org: empire
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP

```

The same as Kubernetes network policy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: rule1
spec:
  podSelector:
    matchLabels:
      org: empire
      class: deathstar
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              org: empire
      ports:
        - port: 80
          protocol: TCP
```


This policy provides the access to the API, but does not do a granular permission check.   
To provide the strongest security (i.e., enforce least-privilege isolation) between microservices: each service that calls deathstar’s API should be limited to making only the set of HTTP requests it requires for legitimate operation.    
Before applying the policy:
```sh
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/sw_l3_l4_l7_policy.yaml
```
Results:
```sh
Panic: deathstar exploded

goroutine 1 [running]:
main.HandleGarbage(0x2080c3f50, 0x2, 0x4, 0x425c0, 0x5, 0xa)
        /code/src/github.com/empire/deathstar/
        temp/main.go:9 +0x64
main.main()
        /code/src/github.com/empire/deathstar/
        temp/main.go:5 +0x85
```
Apply the policy:
```sh
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/sw_l3_l4_l7_policy.yaml
```

After applying the policy:
```sh
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/sw_l3_l4_l7_policy.yaml
```
Results:
```sh
Access Denied
```

With Cilium L7 security policies, we are able to restrict tiefighter's access to only the required API resources on deathstar, thereby implementing a “least privilege” security approach for communication between microservices.
