# Multi-Node Cluster Installation Guide

In this guide I will create a multi-node Clickhouse cluster in Azure VM. For the basic instruction in Azure, I will not cover
too much in this guide and please refer to Azure official document for more details.

## Before we start

---

We will create a simple cluster, totally 3 shards, each shard per node in Clickhouse cluster.

Create Azure Resource Group
```
az group create --name azure-clickhouse-rg --location eastus
```

Create Virtual Network to host Clickhouse cluster
```
az network vnet create \
  --name clickhouseVNet \
  --resource-group azure-clickhouse-rg \
  --subnet-name default
```

Create Zookeeper VM
```
az vm create -n "zookeeper-vm" -g "azure-clickhouse-rg" \
    --image ubuntults \
    --admin-username sshuser \
    --admin-password "xxx" \
    --authentication-type password \
    --vnet-name clickhouseVNet \
    --subnet default \
    --public-ip-address-allocation static \
    --size Standard_E8ds_v5
```

Create Clickhouse VMs (3 nodes)
```
az vm create -n "clickhouse-vm" -g "azure-clickhouse-rg" \
    --image ubuntults \
    --count 3 \
    --admin-username sshuser \
    --admin-password "xxx" \
    --authentication-type password \
    --vnet-name clickhouseVNet \
    --subnet default \
    --public-ip-address-allocation static \
    --size Standard_E8ds_v5
```

Once the Clickhouse VMs are running, we need to set the `ulimit` values in the configuration file `/etc/security/limits.conf` and `/etc/security/limits.d/20-nproc.conf`
```shell
$ vim /etc/security/limits.conf
*	soft nofile 65536
*	hard nofile 65536
*	soft nproc 131072
*	hard nproc 131072

$ vim /etc/security/limits.d/20-nproc.conf
*	soft nofile 65536
*	hard nofile 65536
*	soft nproc 131072
*	hard nproc 131072
```

## Clickhouse Installation

We need to install the clickhouse distribution in all cluster nodes and please run the following steps in all the nodes you provision.

---

- Download Clickhouse binary locally
```
curl https://clickhouse.com/ | sh
```

- Run `install` command to install and configure Clickhouse

```
sudo ./clickhouse install
```

- At the end of the installation, you will see output to start and login clickhouse
```sql

ClickHouse has been successfully installed.

Start clickhouse-server with:
 sudo clickhouse start

Start clickhouse-client with:
 clickhouse-client --password

```

## Cluster Setup

---

### Zookeeper Installation
Install Java Environment to launch zookeeper cluster
```
sudo apt install default-jdk
```

- Download zookeeper latest stable version
```
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.7.1/apache-zookeeper-3.7.1-bin.tar.gz
```
- Install zookeeper
```
tar zxvf apache-zookeeper-3.7.1-bin.tar.gz &&　mv apache-zookeeper-3.7.1-bin zookeeper

cd zookeeper && mkdir data
```
- Configure

    Create a new zookeeper config file and set up the zookeeper initialization config, for example we will configure the data folder `dataDir` which we created before.
    In this case we will set up a single node zookeeper cluster and you may create a multi-nodes cluster if you prefer to.

    ```
    # The number of milliseconds of each tick
    tickTime=2000
    # The number of ticks that the initial
    # synchronization phase can take
    initLimit=10
    # The number of ticks that can pass between
    # sending a request and getting an acknowledgement
    syncLimit=5
    # the directory where the snapshot is stored.
    # do not use /tmp for storage, /tmp here is just
    # example sakes.
    dataDir=/home/sshuser/zookeeper/data
    # the port at which the clients will connect
    clientPort=2181
    # the maximum number of client connections.
    # increase this if you need to handle more clients
    #maxClientCnxns=60
    #
    # Be sure to read the maintenance section of the
    # administrator guide before turning on autopurge.
    #
    # https://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
    #
    # The number of snapshots to retain in dataDir
    #autopurge.snapRetainCount=3
    # Purge task interval in hours
    # Set to "0" to disable auto purge feature
    #autopurge.purgeInterval=1
    
    ## Metrics Providers
    #
    # https://prometheus.io Metrics Exporter
    #metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
    #metricsProvider.httpHost=0.0.0.0
    #metricsProvider.httpPort=7000
    #metricsProvider.exportJvmInfo=true
    
    # Zookeeper Cluster Communication, since it is a single node cluster and only the local VM will be set here
    10.0.0.4:2888:3888
    ```

