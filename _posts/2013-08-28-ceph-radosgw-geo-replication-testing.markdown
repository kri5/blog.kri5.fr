---
layout: post
status: publish
published: true
title: Ceph radosgw geo-replication testing
tags:
- Development
- Ceph
- FOSS
comments: []
---
Ceph is developing a new feature, which is asynchronous geo-replication for disaster recovery purpose (or other use case if you find some).

By now, Ceph is synchronous, ie when you write data into your cluster, the cluster will tell you your datas has been writing if all the replicas of your data has been written.

In the case of geo-replication, we would have a bottleneck here : network throughput and latency.
We would prefer an async replication, with eventual consistency.

This feature will only be available by now with the Ceph Object Store : the Rados Gateway.

For architectural purpose and deadlines purposes, it would have been much more complicated to do it at the Rados level (might come some day).

The main idea is to expose API from a master cluster radosgw, so that an agent will be able to retrieve metadata and data changes, and post it back to the slave region.

To try this feature, which is under development, you can do it locally with a bit of setup.

First you need to get the Ceph code and compile it, as described [here]({% post_url 2013-05-29-ceph-starting-guide %}).

When you got this, you will use vstart to launch multiple clusters locally.

In fact i modified the vstart.sh (which is upstream now so you should have the same) so you will be able to specify an working directory for each cluster.

I recommend you to edit the vstart.sh though, just to add the --system option on the user creation.

You would need to do a little script to wrap all together.

First create your clusters directories

    mkdir -p $PWD/cluster-master/dev
    mkdir -p $PWD/cluster-master/out
    mkdir -p $PWD/cluster-slave/dev
    mkdir -p $PWD/cluster-slave/out

then you will need to export a few variables for later use

    export PYTHONPATH=./pybind
    export LD_LIBRARY_PATH=.libs

Now you can launch your two clusters locally!

    CEPH_DIR="$PWD/cluster-master" CEPH_PORT=6789 CEPH_RGW_PORT=8000 $PWD/vstart.sh -r -l -n -x mon osd
    CEPH_DIR="$PWD/cluster-slave"  CEPH_PORT=6989 CEPH_RGW_PORT=8001 $PWD/vstart.sh -r -l -n -x mon osd

Two ceph clusters are now up and running.
It's now time to setup regions and zone.

We will have 1 zone per region, and each zone will enable the logging of operations.

    $PWD/radosgw-admin -c $PWD/cluster-master/ceph.conf region set < master.region
    $PWD/radosgw-admin -c $PWD/cluster-master/ceph.conf region default --rgw-region=master
    $PWD/radosgw-admin -c $PWD/cluster-master/ceph.conf zone set --rgw-region=master < master.zone
    $PWD/radosgw-admin -c $PWD/cluster-master/ceph.conf region set < slave.region
    $PWD/radosgw-admin -c $PWD/cluster-master/ceph.conf region default --rgw-region=slave
    $PWD/radosgw-admin -c $PWD/cluster-master/ceph.conf zone set --rgw-region=slave < slave.zone

Now it's time to mess with pools!
We will delete unnecessary buckets and create mandatory ones.

    $PWD/rados -c $PWD/cluster-master/ceph.conf -p .rgw.root rm region_info.default
    $PWD/rados -c $PWD/cluster-master/ceph.conf -p .rgw.root rm zone_info.default
    $PWD/rados -c $PWD/cluster-slave/ceph.conf  -p .rgw.root rm region_info.default
    $PWD/rados -c $PWD/cluster-slave/ceph.conf  -p .rgw.root rm zone_info.default
    $PWD/rados -c $PWD/cluster-master/ceph.conf mkpool .rgw.buckets
    $PWD/rados -c $PWD/cluster-master/ceph.conf mkpool .rgw.buckets.index
    $PWD/rados -c $PWD/cluster-master/ceph.conf mkpool .log
    $PWD/rados -c $PWD/cluster-slave/ceph.conf  mkpool .rgw.buckets
    $PWD/rados -c $PWD/cluster-slave/ceph.conf  mkpool .rgw.buckets.index
    $PWD/rados -c $PWD/cluster-slave/ceph.conf  mkpool .log

