```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-exam
  namespace: tenant-b
spec:
  endpointSelector: {}
  egress:
    - toFQDNs:
        - matchName: google.com
      toPorts:
        - ports:
            - port: "443"
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
          rules:
            dns:
              - matchPattern: "*"
    - toEndpoints:
        - matchLabels:
            k8s:app: backend-service
            k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name: tenant-c
            k8s:io.kubernetes.pod.namespace: tenant-c
      toPorts:
        - ports:
            - port: "80"
    - toEndpoints:
        - matchLabels:
            namespace: tenant-c
            pod: backend-service
      toPorts:
        - ports:
            - port: "80"
```
