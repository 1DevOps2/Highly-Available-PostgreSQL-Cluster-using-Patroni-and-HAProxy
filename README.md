# Highly Available PostgreSQL Cluster using Patroni and HAProxy

## Schema
![image](https://github.com/1DevOps2/Highly-Available-PostgreSQL-Cluster-using-Patroni-and-HAProxy/assets/105498424/6e62e60f-4cd9-41e8-a2a6-57c6bb901273)

## Architecture
OS: `Ubuntu 22.04`
Postgres version: `16.2`

| Machines        | IPs           | Roles  |
| ------------- |:-------------:| :-----:|
| pg-node1      | 192.168.xxx.xxx | Postgresql, Patroni |
| pg-node2      | 192.168.xxx.xxx      |  Postgresql, Patroni |
| pg-node3 | 192.168.xxx.xxx    |  Postgresql, Patroni  |
| pg-etcd | 192.168.xxx.xxx  |  etcd  |
| pg-haproxy | 192.168.xxx.xxx   |  HA Proxy  |
		
## Step 1 – Setup pg-node1, pg-node2, pg-node3
```
sudo apt update; sudo apt upgrade -y
sudo apt install net-tools
sudo hostnamectl set-hostname pg-nodeN [Example: pg-nodeN => pg-node1, pg-node2, pg-node3]
exec bash #reflect the new changed hostname
```
### Install and configure PostgreSQL
#### installation:
```
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt install postgresql postgresql-contrib
sudo ln -s /usr/lib/postgresql/16/bin/* /usr/sbin/
```
#### configuring for cluster:
```
sudo su postgres
psql -c "CREATE ROLE test WITH login password 'Secret12test';"
psql -c "CREATE DATABASE testdb WITH OWNER test;"
psql -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'ReplicatorPassword';"
psql -c "ALTER USER postgres PASSWORD 'Postgres123Password';"
```
### Install Python & cluster dependencies
```
sudo apt install python-is-python3
sudo apt -y install python3 python3-pip
sudo apt install python3-testresources
sudo pip3 install --upgrade setuptools
sudo apt-get install libpq-dev
sudo pip3 install psycopg2
sudo pip3 install patroni
sudo pip3 install python-etcd
```
## Step 2 – Setup pg-etcd
```
sudo apt update; sudo apt upgrade -y
sudo hostnamectl set-hostname pg-etcd ;exec bash
sudo apt install net-tools
sudo apt -y install etcd
```
### Configure etcd on the pg-etcd
#### open etcd file:
```
sudo nano /etc/default/etcd
```
#### edit the these lines:
```
ETCD_LISTEN_PEER_URLS="http://192.168.100.32:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.100.32:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.100.32:2380"
ETCD_INITIAL_CLUSTER="default=http://192.168.100.32:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.100.32:2379"
```
#### Restart etcd and check status
```
sudo systemctl restart etcd
sudo systemctl status etcd
```
#### Access
```
curl 192.168.100.32:2380/members
```
#### output
`[{"id":10276657743932975437,"peerURLs":["http://localhost:2380"],"name":"pg-etcd","clientURLs":["http://192.168.100.32:2379"]}]`

## Step 3 – Configure Patroni on the pg-node1, pg-node2, and pg-node3
#### create a file on each node:
```
sudo nano /etc/patroni.yml
```
#### change the ip corresponding to the machine nodes:
```
scope: postgres
namespace: /db/
name: pg-nodeN

restapi:
    listen: <nodeN_ip>:8008
    connect_address: <nodeN_ip>:8008

etcd:
    host: <etcdnode_ip>:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:

  initdb:
  - encoding: UTF8
  - data-checksums

  pg_hba:
  - local all postgres peer
  - local all all peer
  - host all all 0.0.0.0/0 md5
  - host replication replicator 127.0.0.1/32 md5
  - host replication replicator 192.168.100.133/0 md5
  - host replication replicator 192.168.100.3/0 md5
  - host replication replicator 192.168.100.142/0 md5
  - host all all 0.0.0.0/0 md5

  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: <nodeN_ip>:5432
  connect_address: <nodeN_ip>:5432
  data_dir: /data/patroni
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: ReplicatorPassword
    superuser:
      username: postgres
      password: Postgres123Password
  parameters:
      unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```
### Create data directory and set the appropriate permissions + ownership
```
sudo mkdir -p /data/patroni
sudo chown -R postgres:postgres /data/patroni
sudo chmod -R 0750 /data/patroni/
```
### Create patroni service file
open file at `/etc/systemd/system/patroni.service`
```
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.targ
```
### stop the postgresql if you haven't stopped yet, and start the patroni on each node one-by-one
```
sudo systemctl stop postgresql
sudo su
echo 'postgres ALL= NOPASSWD: ALL' >>  /etc/sudoers
su postgres;
sudo systemctl start patroni
sudo systemctl status patroni
```
#### output
```
● patroni.service - High availability PostgreSQL Cluster
Loaded: loaded (/etc/systemd/system/patroni.service; disabled; vendor preset: enabled)
Active: active (running) since Mon 2024-03-18 20:04:11 UTC; 18min ago
Main PID: 22085 (patroni)
Tasks: 15 (limit: 2220)
Memory: 134.5M
CPU: 2.137s
CGroup: /system.slice/patroni.service
├─22085 /usr/bin/python3 /usr/local/bin/patroni /etc/patroni.yml
├─22097 postgres -D /data/patroni --config-file=/data/patroni/postgresql.conf --listen_addresses=192.168.100.133 --port=543> ├─22099 "postgres: postgres: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "> ├─22100 "postgres: postgres: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""> ├─22105 "postgres: postgres: postgres postgres 192.168.100.133(52222) idle" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""> ├─22111 "postgres: postgres: walwriter " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "> ├─22112 "postgres: postgres: autovacuum launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" > ├─22113 "postgres: postgres: logical replication launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" > ├─22177 "postgres: postgres: walsender replicator 192.168.100.142(49110) streaming 0/5000148" "" "" "" "" "" "" "" "" "" ""> └─22192 "postgres: postgres: walsender replicator 192.168.100.3(52512) streaming 0/5000148" "" "" "" "" "" "" "" "" "" "" ">
Mar 18 20:21:23 pg-node1 patroni[22085]: 2024-03-18 20:21:23,339 INFO: no action. I am (pg-node1), the leader with the lock
Mar 18 20:21:33 pg-node1 patroni[22085]: 2024-03-18 20:21:33,339 INFO: no action. I am (pg-node1), the leader with the lock
Mar 18 20:21:43 pg-node1 patroni[22085]: 2024-03-18 20:21:43,339 INFO: no action. I am (pg-node1), the leader with the lock
Mar 18 20:21:53 pg-node1 patroni[22085]: 2024-03-18 20:21:53,339 INFO: no action. I am (pg-node1), the leader with the lock
Mar 18 20:22:03 pg-node1 patroni[22085]: 2024-03-18 20:22:03,339 INFO: no action. I am (pg-node1), the leader with the lock
Mar 18 20:22:13 pg-node1 patroni[22085]: 2024-03-18 20:22:13,339 INFO: no action. I am (pg-node1), the leader with the lock
Mar 18 20:22:23 pg-node1 patroni[22085]: 2024-03-18 20:22:23,339 INFO: no action. I am (pg-node1), the leader with the lock
Mar 18 20:22:33 pg-node1 patroni[22085]: 2024-03-18 20:22:33,340 INFO: no action. I am (pg-node1), the leader with the lock
Mar 18 20:22:43 pg-node1 patroni[22085]: 2024-03-18 20:22:43,339 INFO: no action. I am (pg-node1), the leader with the lock
Mar 18 20:22:53 pg-node1 patroni[22085]: 2024-03-18 20:22:53,341 INFO: no action. I am (pg-node1), the leader with the lock
```
## Step 4 – Install and Configuring HA Proxy on pg-haproxy node
### Installation
```
sudo apt update
sudo hostnamectl set-hostname pg-haproxy
exec bash
sudo apt install -y net-tools  haproxy
```
### Configuring
open file at `/etc/haproxy/haproxy.cfg`
```
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon
        maxconn 100

        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres
    bind *:5000
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 192.168.100.133:5432 maxconn 100 check port 8008
    server node2 192.168.100.3:5432 maxconn 100 check port 8008
    server node3 192.168.100.142:5432 maxconn 100 check port 8008
```
### Restart and check status of haproxy
```
sudo systemctl restart haproxy
sudo systemctl status haproxy
```
#### output
```
● haproxy.service - HAProxy Load Balancer
Loaded: loaded (/lib/systemd/system/haproxy.service; enabled; vendor preset: enabled)
Active: active (running) since Mon 2024-03-18 21:51:20 UTC; 3min 37s ago
Docs: man:haproxy(1)
file:/usr/share/doc/haproxy/configuration.txt.gz
Process: 3308 ExecStartPre=/usr/sbin/haproxy -Ws -f $CONFIG -c -q $EXTRAOPTS (code=exited, status=0/SUCCESS)
Main PID: 3310 (haproxy)
Tasks: 2 (limit: 2220)
Memory: 3.1M
CPU: 87ms
CGroup: /system.slice/haproxy.service
├─3310 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock
└─3312 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock

Mar 18 21:51:20 pg-haproxy haproxy[3213]: [NOTICE] (3213) : path to executable is /usr/sbin/haproxy
Mar 18 21:51:20 pg-haproxy haproxy[3213]: [ALERT] (3213) : Current worker #1 (3215) exited with code 143 (Terminated)
Mar 18 21:51:20 pg-haproxy haproxy[3213]: [WARNING] (3213) : All workers exited. Exiting... (0)
Mar 18 21:51:20 pg-haproxy systemd[1]: haproxy.service: Deactivated successfully.
Mar 18 21:51:20 pg-haproxy systemd[1]: Stopped HAProxy Load Balancer.
Mar 18 21:51:20 pg-haproxy systemd[1]: Starting HAProxy Load Balancer...
Mar 18 21:51:20 pg-haproxy haproxy[3310]: [NOTICE] (3310) : New worker #1 (3312) forked
Mar 18 21:51:20 pg-haproxy systemd[1]: Started HAProxy Load Balancer.
Mar 18 21:51:21 pg-haproxy haproxy[3312]: [WARNING] (3312) : Server postgres/node2 is DOWN, reason: Layer7 wrong status, code: 503, inf>Mar 18 21:51:22 pg-haproxy haproxy[3312]: [WARNING] (3312) : Server postgres/node3 is DOWN, reason: Layer7 wrong status, code: 503, inf>
```
#### Access
```
curl 192.168.100.65:7000
```
## Step 5 – Testing High Availability Cluster Setup of PostgreSQL
stop the patroni on pg-node1:
```
sudo systemctl stop patroni
```
In this case, the another Postgres server is promoted to master. 
Restarting `pg-node1`
```
sudo systemctl start patroni
```
now `pg-node1` is replica and following leader.
to list the cluster role status:
```
patronictl -c /etc/patroni.yml list
```
## References
`https://elma365.com/en//help/configure-postgresql.html`

`https://jfrog.com/community/devops/highly-available-postgresql-cluster-using-patroni-and-haproxy/`






