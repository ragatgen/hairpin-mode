AKS 1.35 Kubenet Hairpin Issue – Why Pod-to-Own-Service Fails and How to Fix It
Overview

While validating workloads on Azure Kubernetes Service (AKS) 1.35, we identified a networking issue affecting clusters using:

Kubenet
Ubuntu 24.04 node images

In this configuration, a pod cannot reach its own Service ClusterIP, resulting in connection timeouts.

This is not an application issue—it is a network datapath behavior related to hairpin traffic handling.

What is the Problem?
Symptom

From inside a pod:

curl http://my-service

Result:

Connection timed out

But:

DNS resolves correctly ✅
Service exists and endpoints are correct ✅
Same setup works on:
AKS 1.34 + kubenet
AKS 1.35 + Azure CNI Overlay
Why This Happens (Root Cause)
1. Kubernetes Service Flow

When a pod calls a Service:
```bash
Pod → Service VIP → kube-proxy → DNAT → Pod Endpoint
```
    
Explanation
Pod calls its own Service
kube-proxy correctly DNATs to pod endpoint
Return traffic must re-enter same pod (hairpin path)
❌ Blocked because:
hairpin_mode=0
bridge not processing iptables
Result: timeout


Example:

```bash

curl http://hairpin-test-svc

```
# hangs or times out

After (Hairpin Enabled – Working Path)


flowchart LR
```bash
    A[Pod A<br>10.244.0.10] -->|curl hairpin-test-svc| B[Service ClusterIP<br>10.0.225.178]
```

    B -->|kube-proxy DNAT| C[Pod A Endpoint<br>10.244.0.10]

    C -->|Return Path| D[Node Bridge cbr0]

    D -->|hairpin_mode=1<br>nf_call_iptables=1| E[Hairpin Forwarding Allowed]

    E --> F[Pod A Receives Response]

    F --> G[HTTP 200 OK ✅]

    

Test Matrix
Configuration	Result
AKS 1.35 + Kubenet + Ubuntu 24.04	❌ Fails
AKS 1.35 + Azure CNI Overlay + Ubuntu 24.04	✅ Works
AKS 1.34 + Kubenet + Ubuntu 22.04	✅ Works

Root Cause

This issue is caused by disabled hairpin networking behavior at the node level.

Hairpin traffic occurs when:

A pod accesses its own Service ClusterIP, and traffic is routed back to the same pod.

This requires proper handling of:

Bridge forwarding
NAT return path
veth interface hairpin mode
Observed Node State

Create a node debugger:
```bash
kubectl debug node/aks-nodepool1-36079231-vmss000002 -it --image=mcr.microsoft.com/aks/fundamental/base-ubuntu:v0.0.11

```

Then Run:
```bash

cat /sys/class/net/cbr0/bridge/nf_call_iptables
# 0


for i in /sys/class/net/*/brport/hairpin_mode; do echo "$i: $(cat $i)"; done
# all = 0

```

This indicates:

Hairpin forwarding is disabled
Bridge traffic is not processed through iptables



Validation

We applied a controlled change at the node level using a DaemonSet.

```bash

DaemonSet (Hairpin Enablement)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: hairpin-fix
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: hairpin-fix
  template:
    metadata:
      labels:
        app: hairpin-fix
    spec:
      hostNetwork: true
      hostPID: true
      tolerations:
      - operator: Exists
      containers:
      - name: hairpin-fix
        image: mcr.microsoft.com/aks/fundamental/base-ubuntu:v0.0.11
        securityContext:
          privileged: true
        command:
        - /bin/bash
        - -lc
        - |
          while true; do
            if [ -f /host-sys/class/net/cbr0/bridge/nf_call_iptables ]; then
              echo 1 > /host-sys/class/net/cbr0/bridge/nf_call_iptables || true
            fi

            for f in /host-sys/class/net/*/brport/hairpin_mode; do
              [ -f "$f" ] || continue
              echo 1 > "$f" || true
            done

            sleep 10
          done
        volumeMounts:
        - name: host-sys
          mountPath: /host-sys
      volumes:
      - name: host-sys
        hostPath:
          path: /sys
          type: Directory
```

Result

After applying the DaemonSet:

```bash

randra@Lenovo-WorkRG:~/databricks-lab$ POD=$(kubectl get pods | grep hairpin-test | awk '{print $1}')
randra@Lenovo-WorkRG:~/databricks-lab$ kubectl exec -it "$POD" -- curl -v --connect-timeout 5 http://hairpin-test-svc
* Host hairpin-test-svc:80 was resolved.
* IPv6: (none)
* IPv4: 10.0.225.178
*   Trying 10.0.225.178:80...
* Established connection to hairpin-test-svc (10.0.225.178 port 80) from 10.244.1.6 port 59260 
* using HTTP/1.x
> GET / HTTP/1.1
> Host: hairpin-test-svc
> User-Agent: curl/8.17.0
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Server: nginx/1.29.7
< Date: Sun, 29 Mar 2026 15:50:31 GMT
< Content-Type: text/html
< Content-Length: 896
< Last-Modified: Tue, 24 Mar 2026 17:22:07 GMT
< Connection: keep-alive
< ETag: "69c2c83f-380"
< Accept-Ranges: bytes
< 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy, 
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional 
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host hairpin-test-svc:80 left intact
randra@Lenovo-WorkRG:~/databricks-lab$

```


Conclusion
The issue is not related to the application
The issue is not related to DNS or Service configuration
The issue is caused by disabled hairpin behavior in kubenet datapath
This is specific to:
AKS 1.35
Ubuntu 24.04
kubenet
Important Disclaimer

⚠️ The DaemonSet workaround:

Modifies node runtime state
Is not persistent across:
node reboot
node replacement
upgrades
Is not supported by AKS for production use

Use only for:

Testing
Validation
Root cause confirmation
Recommended Solution

Move away from kubenet.

Preferred
Azure CNI Overlay
Supported and recommended
Works correctly on AKS 1.35
Alternative
Stay on:
AKS 1.34
Ubuntu 22.04
Key Takeaways
Pod → own Service failure = hairpin issue
kubenet relies on bridge + iptables behavior
AKS 1.35 + Ubuntu 24.04 introduces behavior change/regression
Azure CNI avoids this limitation
References
Kubernetes: Debugging Service Issues
https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/
Author Notes

This issue was identified during real-world validation of production workloads and demonstrates the importance of:

Testing new Kubernetes versions with different networking models
Understanding node-level datapath behavior
Validating assumptions with controlled experiments
