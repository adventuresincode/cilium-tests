### MTLS Cilium
Uses SPIRE...   

Deploy the Star Wars environment and the L3/L4 network policy used in the demo (there's no mutual authentication in the network policy yet).   
```sh
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/http-sw-app.yaml
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/sw_l3_l4_policy.yaml
```


Review the network policy:
```sh
kubectl get cnp rule1 -o jsonpath='{.spec}' | jq
```
Should look like this
```yaml
{
  "description": "L3-L4 policy to restrict deathstar access to empire ships only",
  "endpointSelector": {
    "matchLabels": {
      "class": "deathstar",
      "org": "empire"
    }
  },
  "ingress": [
    {
      "fromEndpoints": [
        {
          "matchLabels": {
            "org": "empire"
          }
        }
      ],
      "toPorts": [
        {
          "ports": [
            {
              "port": "80",
              "protocol": "TCP"
            }
          ]
        }
      ]
    }
  ]
}
```

### Check the network policy
Let's verify that the connectivity model is the expected one.

First, verify that the Death Star Deployment is ready:
```sh
kubectl rollout status deployment deathstar -w
```
reported status:
```sh
deployment "deathstar" successfully rolled out
```

Verify that Tie Fighters (Empire space ships) are allowed to land on the Death Star:
```sh
kubectl exec tiefighter -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```
Should return : **Ship landed**

Verify that the X-Wing ship (belong to the Alliance) is denied access to the Death Star:
```sh
kubectl exec xwing -- \
  curl -s --connect-timeout 1 -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```
Should return:
```sh
command terminated with exit code 28
```

[](./media/star_wars_l3l4.png)
   
The xwing cannot connect to the deathstar. And while the Empire security officers are aware, a HTTP call to a particular path might cause the Deathstar to explode, surely no officers would want to cause damage to the Empire.

### Security Issue
Despite the presence of the L3/L4 network policies, the rebels were somehow able to take control of a tiefighter and cause the Deathstar to explode!
```sh
# kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Panic: deathstar exploded

goroutine 1 [running]:
main.HandleGarbage(0x2080c3f50, 0x2, 0x4, 0x425c0, 0x5, 0xa)
        /code/src/github.com/empire/deathstar/
        temp/main.go:9 +0x64
main.main()
   /code/src/github.com/empire/deathstar/
        temp/main.go:5 +0x85
```


### Rolling out MTLS
Rolling out mutual authentication with Cilium is as simple as adding the following to an existing or new CiliumNetworkPolicy.
```yaml
spec:
  egress|ingress:
    authentication:
        mode: "required"
```

The network policy looks like this:
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "Mutual authentication enabled L7 policy"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    authentication:
      mode: "required"
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/v1/request-landing"
```


To review changes we will be making to an existing network policy:
```sh
KUBECTL_EXTERNAL_DIFF='colordiff -u' \
  kubectl diff -f sw_l3_l4_l7_mutual_authentication_policy.yaml | \
  grep -A30 ' spec:'
```
The result
```yaml
 spec:
-  description: L3-L4 policy to restrict deathstar access to empire ships only
+  description: Mutual authentication enabled L7 policy
   endpointSelector:
     matchLabels:
       class: deathstar
       org: empire
   ingress:
-  - fromEndpoints:
+  - authentication:
+      mode: required
+    fromEndpoints:
     - matchLabels:
         org: empire
     toPorts:
     - ports:
       - port: "80"
         protocol: TCP
+      rules:
+        http:
+        - method: POST
+          path: /v1/request-landing
```

The notable differences are:    
+ we are changing the description of the policy
+ we are adding L7 filtering (only allowing HTTP POST to the /v1/request-landing)
+ we are adding authentication.mode: required to our ingress rules. This will ensure that, in addition to the existing policy requirements, ingress access is only for mutually authenticated workloads.   

Let's now apply this policy.
```sh
kubectl apply -f sw_l3_l4_l7_mutual_authentication_policy.yaml
```

### Verifying Connectivity after Enabling Mutual Authentication
Re-try the connectivity tests.

Let's start with the tiefighter calling the /request-landing path :
```sh
kubectl exec tiefighter -- \
  curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```
**This should still succeed.**

Let's then try access from the tiefigher to the /exhaust-port path:
```sh
kubectl exec tiefighter -- \
  curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
```  
This second request should be denied , thanks to the new L7 Network Policy, preventing any tiefighter - compromised or not - from accessing the /exhaust-port.    

Executing:
```sh
kubectl exec xwing -- \
  curl -s --connect-timeout 1 -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```
This third one should time out, thanks to the L3/L4 Network Policy.

### Verify MTLS is in order
Validate Hubble is working
```sh
hubble --version
```

Observe Mutual Authentication with Hubble.   

Run the connectivity checks again:
```sh
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
kubectl exec xwing -- curl -s --connect-timeout 1 -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

Let's look at flows from the xwing to the deathstar.   
The network policy should have dropped flows from the xwing as the xwing has not got the right labels.  
Run
```sh
hubble observe --type drop --from-pod default/xwing
```

This should return
```sh

Dec 28 12:53:01.530: default/xwing:36844 (ID:7360) <> default/deathstar-8464cdd4d9-qgzbd:80 (ID:40182) Policy denied DROPPED (TCP Flags: SYN)
Dec 28 13:01:29.426: default/xwing:42300 (ID:7360) <> default/deathstar-8464cdd4d9-p58cs:80 (ID:40182) Policy denied DROPPED (TCP Flags: SYN)
Dec 28 13:03:47.606: default/xwing:38934 (ID:7360) <> default/deathstar-8464cdd4d9-qgzbd:80 (ID:40182) Policy denied DROPPED (TCP Flags: SYN)
```

The policy verdict for this traffic should be DROPPED by the L3/L4 section of the Network Policy.   

Let's now look at traffic from the **tiefighter** to the deathstar.   
The network policy should have dropped the first flow from the tiefighter to the deathstar Service over /request-landing.   

The network policy should have dropped the first flow from the tiefighter to the deathstar Service over /request-landing. Why ? Because the first packet to match the mutual authentication-based network policy will kickstart the mutual authentication handshake.
```sh
hubble observe --type drop --from-pod default/tiefighter
```
You should see a similar behaviour when looking for flows with the policy-verdict filter:
```sh
hubble observe --type policy-verdict --from-pod default/tiefighter
```
Result: 
```sh
Dec 28 13:00:45.658: default/tiefighter:37618 (ID:30913) <> default/deathstar-8464cdd4d9-p58cs:80 (ID:40182) policy-verdict:L3-L4 INGRESS DENIED (TCP Flags: SYN; Auth: SPIRE)
Dec 28 13:00:46.690: default/tiefighter:37618 (ID:30913) -> default/deathstar-8464cdd4d9-p58cs:80 (ID:40182) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN; Auth: SPIRE)
Dec 28 13:00:51.385: default/tiefighter:45532 (ID:30913) -> default/deathstar-8464cdd4d9-p58cs:80 (ID:40182) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN; Auth: SPIRE)
Dec 28 13:03:46.408: default/tiefighter:44616 (ID:30913) <> default/deathstar-8464cdd4d9-qgzbd:80 (ID:40182) policy-verdict:L3-L4 INGRESS DENIED (TCP Flags: SYN; Auth: SPIRE)
Dec 28 13:03:47.422: default/tiefighter:44616 (ID:30913) -> default/deathstar-8464cdd4d9-qgzbd:80 (ID:40182) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN; Auth: SPIRE)
```

Let's explain these 3 lines of logs.

1️⃣ ALLOWED log (no mutual auth)
```sh
default/tiefighter:58032 (ID:1985) -> default/deathstar-f694cf746-vdds5:80 (ID:9215) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
```
[In my experiement this message was not visible. This is possibly because I did not attempt to access the endpoint beforfe the MTLS was applied.]

The first request was allowed as it happened before we applied Mutual Authentication (note that Auth: SPIRE is not in the error message).

2️⃣ DENIED (mutual auth)
```sh
default/tiefighter:45302 (ID:24806) <> default/deathstar-f694cf746-28vbm:80 (ID:9076) policy-verdict:L3-L4 INGRESS DENIED (TCP Flags: SYN; Auth: SPIRE)
```
The second request was denied because the mutual authentication handshake had not completed yet.

3️⃣ ALLOWED (mutual auth)
```sh
default/tiefighter:45302 (ID:24806) -> default/deathstar-f694cf746-28vbm:80 (ID:9076) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN; Auth: SPIRE)
```
The last request was successful, as the handshake was successful.   

*** Enabling MTLS requeries enablement at the globala level ***

###  A suspicious Emperor
The Emperor's paranoia is getting out of control.

While you impressed him with your knowledge of cloud native security, he wants to know exactly where the identities of the officers are stored and whether certificates are automatically issued and rotated.

It's time for you to explain to the Emperor how Cilium integrates with SPIFFE.

### 🪪 Identity Management
To address the challenges of identity verification in dynamic and heterogeneous environments, we need a framework to secure identity verification for distributed systems.

In Cilium’s current mutual auth support, that is provided through SPIFFE (Secure Production Identity Framework for Everyone).


### SPIFFE Benefits
Here are some of the benefits of SPIFFE:

*1. Trustworthy identity issuance:*
SPIFFE provides a standardized mechanism for issuing and managing identities. It ensures that each service in a distributed system receives a unique and verifiable identity, even in dynamic environments where services may scale up or down frequently.

*2. Identity attestation:*
SPIFFE allows services to prove their identities through attestation. It ensures that services can demonstrate their authenticity and integrity by providing verifiable evidence about their identity, such as digital signatures or cryptographic proofs.

*3. Dynamic and scalable environments:*
SPIFFE addresses the challenges of identity management in dynamic environments. It supports automatic identity issuance, rotation, and revocation, which are critical in cloud-native architectures where services may be constantly deployed, updated, or retired.  

By combining Cilium mutual authentication with SPIFFE, we establish a robust and scalable security infrastructure that provides strong mutual authentication and verifiable identities in dynamic distributed systems.


### SPIRE and Cilium
A SPIRE server was automatically deployed when installing Cilium with the mutual authentication feature.  

The SPIRE environment will manage the TLS certificates for the workloads managed by Cilium.


** 🩺 Verify SPIRE Health **
Let's first verify that the SPIRE server and agents automatically deployed are working as expected.

The SPIRE server is deployed as a StatefulSet and the SPIRE agents are deployed as a DaemonSet (you should therefore see one SPIRE agent per node). Check them with:   
```sh
kubectl get all -n cilium-spire
```
The SPIRE server StatefulSet and Spire agent DaemonSet should both be Ready.   
+ The Spire agents will only run on the worker node
+ The Spire server instances are also running on the worker nodes. The control plane is tainted and the serer is not tolerated on it.

```
kubectl get statefulset.apps --all-namespaces -o wide --field-selector spec.nodeName=kind-control-plane
```

Let's run a healthcheck on the SPIRE server.
```sh
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server healthcheck
```
Expect a healthy response:
```sh
Server is healthy.
```

Let's verify the list of SPIRE agents:

```sh
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server agent list
```
Expect a reply such as the following:
```sh
Found 2 attested agents:

SPIFFE ID         : spiffe://spiffe.cilium/spire/agent/k8s_psat/default/7341eacf-763d-4396-a455-8db3e3323c68
Attestation type  : k8s_psat
Expiration time   : 2023-05-17 17:35:42 +0000 UTC
Serial number     : 87767126404300269148814680099485251869

SPIFFE ID         : spiffe://spiffe.cilium/spire/agent/k8s_psat/default/95bffb3d-4677-4618-ad1e-7244f4a81c41
Attestation type  : k8s_psat
Expiration time   : 2023-05-17 17:35:45 +0000 UTC
Serial number     : 100095087906107800458890473530290535817
Note that there are 2 agents, one per node (and we have two nodes in this cluster).
```

Note that there are 2 agents, one per node (and we have two nodes in this cluster).    

Notice as well that the SPIRE Server uses Kubernetes Projected Service Account Tokens (PSATs) to verify the identity of a SPIRE Agent running on a Kubernetes Cluster.   

**Verify SPIFFE identity for Cilium**
Now that we know the SPIRE service is healthy, let's verify that the Cilium and SPIRE integration has been successful.

First, verify that the Cilium agent and operator have identities on the SPIRE server:
```sh
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server entry show -parentID spiffe://spiffe.cilium/ns/cilium-spire/sa/spire-agent
```  
Expect an output such as:
```sh
Found 2 entries
Entry ID         : 7a4a01ed-9d3c-4262-8dc1-f0d425a9e3fc
SPIFFE ID        : spiffe://spiffe.cilium/cilium-agent
Parent ID        : spiffe://spiffe.cilium/ns/cilium-spire/sa/spire-agent
Revision         : 0
X509-SVID TTL    : default
JWT-SVID TTL     : default
Selector         : k8s:ns:kube-system
Selector         : k8s:sa:cilium

Entry ID         : 03f7ed38-6412-4792-94ce-19280723fbf9
SPIFFE ID        : spiffe://spiffe.cilium/cilium-operator
Parent ID        : spiffe://spiffe.cilium/ns/cilium-spire/sa/spire-agent
Revision         : 0
X509-SVID TTL    : default
JWT-SVID TTL     : default
Selector         : k8s:ns:kube-system
Selector         : k8s:sa:cilium-operator
```


Note the SPIFFE ID and Selector fields, which point to two workloads: the Cilium agent and the Cilium Operator, showing that the Cilium agent and operator each have a registered delegate identity with the SPIRE Server.   

Let's now verify that the Cilium operator has registered identities with the SPIRE server on behalf of the workloads (Kubernetes Pods).

**🌐 Verify SPIFFE identity for the Death Star**
First, get the Cilium Identity of the deathstar Pods:
```sh
IDENTITY_ID=$(kubectl get cep -l app.kubernetes.io/name=deathstar -o=jsonpath='{.items[0].status.identity.id}')
echo $IDENTITY_ID
```
Result:
```sh
40182
```
**Note**   
>Even though there are two of these Pods, they share the same Cilium Identity, since they use the same set of Kubernetes labels.

The SPIFFE ID —that uniquely identifies a workload— is based on the Cilium identity. It follows the ```spiffe://spiffe.cilium/identity/$IDENTITY_ID``` format.   

Verify that the Death Star pods have a registered SPIFFE identity on the SPIRE server:
```sh
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server entry show -spiffeID spiffe://spiffe.cilium/identity/$IDENTITY_ID
```
Expect an output such as:
```sh
Found 1 entry
Entry ID         : a580f893-0e7f-4717-96e6-c125566cabe3
SPIFFE ID        : spiffe://spiffe.cilium/identity/48927
Parent ID        : spiffe://spiffe.cilium/cilium-operator
Revision         : 0
X509-SVID TTL    : default
JWT-SVID TTL     : default
Selector         : cilium:mutual-auth

```

You can see the that the cilium-operator was listed as the Parent ID. That is because the Cilium operator is responsible for creating SPIRE entries for each Cilium identity.
List all the registration entries with:
```sh
kubectl exec -n cilium-spire spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server entry show -selector cilium:mutual-auth
```
There are as many entries as there are identities. Verify this these match by listing the Cilium identities in the cluster:
```sh
kubectl get ciliumidentities
```
The identify ID listed under NAME should match the digits at the end of the SPIFFE ID executed in the previous command.

**📃 SPIFFE Verifiable Identity Document (SVID)**
An SVID is the document with which a workload proves its identity to a resource or caller. An SVID is considered valid if it has been signed by an authority within the SPIFFE ID’s trust domain.

An SVID contains a single SPIFFE ID, which represents the identity of the service presenting it. It encodes the SPIFFE ID in a cryptographically-verifiable document in an X.509 certificate.

One of the reasons for choosing SPIRE for the initial implementation of Cilium Mutual Authentication is that it will automatically rekey SVIDs before their certificate expires, and when this happens, it will notify SVID watchers, which includes the Cilium Agent.