### Clickhouse Cluster Configuration

In Clickhouse server config `/etc/clickhouse-server/config.xml`, we have several parts need to modify in existing config.

First, we will create a cluster config in `/etc/clickhouse-server/config.d/metrika.xml`, please make sure it needs to be created in all cluster nodes.

You may also change the cluster identifier as you want, I will put `benchmark` as the cluster identifier here.

```
<yandex>
    <zookeeper-servers>
    <node index="1">
      <host>10.0.0.4</host>
      <port>2181</port>
    </node>
  </zookeeper-servers>
  <remote_servers>
    <benchmark>
      <shard>
        <internal_replication>true</internal_replication>
        <replica>
          <host>10.0.0.5</host>
          <port>9000</port>
        </replica>
      </shard>
      <shard>
        <internal_replication>true</internal_replication>
        <replica>
          <host>10.0.0.6</host>
          <port>9000</port>
        </replica>
      </shard>
      <shard>
        <internal_replication>true</internal_replication>
        <replica>
          <host>10.0.0.7</host>
          <port>9000</port>
        </replica>
      </shard>
    </benchmark>
  </remote_servers>
</yandex>
```

Once done, we need to add a `<include_from>` sector to include the cluster config we just created in `/etc/clickhouse-server/config.xml`

```
<include_from>/etc/clickhouse-server/config.d/metrika.xml</include_from>
```

Second, we need to change the zookeeper setting and input the zookeeper ip you provision.

```
    <zookeeper>
        <node>
            <host>10.0.0.4</host>
            <port>2181</port>
        </node>
    </zookeeper>
```

Third, I suggest that we could point to another data/tmp location to store the clickhouse data/tmp files instead of the existing one in system disk. Please make sure you create a new path in a data disk and change the `<path>` section in `config.xml`

```
<!-- Path to data directory, with trailing slash. -->
    <path>/mnt/clickhouse/data/</path>
    
<!-- Path to temporary data for processing hard queries. -->
    <tmp_path>/mnt/clickhouse/tmp/</tmp_path>
```

## Launch Clickhouse Cluster

---

- Launch zookeeper cluster
```
sshuser@zookeeper-vm:~/zookeeper$ bin/zkServer.sh start
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /home/sshuser/zookeeper/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

```

- Launch Clickhouse cluster, run the same command in all cluster nodes
```
chown -R clickhouse:clickhouse /var/lib/clickhouse
chown -R clickhouse:clickhouse /var/log/clickhouse-server

sudo -u clickhouse clickhouse-server --config-file=/etc/clickhouse-server/config.xml
```

- Login into clickhouse cluster
```shell
clickhouse-client --password 'password' -m
```

- Verify the cluster is running normally
```sql
SELECT * FROM system.clusters WHERE cluster = 'benchmark' FORMAT Vertical;

Row 1:
──────
cluster:                 benchmark
shard_num:               1
shard_weight:            1
replica_num:             1
host_name:               10.0.0.5
host_address:            10.0.0.5
port:                    9000
is_local:                0
user:                    default
default_database:
errors_count:            0
slowdowns_count:         0
estimated_recovery_time: 0

Row 2:
──────
cluster:                 benchmark
shard_num:               2
shard_weight:            1
replica_num:             1
host_name:               10.0.0.6
host_address:            10.0.0.6
port:                    9000
is_local:                1
user:                    default
default_database:
errors_count:            0
slowdowns_count:         0
estimated_recovery_time: 0

Row 3:
──────
cluster:                 benchmark
shard_num:               3
shard_weight:            1
replica_num:             1
host_name:               10.0.0.7
host_address:            10.0.0.7
port:                    9000
is_local:                0
user:                    default
default_database:
errors_count:            0
slowdowns_count:         0
estimated_recovery_time: 0

3 rows in set. Elapsed: 0.002 sec.
```

## Initialize SSB Benchmark

Please refer to [SSB Benchmark](../README.md)

---


## Clean Environment
```
az group delete \
    --name azure-clickhouse-rg \
    --yes
```

## Reference
[Clickhouse Cluster Deployment](https://clickhouse.com/docs/ru/getting-started/tutorial/#cluster-deployment)

[How-to-set-up-a-clickhouse-cluster-with-zookeeper](https://altinity.com/blog/how-to-set-up-a-clickhouse-cluster-with-zookeeper)