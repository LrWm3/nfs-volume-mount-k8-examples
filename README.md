# nfs-volume-mount-k8

Experiments with kubernetes nfs mounts.

To run the basic version, it is required to have access to `kubectl`, and the `busybox` image must be available.

An nfs share must be running somewhere. The `nfs-server-alpine` and `nfs-client` directories were attempts at using
docker compose to do this. However, when I mounted the `nfs-server-alpine` on my mac using `, it would reboot (!)

I did this like so: 

`sudo mount -t nfs -o vers=4 <server-ip>:/export/share /NFS/Server_Media/`

I used `https://double-dutch.ca/public` as a test nfs server afterwards.

I ran some experiments to see how namespaces, persistent volumes, persistent volume claims and pods interacted.

## e0: One namespace, one nfs PV, one PVC, one Pod

Testing to see if I can get nfs PV to work at all.

To run the test:

```bash
## To deploy 
kubectl apply -f kubernetes/e0

# To verify
kubectl get pods busybox
kubectl exec -it busybox ls /nfsshare

# To clean-up
kubectl delete -f kubernetes/e0
```

Expected output is busybox is running, and `kubectl exec -it busybox ls /nfsshare` shows contents of nfs.

This test has passed.

## e1: Two namespaces, one PV, two PVC, each namespace has a pod

Testing to see if I can bind two pods in different namespaces to two different PVC, with both PVC attached to the same 
PV.

e1 will deploy two namespaces, one called `novascotia`, the other called `pei`. Pods `busybox` and 
`busybox` will be deployed to these namespaces. A persistent volume with `storageClassName: "nfs-jar"` is created.
Two persistent volume claims, one in `novascotia` and the other in `pei`, both with `storageClassName: "nfs-jar"` are 
created.

The successful condition is if both PVC bind to the same PV, since they are both requesting the same storageClassName,
and are requesting less than the total size of the PV.

```bash
## To deploy 
kubectl apply -f kubernetes/e1

# To verify
kubectl get pods --all-namespaces
kubectl exec -it -n novascotia busybox ls /nfsshare

# To clean-up
kubectl delete -f kubernetes/e1
```

Expected output is `busybox` & `busybox` are running, and 
`kubectl exec -it busybox ls /nfsshare` & `kubectl exec -it busybox ls /nfsshare` both find files.

Actual output of this test found only one persistent volume bound to one of the persistent volume claims. My conclusions 
are:
* PV can only be bound to one PVC at a time
* PVC will not be bound across namespaces by default

## e2: Two namespaces, two PV, two PVC, each namespace has two pods, one storageClassName

Testing to see if multiple pods within the namespace can bind to a PVC. Also, a single storage class name is used, to
see if we could simply create a bunch of PV up-front so that we don't actually need to create a specific storage class
for each PVC which would like to bind to the PV.

```bash
## To deploy 
kubectl apply -f kubernetes/e2

# To verify
kubectl get pods --all-namespaces

# To clean-up
kubectl delete -f kubernetes/e2
```

Expected output is all pods are running in all namespaces.

Actual output is all pods are running in all namespaces.

## e3. e1 again but this time we try and bind to the PVC across namespaces

Lets repeat e1 to see if we can jump the namespace.

```bash
## To deploy 
kubectl apply -f kubernetes/e1

# To verify
kubectl get pods --all-namespaces
kubectl exec -it -n novascotia busybox ls /nfsshare

# To clean-up
kubectl delete -f kubernetes/e1
```

