# Network in kubernetes

## Summary: 
By default, in kubernetes everything is opened. This course will let you understand the steps you need to isolated communication within a namespace.

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
Copy the next command
```bash
echo "Get namespaces" && kubectl get namespaces --show-labels | grep -v "^kube" && \ 
echo "\nGet pods"     && kubectl get pods -o wide --show-labels -A | grep -v "^kube"
```

It should return
<pre>
Get namespaces
NAME              STATUS   AGE     LABELS
default           Active   5d22h   kubernetes.io/metadata.name=default
<span style="color:blue">development</span>       Active   162m    kubernetes.io/metadata.name=development,name=development
<span style="color:green">production</span>         Active   162m    kubernetes.io/metadata.name=production,name=production

Get pods
NAMESPACE     NAME         READY   STATUS    RESTARTS        AGE     IP         NODE        NO...TES   LABELS
<span style="color:blue">development</span>    devpod1      1/1     Running   0               148m    10.X.X.X   FQDN-node1  ...        app=devpod1
<span style="color:green">production</span>     prodpod1     1/1     Running   0               148m    10.Y.Y.Y   FQDN-node2  ...        app=prodpod1

</pre>