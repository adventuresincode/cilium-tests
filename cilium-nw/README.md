## Cilium Networking

Star Wars inspired demo app:   
endor.yml
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: endor
---
apiVersion: v1
kind: Service
metadata:
  namespace: endor
  name: deathstar
  labels:
    app.kubernetes.io/name: deathstar
spec:
  type: ClusterIP
  ports:
  - port: 80
    name: http
  selector:
    org: empire
    class: deathstar
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: endor
  name: deathstar
  labels:
    app.kubernetes.io/name: deathstar
spec:
  replicas: 2
  selector:
    matchLabels:
      org: empire
      class: deathstar
  template:
    metadata:
      labels:
        org: empire
        class: deathstar
        app.kubernetes.io/name: deathstar
    spec:
      containers:
      - name: deathstar
        image: docker.io/cilium/starwars
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: endor
  name: tiefighter
spec:
  replicas: 1
  selector:
    matchLabels:
      org: empire
      class: tiefighter
      app.kubernetes.io/name: tiefighter
  template:
    metadata:
      labels:
        org: empire
        class: tiefighter
        app.kubernetes.io/name: tiefighter
    spec:
      containers:
        - name: starship
          image: docker.io/tgraf/netperf
          command: ["/bin/sh"]
          args: ["-c", "while true; do curl -s -XPOST deathstar.endor.svc.cluster.local/v1/request-landing; curl -s https://disney.com; curl -s https://swapi.dev/api/starships; sleep 1; done"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: endor
  name: xwing
spec:
  replicas: 1
  selector:
    matchLabels:
      org: alliance
      class: xwing
      app.kubernetes.io/name: xwing
  template:
    metadata:
      labels:
        org: alliance
        class: xwing
        app.kubernetes.io/name: xwing
    spec:
      containers:
        - name: starship
          image: docker.io/tgraf/netperf
          command: ["/bin/sh"]
          args: ["-c", "while true; do curl -s -XPOST deathstar.endor.svc.cluster.local/v1/request-landing; curl -s https://disney.com; curl -s https://swapi.dev/api/starships; sleep 1; done"]
```

Apply the yaml and check the status:
```sh
kubectl apply -f endor.yml
kubectl get -f endor.yml
```

All traffic coming from the xwing identity is being dropped - this is because the endor namespace has been secures with Network Policies that allow only specific traffic.


**Pixie Dust:**  
Annotate the deathstar pod to add layer 7 visibility on 80/TCP ingress traffic, parsing it as HTTP.
```sh
kubectl -n endor annotate pods -l class=deathstar policy.cilium.io/proxy-visibility="<Ingress/80/TCP/HTTP>"
```

To check the network policy:   
```sh
kubectl -n endor get ciliumnetworkpolicy deathstar -o yaml | yq '.spec'
```

### References
1. https://swapi.dev/documentation