To remove a warning, because we have no regionmap so far, we can use the following commands

    $PWD/radosgw-admin -c $PWD/cluster-master/ceph.conf regionmap update
    $PWD/radosgw-admin -c $PWD/cluster-slave/ceph.conf  regionmap update

Because we have updated the regions and zone, but mostly because we enabled logging on theses ones, we need to restart the radosgws.

    killall -w lt-radosgw
    $PWD/radosgw -n client.radosgw.rgw0 -c $PWD/cluster-master/ceph.conf
    $PWD/radosgw -n client.radosgw.rgw0 -c $PWD/cluster-slave/ceph.conf

We are now ready to go!

You can now checkout the radosgw-agent repository, and use the radosgw-agent to sync your master-cluster with the slave cluster. (By now, only metadata are synced)

    ./setup.py
    ./radosgw-agent -c conf.yaml --sync-scope=full

Ps: Here comes the few files i refer to before

`master.region`:

    {
        "name": "master",
        "api_name": "master_api",
        "is_master": "true",
        "endpoints": [ "http:\/\/localhost:8000\/" ],
        "master_zone": "master-1",
        "zones": [{
            "name": "master-1",
            "endpoints": [ "http:\/\/localhost:8000\/" ],
            "log_meta" : "true",
            "log_data" : "true"
        }]
    }

`master.zone`:

    {
        "domain_root": ".rgw",
        "control_pool": ".rgw.control",
        "gc_pool": ".rgw.gc",
        "log_pool": ".log",
        "intent_log_pool": ".intent-log",
        "usage_log_pool": ".usage",
        "user_keys_pool": ".users",
        "user_email_pool": ".users.email",
        "user_swift_pool": ".users.swift",
        "user_uid_pool": ".users.uid",
        "system_key":
        {
            "user": "tester",
            "access_key": "0555b35654ad1656d804",
            "secret_key": "h7GhxuBLTrlhVUyxSPUKUV8r/2EI4ngqJxD7iBdBYLhwluN30JaT3Q=="
        }
    }

`slave.region`:

    {
        "name": "slave",
        "api_name": "slave_api",
        "is_master": "true",
        "endpoints": [ "http:\/\/localhost:8001\/" ],
        "master_zone": "slave-1",
        "zones": [{
            "name": "slave-1",
            "endpoints": [ "http:\/\/localhost:8001\/" ],
            "log_meta" : "true",
            "log_data" : "true"
        }]
    }

`slave.zone`:

    {
        "domain_root": ".rgw",
        "control_pool": ".rgw.control",
        "gc_pool": ".rgw.gc",
        "log_pool": ".log",
        "intent_log_pool": ".intent-log",
        "usage_log_pool": ".usage",
        "user_keys_pool": ".users",
        "user_email_pool": ".users.email",
        "user_swift_pool": ".users.swift",
        "user_uid_pool": ".users.uid",
        "system_key":
        {
            "user": "tester",
            "access_key": "0555b35654ad1656d804",
            "secret_key": "h7GhxuBLTrlhVUyxSPUKUV8r/2EI4ngqJxD7iBdBYLhwluN30JaT3Q=="
        }
    }

`conf.yaml`:

    src_access_key : "0555b35654ad1656d804"
    src_secret_key : "h7GhxuBLTrlhVUyxSPUKUV8r/2EI4ngqJxD7iBdBYLhwluN30JaT3Q=="
    src_host : "localhost"
    src_port : 8000
    src_zone : "master-1"
    dest_access_key : "0555b35654ad1656d804"
    dest_secret_key : "h7GhxuBLTrlhVUyxSPUKUV8r/2EI4ngqJxD7iBdBYLhwluN30JaT3Q=="
    dest_host : "localhost"
    dest_port : 8001
    dest_zone : "slave-1"
    daemon_id : 1337
    sync_scope : "full"

Note that i use the same `access_key` and `secret_key` that are in `vstart.sh`, this is only for convinience and testing purpose.
