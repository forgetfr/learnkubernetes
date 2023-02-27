# Network in kubernetes

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

### Step 3 : valide the lab
Copy the next command in the shell
```bash
echo "Get namespaces" && kubectl get namespaces --show-labels | grep -v "^kube" && \ 
echo "\nGet pods"     && kubectl get pods -o wide --show-labels -A | grep -v "^kube"
```

It should return
<pre>
Get namespaces
NAME              STATUS   AGE     LABELS
default           Active   5d22h   kubernetes.io/metadata.name=default
development       Active   162m    kubernetes.io/metadata.name=development,name=development
production        Active   162m    kubernetes.io/metadata.name=production,name=production

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

In the shell, let kick-off a shell in the container **devprod1**. 

```bash
kubectl exec --namespace=development --stdin --tty devprod1 -- /bin/bash
```

- This container is based on Ubuntu and provide us a shell with some networks tools like *ping, ip, nmap, netstat* and a SSH server and client.
- As you can see, we explicit wrote down the namespace, because *kubectl* is setup by default to execute all the command in the default namespace. **devprod1** is not located in the default namespace. We will cover that more in detail in another course.
- You will see that automatically, the prompt of the shell will display the user and the FQDN of the container (which is it name) in the format *user@FQDN*. In our case, it will be *root@devprod1*

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

> **_NOTE:_**  If DNS is enabled (our case), pods are assigned a DNS A record in the form of pod-ip-address.my-namespace.pod.cluster.local . For example, a pod with IP 172.12.3.4 in the namespace default with a DNS name of cluster.local would have an entry of the form 172–12–3–4.default.pod.cluster.local. In our cases:
> - 10-X-X-X.development.pod.cluster.local is the FQDN of *devpod1* 
> - 10-Y-Y-Y.production.pod.cluster.local is the FQDN of *prodpod1* 
```bash
root@devpod1:/# ip addr show eth0
3: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether ae:43:e2:81:04:89 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.X.X.X/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::ac43:e2ff:fe81:489/64 scope link 
       valid_lft forever preferred_lft forever
```


