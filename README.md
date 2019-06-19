# nfs-volume-mount-k8

Experiments with kubernetes nfs mounts.

To run the basic version, it is required to have access to `kubectl`, and the `busybox` image must be available.

An nfs share must be running somewhere. The `nfs-server-alpine` and `nfs-client` directories were attempts at using
docker compose to do this. However, when I mounted the `nfs-server-alpine` on my mac using `, it would reboot (!)

I did this like so: 

`sudo mount -t nfs -o vers=4 <server-ip>:/export/share /NFS/Server_Media/`

I used `https://double-dutch.ca/public` as a test nfs server afterwards.

