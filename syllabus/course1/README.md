# Network Policies in kubernetes

## Table of contents
1. Summary
2. Prerequisite
3. [Cross namespace communication](#CrossName)
    1. [A simple lab](#createlab1)
    2. [Exercice 1 : Testing the connectivity without policies (default behavior)](#Exercice1)
    3. [Exercice 2 : Blocking cross namespaces communication](#Exercice2)
    4. [Exercice 3 : Allowing only SSH connexion from the default namespace to the development namespace](#Exercice3)
    5. [Delete the simple lab](#dellab1)
3. A more complex lab for the next exercice
8. [Exercice 4 : Segregation within a namespace](#Exercice4)


## Summary: 
By default, in kubernetes everything is opened. This course will let you understand the steps you need to isolated communication within a namespace.

## Prerequisite

At this point, you should have acces to a kubernetes environment with two nodes (called NODE_A and NODE_B).
Please read the following instructions on this channel to set up a secure kubernetes virtual environment with a devkit.

## Cross namespace communication <a name="CrossName"></a>
### A simple lab <a name="createlab1"></a>

#### Step 1 : create two basics namespaces 

```bash
# create the DEV namespace
kubectl create -f ~/learnkubernetes/manifests/course1/namespace-dev.yaml
# create the PROD namespace
kubectl create -f ~/learnkubernetes/manifests/course1/namespace-prod.yaml
```

#### Step 2 : deploy two pods on each namespace

```bash
# create the DEV pod
kubectl apply -f ~/learnkubernetes/manifests/course1/deploy-dev.yaml
# create the PROD pod
kubectl apply -f ~/learnkubernetes/manifests/course1/deploy-prod.yaml
```
The containers are based on Ubuntu and provide networks tools like *ping, ip, nmap, netstat* and a SSH server and client.

#### Step 3 : valide the lab
Copy the next command in the shell (or *bin/k8s-envls* if you have cloned the entire repository)
```bash
echo "Get namespaces"           && kubectl get namespaces --show-labels | grep -v "^kube"  
echo "\nGet deployments"        && kubectl get deployments -o wide --show-labels -A | grep -v "^kube"
echo "\nGet pods"               && kubectl get pods -o wide --show-labels -A | grep -v "^kube"
echo "\nGet network policies"   && kubectl get networkpolicies.networking.k8s.io -A
```

It should return
<pre>
Get namespaces
NAME              STATUS   AGE    LABELS
default           Active   7d2h   kubernetes.io/metadata.name=default
development       Active   30h    kubernetes.io/metadata.name=development,purpose=development
production        Active   30h    kubernetes.io/metadata.name=production,purpose=production

Get deployments
NAMESPACE     NAME                        READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES                  SELECTOR         LABELS
development   devpod1-deployment          2/2     2            2           Ss     devctd1      capso/capsonet:latest   app=devapp1      deploy=devapp1
production    prodpod1-deployment         2/2     2            2           Ss     prodctd1     capso/capsonet:latest   app=prodapp1     deploy=prodapp1

Get pods
NAMESPACE     NAME                                   READY   STATUS    RESTARTS  AGE  IP             NODE     LABELS
development   devpod1-deployment-7bbf94b866-9zhbm    1/1     Running   0         35m  10.1.83.164    NODE_A   app=devapp1,pod-template-hash=7bbf94b866
development   devpod1-deployment-7bbf94b866-5mq5x    1/1     Running   0         35m  10.1.54.74     NODE_B   app=devapp1,pod-template-hash=7bbf94b866
production    prodpod1-deployment-676dc4948b-xmf5v   1/1     Running   0         53s  10.1.83.165    NODE_A   app=prodapp1,pod-template-hash=676dc4948b
production    prodpod1-deployment-676dc4948b-4d48z   1/1     Running   0         53s  10.1.54.75     NODE_B   app=prodapp1,pod-template-hash=676dc4948b

Get network policies
No resources found
</pre>

As you can see:
- pods **devpod1-deployment-\*** belong to the **devpod1-deployment** and are located in the **development** namespace
- pods **prodpod1-deployment-\*** belong to the **prodpod1-deployment** and are located in the **production** namespace

> **_NOTE:_**  IPs are related to the cluster. You should get something different. 

### Exercice 1 : Testing the connectivity without policies (default behavior) <a name="#Exercice1"></a>

Let kick-off a shell in a pod belong to **devpod1-deployment**. 

```bash
kubectl exec --namespace=development --stdin --tty devpod1-deployment-7bbf94b866-9zhbm  -- /bin/bash
```
> **_NOTE:_** 
>
>We explicit wrote down the --namespace=development, because **devpod1** is not located in the --namespace=default. We will cover more in details *kubectl* in another course.

Let look at the container itself IP address, it shoulb be *10.X.X.X*

```bash
root@devpod1-deployment-7bbf94b866-9zhbm :/# ip addr show eth0
3: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether ae:43:e2:81:04:89 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.1.83.164/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::ac43:e2ff:fe81:489/64 scope link 
       valid_lft forever preferred_lft forever
```
Because each pod have it own DNS A record, both of them will be able to resolv each other.

> **_NOTE:_**  
>
>Pods are assigned a DNS A record in the form of pod-ip-address.my-namespace.pod.cluster.local . For example, a pod with IP 172.12.3.4 in the namespace default with a DNS name of cluster.local would have an entry of the form 172–12–3–4.default.pod.cluster.local. In our cases:
> - 10-X-X-X.development.pod.cluster.local is the FQDN of **devpod1** 
> - 10-Y-Y-Y.production.pod.cluster.local is the FQDN of **prodpod1** 
> Can we change the domain? We will discuss that in more detail in another course.
```bash
root@devpod1-deployment-7bbf94b866-9zhbm:/# nslookup 10-1-83-164.development.pod.cluster.local
Server:		10.152.183.10
Address:	10.152.183.10#53

Name:	10-1-83-164.development.pod.cluster.local
Address: 10.1.83.164

root@devpod1-deployment-7bbf94b866-9zhbm:/# nslookup 10-1-54-75.production.pod.cluster.local
Server:		10.152.183.10
Address:	10.152.183.10#53

Name:	10-1-54-75.production.pod.cluster.local
Address: 10.1.54.75
```

At this point, all pods in **devpod1-deployment** as no restrictions on any pods in **prodpod1-deployment**. The following table illustrates communications (*ping* and *"ssh"*) between the pods **devpod1-deployment-7bbf94b866-9zhbm** and the two pods in **prodpod1-deployment**.

<table>
<tr>
<th>
devpods can ping prodpods :heavy_check_mark:
</th>
<th>
devpods can knock at prodpods ports (like 22) :heavy_check_mark:
</th>
</tr>
<tr>
<td>

```bash
root@devpod1-deployment-7bbf94b866-9zhbm:/# ping -c 2 10-1-54-75.production.pod.cluster.local 
PING 10-1-54-75.production.pod.cluster.local (10.1.54.75) 56(84) bytes of data.
64 bytes from 10.1.54.75 (10.1.54.75): icmp_seq=1 ttl=62 time=0.520 ms
64 bytes from 10.1.54.75 (10.1.54.75): icmp_seq=2 ttl=62 time=0.583 ms

--- 10-1-54-75.production.pod.cluster.local ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
                       ^^^^^^^^^^^^^^^^^^^^^^^^^^


root@devpod1-deployment-7bbf94b866-9zhbm:/# ping -c 2 10-1-83-165.production.pod.cluster.local 
PING 10-1-83-165.production.pod.cluster.local (10.1.83.165) 56(84) bytes of data.
64 bytes from 10.1.83.165 (10.1.83.165): icmp_seq=1 ttl=63 time=0.031 ms
64 bytes from 10.1.83.165 (10.1.83.165): icmp_seq=2 ttl=63 time=0.055 ms

--- 10-1-83-165.production.pod.cluster.local ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
                       ^^^^^^^^^^^^^^^^^^^^^^^^^^
```

</td>

<td>

```bash
root@devpod1-deployment-7bbf94b866-9zhbm:/# nmap 10-1-54-75.production.pod.cluster.local -p 22
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-28 18:20 EST
Nmap scan report for 10-1-54-75.production.pod.cluster.local (10.1.54.75)
Host is up (0.00067s latency).

PORT   STATE SERVICE
22/tcp open  ssh 
^^^^^^^^^^^^^^^^^

Nmap done: 1 IP address (1 host up) scanned in 0.32 seconds

root@devpod1-deployment-7bbf94b866-9zhbm:/# nmap 10-1-83-165.production.pod.cluster.local -p 22
Starting Nmap 7.80 ( https://nmap.org ) at 2023-02-28 18:20 EST
Nmap scan report for 10-1-83-165.production.pod.cluster.local (10.1.83.165)
Host is up (0.000055s latency).

PORT   STATE SERVICE
22/tcp open  ssh 
^^^^^^^^^^^^^^^^^

Nmap done: 1 IP address (1 host up) scanned in 0.22 seconds
```

</td>
</tr>
</table>

## Exercice 2 : Blocking cross namespaces communication <a name="Exercice2"></a>

Namespaces are “hidden” from each other, but they are not fully isolated by default. In the previous exercice, we saw  a pod in one Namespace can talk to a pod in another Namespace. This can often be very useful, for example to have your team’s pods in your Namespace communicate with another team’s pods in another Namespace. Generally, we want to control the communications with *NetworkPolicies*. 

*NetworkPolicies* are an **application-centric** construct which allow you to specify how a "entitie" is allowed to communicate with various network "entities" over the network. *NetworkPolicies* apply to a connection with a pod on one or both ends, and are not relevant to other connections.

The entities that a Pod can communicate with are identified through a combination of the following 3 identifiers:

- Other pods that are allowed (exception: a pod cannot block access to itself) 
- Namespaces that are allowed
- IP blocks (exception: traffic to and from the node where a Pod is running is always allowed, regardless of the IP address of the Pod or the node)

When defining a pod- or namespace- based *NetworkPolicies*, you use a selector to specify what traffic is allowed to and from the Pod(s) that match the selector.

As soon as you have a *NetworkPolicies* that selects a certain group of Pods, those Pods become isolated and reject any traffic that is not allowed by any *NetworkPolicies*.

> **_NOTE:_**
>
> *NetworkPolicies* are additive, so having two *NetworkPolicies* that select the same Pods will result in allowing both defined policies.
> 
> Keep in mind that a *NetworkPolicies*is applied to a particular Namespace and only selects Pods in that particular Namespace.

Let look at the  *learnkubenetes/manifests/course1/netpo-dev.yaml*

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: development-deny-ingress
  namespace: development #<--- namespace to apply the policy
spec:
  podSelector: #<--- on which pods
    matchLabels:
      app: devapp1 #<--- those with this label
  ingress:            
  - from:  #<-- from who
    - namespaceSelector:  #<-- from the namespace
        matchLabels:  
          purpose: development #<-- having this label
```
This policy will limit and accept the incoming connexions only from pods inside the SAME namespace.

The same policy should be apply on the namespace production  *learnkubenetes/manifests/course1/netpo-prod.yaml*

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: production-deny-ingress
  namespace: production #<--- namespace to apply the policy
spec:
  podSelector: #<--- on which pods
    matchLabels:
      app: prodapp1 #<--- those with this label
  ingress:            
  - from:  #<-- from who
    - namespaceSelector:  #<-- from the namespace
        matchLabels:  
          purpose: production #<-- having this label
```
To apply the network policies:

```bash
kubectl create -f ~/learnkubernetes/manifests/course1/netpo-dev.yaml
kubectl create -f ~/learnkubernetes/manifests/course1/netpo-prod.yaml
```

From now, communication are isoled within the namespace. 

The following table illustrates the pods **devpod1-deployment-7bbf94b866-9zhbm** cannot communicate with the two pods in **prodpod1-deployment**.
<table>
<tr>
<th>
devpod1-*.ping(prodpod1-*) = :x:
</th>
<th>
devpod1-*.nmap(prodpod1-*,22) = :x:
</th>
</tr>
<tr>
<td>

```bash
root@devpod1-deployment-7bbf94b866-9zhbm:/# ping -c 2 10-1-54-75.production.pod.cluster.local 
PING 10-1-54-75.production.pod.cluster.local (10.1.54.75) 56(84) bytes of data.

--- 10-1-54-75.production.pod.cluster.local ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1007ms
                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^

root@devpod1-deployment-7bbf94b866-9zhbm:/# ping -c 2 10-1-83-165.production.pod.cluster.local
PING 10-1-83-165.production.pod.cluster.local (10.1.83.165) 56(84) bytes of data.

--- 10-1-83-165.production.pod.cluster.local ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1027ms
                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```
</td>

<td>

```bash
root@devpod1-deployment-7bbf94b866-9zhbm:/# nmap 10-1-54-75.production.pod.cluster.local -Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2023-03-15 13:58 EDT
Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn
Nmap done: 1 IP address (0 hosts up) scanned in 4.52 seconds

root@devpod1-deployment-7bbf94b866-9zhbm:/# nmap 10-1-83-165.production.pod.cluster.local -Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2023-03-15 13:58 EDT
Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn
Nmap done: 1 IP address (0 hosts up) scanned in 4.52 seconds
```

</td>
</tr>
</table>

And vice versa, **prodpod1-deployment-676dc4948b-xmf5v** cannot communicate with the two pods in **devpod1-deployment**.
<table>
<tr>
<th>
prodpod1-*.ping(devpod1-*) = :x:
</th>
<th>
prodpod1-*.nmap(devpod1-*,22) = :x:
</th>
</tr>
<tr>
<td>

```bash
root@devpod1-deployment-7bbf94b866-9zhbm:/# ping -c 2 10-1-54-75.production.pod.cluster.local 
PING 10-1-54-75.production.pod.cluster.local (10.1.54.75) 56(84) bytes of data.

--- 10-1-54-75.production.pod.cluster.local ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1007ms
                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^

root@devpod1-deployment-7bbf94b866-9zhbm:/# ping -c 2 10-1-83-165.production.pod.cluster.local
PING 10-1-83-165.production.pod.cluster.local (10.1.83.165) 56(84) bytes of data.

--- 10-1-83-165.production.pod.cluster.local ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1027ms
                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```
</td>

<td>

```bash
root@devpod1-deployment-7bbf94b866-5mq5x:/# nmap 10-1-54-75.production.pod.cluster.local -Pn
SStarting Nmap 7.80 ( https://nmap.org ) at 2023-03-15 13:58 EDT
Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn
Nmap done: 1 IP address (0 hosts up) scanned in 4.52 seconds

root@devpod1-deployment-7bbf94b866-9zhbm:/# nmap 10-1-83-165.production.pod.cluster.local -Pn
Starting Nmap 7.80 ( https://nmap.org ) at 2023-03-15 13:58 EDT
Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn
Nmap done: 1 IP address (0 hosts up) scanned in 4.52 seconds

```

</td>
</tr>
</table>

## Exercice 3 : Allowing only SSH connexion from the default namespace <a name="Exercice3"></a>

### Step 1: We need to add a name=default label on the --namespace=default

```bash
kubectl label namespace default name=default
```

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: development-accept-ingress-ssh-from-defaultnamespace
  namespace: development #<--- namespace to apply the policy
spec:
  podSelector: {} #<--- on all pods
  ingress:
    - ports: 
        - protocol: TCP
          port: 22  #<---- port use by SSH
      from:
      - namespaceSelector:
          matchLabels:
             name: default  #<---- default namespace
        podSelector: {} #<---- all pods in default namespace.
```