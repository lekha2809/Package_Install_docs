Patroni is a cluster manager tool used for customizing and automating deployment and maintenance of high availability PostgreSQL clusters. It is written in Python and uses etcd, Consul, and ZooKeeper as a distributed configuration store for maximum accessibility. 

## Main Components of PostgreSQL cluster

- Patroni provides a template for configuring a highly available PostgreSQL cluster.

- ETCD stores the state of the PostgreSQL cluster.  When any changes in the state of any PostgreSQL node are found, Patroni updates the state change in the ETCD key-value store. ETCD uses this information to elects the master node and keeps the cluster UP and running.
- HAProxy keeps track of changes in the Master/Slave nodes and connects to the appropriate master node when the clients request a connection.
---
| Node | Service  | Ip  |  
|---|---|---|
| Node01 | Etcd, Postgresql  | 10.10.25.8 |  
| Node02 | Etcd, Postgresql  | 10.10.25.9 |  
| Node03 | Etcd, Postgresql, Haproxy | 10.10.25.62 |   
---
### Installing PostgreSQL

> yum install postgresql postgresql-contrib -y

After the installation, you will need to stop the PostgreSQL service on both nodes:

> systemctl stop postgresql

Next, you will need to symlink <code>/usr/lib/postgresql/12/bin/</code> to <code>/usr/sbin</code> because it contains tools used in Patroni.

