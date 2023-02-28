# Network Policies in kubernetes

## Summary: 
By default, in kubernetes everything is opened. This course will let you understand the steps you need to isolated communication within a namespace.

## Prerequisite

At this point, you should have acces to a kubernetes environment with at least two nodes.
Please read the following instructions on this channel to set up a secure kubernetes virtual environment with a devkit.

## The Lab

### Step 1 : create two basics namespaces 

```bash
# create the DEV namespace
kubectl create -f ~/learnkubenetes/manifests/course1/namespace-dev.yaml
# create the PROD namespace
kubectl create -f ~/learnkubenetes/manifests/course1/namespace-prod.yaml
```

### Step 2 : create a container inside each namespace

```bash
# create the DEV pod
kubectl apply -f ~/learnkubenetes/manifests/course1/pod-dev.yaml
# create the PROD pod
kubectl apply -f ~/learnkubenetes/manifests/course1/pod-prod.yaml
```
The containers are based on Ubuntu and provide networks tools like *ping, ip, nmap, netstat* and a SSH server and client.

### Step 3 : valide the lab
Copy the next command in the shell
```bash
echo "Get namespaces" && kubectl get namespaces --show-labels | grep -v "^kube"  
echo "\nGet pods"     && kubectl get pods -o wide --show-labels -A | grep -v "^kube"
```

It should return
<pre>
Get namespaces
NAME              STATUS   AGE     LABELS
default           Active   TTTh    kubernetes.io/metadata.name=default
development       Active   TTTm    kubernetes.io/metadata.name=development,name=development
production        Active   TTTm    kubernetes.io/metadata.name=production,name=production

Get pods
NAMESPACE     NAME         READY   STATUS    RESTARTS        AGE     IP         NODE        NO...TES   LABELS
development   devpod1      1/1     Running   0               TTTm    10.X.X.X   FQDN-node1  ...        app=devpod1
production    prodpod1     1/1     Running   0               TTTm    10.Y.Y.Y   FQDN-node2  ...        app=prodpod1
</pre>

As you can see:
- the pod **devpod1** is located in the **development** namespace, with the IP adress *10.X.X.X*
- the pod **prodpod1** is located in the **production** namespace, with the IP adress *10.Y.Y.Y*

> **_NOTE:_**  10.X.X.X and 10.Y.Y.Y should be replace by your values. 

## Exercice 1 : Testing the connectivity without policies (default behavior)

Let kick-off a shell in the container **devpod1**. 

```bash
kubectl exec --namespace=development --stdin --tty devpod1 -- /bin/bash
```
> **_NOTE:_** We explicit wrote down the --namespace=development, because **devpod1** is not located in the --namespace=default. We will cover more in details *kubectl* in another course.

Let look at the container itself IP address, it shoulb be *10.X.X.X*

```bash
root@devpod1:/# ip addr show eth0
3: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether ae:43:e2:81:04:89 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.X.X.X/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::ac43:e2ff:fe81:489/64 scope link 
       valid_lft forever preferred_lft forever
```
Because each pod have it own DNS A record, both of them will be able to resolv each other.

> **_NOTE:_**  Pods are assigned a DNS A record in the form of pod-ip-address.my-namespace.pod.cluster.local . For example, a pod with IP 172.12.3.4 in the namespace default with a DNS name of cluster.local would have an entry of the form 172–12–3–4.default.pod.cluster.local. In our cases:
> - 10-X-X-X.development.pod.cluster.local is the FQDN of **devpod1** 
> - 10-Y-Y-Y.production.pod.cluster.local is the FQDN of **prodpod1** 
> Can we change the domain? We will discuss that in more detail in another course.
```bash
root@devpod1:/# nslookup 10-X-X-X.development.pod.cluster.local
Server:		10.152.183.10
Address:	10.152.183.10#53

Name:	10.X.X.X.development.pod.cluster.local
Address: 10.X.X.X

root@devpod1:/# nslookup 10-Y-Y-Y.production.pod.cluster.local
Server:		10.152.183.10
Address:	10.152.183.10#53

Name:	10.Y.Y.Y.production.pod.cluster.local
Address: 10.Y.Y.Y
```

At this point, wihout any network policies, **devpod1** is capabled to *ping* and connect to port *ssh* on **prodpod1** even if the two pods are in different namespace).
```bash
root@devpod1:/# ping -c 2 10.Y.Y.Y
PING 10.Y.Y.Y (10.Y.Y.Y) 56(84) bytes of data.
64 bytes from 10.Y.Y.Y: icmp_seq=1 ttl=62 time=0.554 ms
64 bytes from 10.Y.Y.Y: icmp_seq=2 ttl=62 time=0.661 ms

--- 10.Y.Y.Y ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1008ms
rtt min/avg/max/mdev = 0.554/0.607/0.661/0.053 ms

root@devpod1:/# nmap 10.Y.Y.Y -p 22
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-27 16:38 EST
Nmap scan report for 10.Y.Y.Y
Host is up (0.00065s latency).

PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 1 IP address (1 host up) scanned in 0.33 seconds
```
As you can see **devpod1** as no restriction on **prodpod1**


## Exercice 2 : Blocking cross namespaces communication

Namespaces are “hidden” from each other, but they are not fully isolated by default. In the previous exercice, we saw  a pod in one Namespace can talk to a pod in another Namespace. This can often be very useful, for example to have your team’s pods in your Namespace communicate with another team’s pods in another Namespace. Generally, we want to control the communications with *NetworkPolicies*. 

*NetworkPolicies* are an **application-centric** construct which allow you to specify how a "entitie" is allowed to communicate with various network "entities" over the network. *NetworkPolicies* apply to a connection with a pod on one or both ends, and are not relevant to other connections.

The entities that a Pod can communicate with are identified through a combination of the following 3 identifiers:

- Other pods that are allowed (exception: a pod cannot block access to itself) 
- Namespaces that are allowed
- IP blocks (exception: traffic to and from the node where a Pod is running is always allowed, regardless of the IP address of the Pod or the node)

When defining a pod- or namespace- based *NetworkPolicies*, you use a selector to specify what traffic is allowed to and from the Pod(s) that match the selector.

As soon as you have a *NetworkPolicies* that selects a certain group of Pods, those Pods become isolated and reject any traffic that is not allowed by any *NetworkPolicies*.

> **_NOTE:_**  that *NetworkPolicies* are additive, so having two *NetworkPolicies* that select the same Pods will result in allowing both defined policies.
> 
> Keep in mind that a *NetworkPolicies*is applied to a particular Namespace and only selects Pods in that particular Namespace.