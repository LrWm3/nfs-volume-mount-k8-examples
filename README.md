# nfs-volume-mount-k8

Experiments with kubernetes nfs mounts.

To run the basic version, it is required to have access to `kubectl`, and the `busybox` image must be available.

An nfs share must be running somewhere. The `nfs-server-alpine` and `nfs-client` directories were attempts at using
docker compose to do this. However, when I mounted the `nfs-server-alpine` on my mac, it forced my computer to reboot (!)

I performed the mount like so: 

`sudo mount -t nfs -o vers=4 <server-ip>:/export/share /NFS/Server_Media/`

Eventually I got tired of trying to do everything locally and began using my own public nfs server to test. I used 
`https://double-dutch.ca/public` as a test nfs server afterwards.

I ran some experiments to see how namespaces, persistent volumes, persistent volume claims and pods interact, to try
and assess how best to use PV, PVC and volumes on our project.

## TLDR: Don't use persistent volumes or persistent volume claims; use the nfs volume plugin directly

This works, requires only 5 additional lines per yaml, with no special config or subsituition, and is generally a 
set it and forget it type approach. I found [this post on stackoverflow](https://stackoverflow.com/a/35366775) 
as I was beginning to set up a fourth PV & PVC experiment.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
    - name: busybox
      image: busybox
      command:
        - sleep
        - "3600"
      volumeMounts:
      - mountPath: "/nfsshare"
        name: nfs-jar-volume

  # To add the nfs mount
  volumes:
    - name: nfs-jar-volume
      nfs:
        path: /public
        server: double-dutch.ca
        readOnly: true
```

The rest of this `README.md` details different ways of setting up PV and PVC, all of which are much more complicated
than this. I've left it here for completeness.


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

## e3. bypass all this crap by mounting the nfs volume directly onto a pod/deployment

Lets see if we can simply mount the nfs volume directly, ala [this example](https://stackoverflow.com/a/35366775): 

The pod-file can be found in `kubernetes/e3`, but its actually small enough I can inline it here;

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
    - name: busybox
      image: busybox
      command:
        - sleep
        - "3600"
      volumeMounts:
      - mountPath: "/nfsshare"
        name: nfs-jar-volume
  volumes:
    - name: nfs-jar-volume
      nfs:
        path: /public
        server: double-dutch.ca
        readOnly: true

```

Yeah so according to the linked post, volume plugins can be used directly inline with pods. 

```bash
## To deploy 
kubectl apply -f kubernetes/e3

# To verify
kubectl get pods busybox
kubectl exec -it busybox ls /nfsshare

# To clean-up
kubectl delete -f kubernetes/e1
```

Of course this worked. This is obviously the way to go.