> ln -s /usr/lib/postgresql/12/bin/* /usr/sbin/

---
### Installing Patroni, ETCD, and HAProxy

Install all the required dependencies:
```bash
yum install python3-pip python3-devel libpq-devel -y
pip3 install --upgrade pip
```

Use the <code>pip</code> command to install the Patroni and other dependencies:
```bash
pip install patroni
pip install python-etcd
pip install psycopg2
```

Install Etcd

> yum install etcd haproxy -y

---
### Configuring ETCD and Patroni

Edit the <code>/etc/etcd/etcd.conf</code> file and add the following configuration:

*Node01*

> vim /etc/etcd/etcd.conf

```bash
[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.25.8:2380,http://localhost:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.25.8:2379,http://localhost:2379"


ETCD_NAME="node1"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.25.8:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.25.8:2379"
ETCD_INITIAL_CLUSTER="node1=http://10.10.25.8:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```
*Node02*

> vim /etc/etcd/etcd.conf

```bash
[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.25.9:2380,http://localhost:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.25.9:2379,http://localhost:2379"


ETCD_NAME="node2"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.25.9:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.25.9:2379"
ETCD_INITIAL_CLUSTER="node1=http://10.10.25.8:2380,node2=http://10.10.25.9:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="existing"
```
*Node03*

> vim /etc/etcd/etcd.conf

```bash
[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.25.62:2380,http://localhost:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.25.62:2379,http://localhost:2379"


ETCD_NAME="node3"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.25.62:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.25.62:2379"
ETCD_INITIAL_CLUSTER="node1=http://10.10.25.8:2380,node2=http://10.10.25.9:2380,node3=http://10.10.25.62:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="existing"
```
Save the file, then restart the <code>etcd service</code> to apply the configuration changes.
```bash
systemctl restart etcd
systemctl status etcd
```
---
Next, you will need to create a <code>patroni.yml</code> file

Add the following configuration:

*Node01*

> vim /etc/patroni/patroni.yml

```bash
scope: postgres
namespace: /pg_cluster/
name: node1

restapi:
  listen: 10.10.25.8:8008            # PostgreSQL node IP address
  connect_address: 10.10.25.8:8008   # PostgreSQL node IP address

etcd:
  hosts: 10.10.25.8:2379,10.10.25.9:2379,10.10.25.62:2379  # ETCD node IP address

bootstrap:
  # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        logging_collector: 'on'
        max_wal_senders: 5
        max_replication_slots: 5
        wal_log_hints: "on"

  # some desired options for 'initdb'
  initdb:  # Note: It needs to be a list (some options need values, others are switches)
  - encoding: UTF8
  - data-checksums

  pg_hba:  # Add following lines to pg_hba.conf after running 'initdb'
  - host replication replicator 127.0.0.1/32 md5
  - host replication replicator 0.0.0.0/0 md5
  - host replication all 10.10.25.8/32 trust
  - host replication all 10.10.25.9/32 trust
  - host replication all 10.10.25.62/32 trust
  - host all all 0.0.0.0/0 md5
#  - hostssl all all 0.0.0.0/0 md5

  # Some additional users users which needs to be created after initializing new cluster
  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 10.10.25.8:5432            # PostgreSQL node IP address
  connect_address: 10.10.25.8:5432   # PostgreSQL node IP address
  data_dir: /data/patroni            # The datadir you created
  bin_dir: /usr/pgsql-14/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: replicator
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

*Node02*

> vim /etc/patroni/patroni.yml

```bash
scope: postgres
namespace: /pg_cluster/
name: node2

restapi:
  listen: 10.10.25.9:8008            # PostgreSQL node IP address
  connect_address: 10.10.25.9:8008   # PostgreSQL node IP address

etcd:
  hosts: 10.10.25.8:2379,10.10.25.9:2379,10.10.25.62:2379  # ETCD node IP address

bootstrap:
  # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        logging_collector: 'on'
        max_wal_senders: 5
        max_replication_slots: 5
        wal_log_hints: "on"

  # some desired options for 'initdb'
  initdb:  # Note: It needs to be a list (some options need values, others are switches)
  - encoding: UTF8
  - data-checksums

  pg_hba:  # Add following lines to pg_hba.conf after running 'initdb'
  - host replication replicator 127.0.0.1/32 md5
  - host replication replicator 0.0.0.0/0 md5
  - host replication all 10.10.25.8/32 trust
  - host replication all 10.10.25.9/32 trust
  - host replication all 10.10.25.62/32 trust
  - host all all 0.0.0.0/0 md5
#  - hostssl all all 0.0.0.0/0 md5

  # Some additional users users which needs to be created after initializing new cluster
  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 10.10.25.9:5432            # PostgreSQL node IP address
  connect_address: 10.10.25.9:5432   # PostgreSQL node IP address
  data_dir: /data/patroni            # The datadir you created
  bin_dir: /usr/pgsql-14/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: replicator
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

*Node03*

> vim /etc/patroni/patroni.yml

```bash
scope: postgres
namespace: /pg_cluster/
name: node3

restapi:
  listen: 10.10.25.62:8008            # PostgreSQL node IP address
  connect_address: 10.10.25.62:8008   # PostgreSQL node IP address

etcd:
  hosts: 10.10.25.8:2379,10.10.25.9:2379,10.10.25.62:2379  # ETCD node IP address

bootstrap:
  # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        logging_collector: 'on'
        max_wal_senders: 5
        max_replication_slots: 5
        wal_log_hints: "on"

  # some desired options for 'initdb'
  initdb:  # Note: It needs to be a list (some options need values, others are switches)
  - encoding: UTF8
  - data-checksums

  pg_hba:  # Add following lines to pg_hba.conf after running 'initdb'
  - host replication replicator 127.0.0.1/32 md5
  - host replication replicator 0.0.0.0/0 md5
  - host replication all 10.10.25.8/32 trust
  - host replication all 10.10.25.9/32 trust
  - host replication all 10.10.25.62/32 trust
  - host all all 0.0.0.0/0 md5
#  - hostssl all all 0.0.0.0/0 md5

  # Some additional users users which needs to be created after initializing new cluster
  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 10.10.25.62:5432            # PostgreSQL node IP address
  connect_address: 10.10.25.62:5432   # PostgreSQL node IP address
  data_dir: /data/patroni            # The datadir you created
  bin_dir: /usr/pgsql-14/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: replicator
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

Save the file and create data directory

```bash
mkdir /data/patroni
chown postgres:postgres /data/patroni
chmod 700 /data/patroni
```

Create <code>systemd</code>

> vim /etc/systemd/system/patroni.service

Add the following configuration:

```bash
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target

[Service]
Type=simple

User=postgres
Group=postgres

# Start the patroni process
ExecStart=/usr/bin/patroni /etc/patroni/patroni.yml

# Send HUP to reload from patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID

# only kill the patroni process, not its children, so it will gracefully stop postgres
KillMode=process

# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=30

# Do not restart the service if it crashes, we want to manually inspect database on failure
Restart=no

[Install]
WantedBy=multi-user.target
```

Save the file, then reload the <code>systemd daemon</code> and start services:

```bash
systemctl daemon-reload
systemctl start patroni
systemctl start postgresql
```

To verify the status of Patroni, run:

> systemctl status patroni

---
### Configuring HAProxy

*Node03*

> vim /etc/haproxy/haproxy.cfg

Remove default configuration and add the following configuration:
```bash
global
    maxconn 100

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen primary
    bind *:5000
    option httpchk /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 node1:5432 maxconn 100 check port 8008
    server node2 node2:5432 maxconn 100 check port 8008
    server node3 node3:5432 maxconn 100 check port 8008

listen standbys
    balance roundrobin
    bind *:5001
    option httpchk /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 node1:5432 maxconn 100 check port 8008
    server node2 node2:5432 maxconn 100 check port 8008
    server node3 node3:5432 maxconn 100 check port 8008
```

Save the file, then restart the <code>HAProxy service</code> to apply the changes:

```bash
systemctl restart haproxy
systemctl status haproxy
```