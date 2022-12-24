---
title: "I'm self-hosting my own Mastodon server on a Kubernetes in the home network"
date: 2022-12-24
draft: false
author: Tero Keski-Valkama
---

## Mastodon on Kubernetes

Merry Yule! I finally managed to make my home Kubernetes cluster highly available as I now have three nodes running, one traditional desktop PC and two Intel NUCs.

I learned that making the cluster highly available is extremely important if you have more than one node or non-trivial persistence. And luckily I already knew that backups are important as well.

If you have one node, most things just tend to work. With two nodes you pretty much need to use node affinities to put work loads with persistence to specific nodes because hostpath volumes are node specific, and using anything more complicated for two node situation generally makes no sense.

With three or more nodes things become more complicated. It becomes too much of a hassle to manually assign workloads, and it's better to have persistence highly available, so you start looking into things like MetalLB to have a pool of external IP addresses for the cluster load balancer so that ingress connections can be received by any node, and highly available persistence like OpenEBS Jiva which replicates the volumes across different nodes using iSCSI.

You will also need to enable the cluster control plane [high availability](https://microk8s.io/docs/high-availability) at this point, for MicroK8S I use it's a process involving `microk8s enable ha-cluster` and rejoining with nodes until the cluster is highly available.

I learned the hard way that before letting pods with persistence requirements migrate freely, it is very important to enable high availability for the cluster control plane. This is because you will get into a situation where a node goes down, pods with iSCSI persistence (which only allows ReadWriteOnce) will migrate around, and as they do that, they will leave behind orphaned containers with mounts, which prevent new pods from starting up with errors about volume having been already mounted somewhere else.

For workloads which require `ReadWriteMany` for highly available persistent volumes like Mastodon I'm running you can put [dynamic-nfs-provisioner](https://github.com/openebs/dynamic-nfs-provisioner), `openebs-rwx` storage class on top of [OpenEBS Jiva](https://github.com/openebs/jiva) replicated volumes, on top of OpenEBS hostpath volumes.

## Issues and Workarounds

Setting up Mastodon to a local MicroK8S Kubernetes cluster was a journey of many learnings.

First, with two nodes it became apparent that the default pod affinities weren't correctly set in [Mastodon charts](https://github.com/mastodon/chart/pull/13). This caused so many lockdowns and outages that it's not even funny. The pods which use the same persistent volume claims need to be co-located on the same node when the volume is attached as `ReadWriteOnce`.

Setting up a high availability [MetalLB](https://microk8s.io/docs/addon-metallb) load balancer was pretty straight-forward actually.

Setting up the high availability storage classes was an adventure without a map because all the documentation is either non-existent or out of date and misleading. The MicroK8S add-on for OpenEBS didn't even work, because it had a [bug](https://github.com/canonical/microk8s/issues/3639) which has later been fixed in OpenEBS upstream. Also, I first installed plain OpenEBS, but it quickly became clear what I actually needed was [OpenEBS Jiva](https://github.com/openebs/jiva) for replication. At first I tried it as such but multiple lock-ups made it clear I needed `ReadWriteMany` for some sanity and it didn't support that. First I tried setting up [dynamic-nfs-provisioner](https://github.com/openebs/dynamic-nfs-provisioner) to back the Jiva volumes, but that didn't work, but the other way around, backing the `ReadWriteMany` NFS volumes with Jiva replication, it seems to work so far.

Had to restore back-ups multiple times during this journey, luckily I had made them very recently and restoring them worked as expected. I am only backing up specific resources, not the whole set as [instructed by Mastodon docs](https://docs.joinmastodon.org/admin/backups/), so I need to rebuild indices and refetch external user profiles after restoring backups.

Author: Tero Keski-Valkama