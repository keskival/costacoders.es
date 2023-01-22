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

Setting up the high availability storage classes was an adventure without a map because all the documentation is either non-existent or out of date and misleading. The MicroK8S add-on for OpenEBS didn't even work, because it had a [bug](https://github.com/canonical/microk8s/issues/3639) which has later been fixed in OpenEBS upstream. Also, I first installed plain OpenEBS, but it quickly became clear what I actually needed was [OpenEBS Jiva](https://github.com/openebs/jiva) for replication. At first I tried it as such but multiple lock-ups made it clear I needed `ReadWriteMany` for some sanity and it didn't support that. First I tried setting up [dynamic-nfs-provisioner](https://github.com/openebs/dynamic-nfs-provisioner) to back the Jiva volumes, but that didn't work, but the other way around, backing the `ReadWriteMany` NFS volumes with Jiva replication, it seems to work.

However, and this is important! I discovered that installing OpenEBS Jiva from the related Helm charts didn't work. It seemed to work, but behind the scenes it [stored all the data on the dynamic NFS provisioner volume pod ephemeral store](https://github.com/openebs/jiva/issues/367). I found out that instead installing the latest MicroK8S community OpenEBS Jiva support works. I never found out what the difference actually is. The latest MicroK8S community plug-in is installed like so:

```
microk8s addons repo add community https://github.com/canonical/microk8s-community-addons --reference main
microk8s enable community/openebs
```

It was extremely important (and a huge hassle) to configure the `startupProbe`s for the Postgres, ElasticSearch and Redis statefulsets. That is because the default timeouts are way too short, and in the best case they leave the pods into eternal restart loop, and in the worst case, they corrupt the Postgres database.

Like so for Redis, similarly for the other services:
```
redis:
    image:
        pullPolicy: IfNotPresent
    global:
        persistence:
            storageClass: openebs-rwx
    master:
        startupProbe:
            enabled: true
            initialDelaySeconds: 40
            periodSeconds: 5
            timeoutSeconds: 5
            failureThreshold: 40
        persistence:
            storageClass: openebs-rwx
    replica:
        startupProbe:
            enabled: true
            initialDelaySeconds: 40
            periodSeconds: 5
            timeoutSeconds: 5
            failureThreshold: 40
        persistence:
            storageClass: openebs-rwx
```

## Restoring Backups

Had to restore back-ups multiple times during this journey, luckily I had made them very recently and restoring them worked as expected. I am only backing up specific resources, not the whole set as [instructed by Mastodon docs](https://docs.joinmastodon.org/admin/backups/), so I need to rebuild indices and refetch external user profiles after restoring backups.

There are no good instructions on rebuilding all the caches and external data after restoring local-only backups without the `system/cache` directory. [The official site recommends backing up the caches as well](https://docs.joinmastodon.org/admin/backups/), although they take a huge amount of space.

I recommend backing up at least the cached emoticons in addition to the local `system` and `assets` contents and the PostgreSQL database, because refreshing the profiles [isn't perfect yet](https://github.com/mastodon/mastodon/issues/17862). However, if you did like I did, you will be able to refresh most of the broken references by running the following `tootctl` commands:

```
tootctl cache clear
tootctl media remove
tootctl emoji purge --remote-only
tootctl accounts refresh --all
tootctl search deploy
```

These will remove all attachments cached from external instances, but don't affect the contents uploaded by your users. External emojis are also similarly removed, although these are just the broken references you would still have lingering in your database, without corresponding files restored from the partial backup.

Once the posts are loaded again by a user, your instance redownloads them from the original instance if they still exist there and the originating instance is up. The same happens for emojis, except for the emojis people use in their profiles, which is a bit of a nuisance at the moment. [There is an open bug about it](https://github.com/mastodon/mastodon/issues/17862).

Anyhow, here's a script I use to back up everything:

```
#!/bin/bash
PGPASSWORD="YOURPGPASSWORD"
MASTODON_SYSTEM_DIR=/opt/mastodon/public/system
BACKUPS_DIR=/mnt/WHEREYOUWANTBACKUPSTOGO/backups
POSTGRES_DUMP_FILENAME=pg_dump_mastodon_backup.sqlc
POSTGRES_DUMP_PATH=/bitnami/postgresql/${POSTGRES_DUMP_FILENAME}

microk8s kubectl exec -it -n mastodon mastodon-postgresql-0 -- env PGPASSWORD="${PGPASSWORD}" pg_dump -U mastodon -d mastodon_production --format=c --file=${POSTGRES_DUMP_PATH}
microk8s kubectl cp -n mastodon mastodon-postgresql-0:${POSTGRES_DUMP_PATH} ${BACKUPS_DIR}/${POSTGRES_DUMP_FILENAME}
PODNAME=$(microk8s kubectl get pod -n mastodon -l app.kubernetes.io/component=web -o name)
PODARRAY=(${PODNAME//\// })
PODNAME=${PODARRAY[1]}
microk8s kubectl exec -n mastodon $PODNAME -- tar zcf - ${MASTODON_SYSTEM_DIR}/accounts > ${BACKUPS_DIR}/system_accounts.tar.gz
microk8s kubectl exec -n mastodon $PODNAME -- tar zcf - ${MASTODON_SYSTEM_DIR}/media_attachments > ${BACKUPS_DIR}/system_media_attachments.tar.gz
microk8s kubectl exec -n mastodon $PODNAME -- tar zcf - ${MASTODON_SYSTEM_DIR}/site_uploads > ${BACKUPS_DIR}/system_site_uploads.tar.gz
microk8s kubectl exec -n mastodon $PODNAME -- tar zcf - ${MASTODON_SYSTEM_DIR}/cache > ${BACKUPS_DIR}/system_cache.tar.gz
microk8s kubectl exec -n mastodon $PODNAME -- env > ${BACKUPS_DIR}/mastodon_environment.txt
```

Restoring it is manual work which involves scaling down web and Sidekiq pods, deploying a custon utility pod which mounts up `system` and `assets` PVCs, `kubectl cp`ing the files there and extracting them in the correct places, and then `pg_restore -c` the database dump to the PostgreSQL, and then scaling everything back up again. Having learned from my experience, I am now backing up the `cache` directory as well.

But I am not backing up ElasticSearch and Redis still, so after the above-mentioned restoration, one still needs to run `tootctl search deploy` to redeploy the search indices.

Author: Tero Keski-Valkama