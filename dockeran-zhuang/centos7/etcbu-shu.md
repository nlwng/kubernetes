1.下载地址--二进制文件安装

```
#etcd
https://github.com/coreos/etcd/releases/

[root@k8s-master ~]# wget https://github.com/coreos/etcd/releases/download/v3.3.2/etcd-v3.3.2-linux-amd64.tar.gz

[root@k8s-master ~]# tar zxf etcd-v3.3.2-linux-amd64.tar.gz 
[root@k8s-master ~]# mkdir -p /opt/etcd/bin /opt/etcd/conf /opt/etcd/default.etcd
[root@k8s-master ~]# cp /root/etcd-v3.3.2-linux-amd64/etc* /opt/etcd/bin/
[root@k8s-master ~]# useradd -M -s /sbin/nologin etcd
[root@k8s-master ~]# chown -R etcd.etcd /opt/etcd
```

vim /opt/etcd/conf/etcd.conf

```
# [member]
ETCD_NAME=default
ETCD_DATA_DIR="/opt/etcd/default.etcd"
# ETCD_WAL_DIR=""
# ETCD_SNAPSHOT_COUNT="10000"
# ETCD_HEARTBEAT_INTERVAL="100"
# ETCD_ELECTION_TIMEOUT="1000"
# ETCD_LISTEN_PEER_URLS="http://localhost:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
# ETCD_MAX_SNAPSHOTS="5"
# ETCD_MAX_WALS="5"
# ETCD_CORS=""
#
# [cluster]
# ETCD_INITIAL_ADVERTISE_PEER_URLS="http://localhost:2380"
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
# ETCD_INITIAL_CLUSTER="default=http://localhost:2380"
# ETCD_INITIAL_CLUSTER_STATE="new"
# ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
# ETCD_DISCOVERY=""
# ETCD_DISCOVERY_SRV=""
# ETCD_DISCOVERY_FALLBACK="proxy"
# ETCD_DISCOVERY_PROXY=""
#
# [proxy]
# ETCD_PROXY="off"
# ETCD_PROXY_FAILURE_WAIT="5000"
# ETCD_PROXY_REFRESH_INTERVAL="30000"
# ETCD_PROXY_DIAL_TIMEOUT="1000"
# ETCD_PROXY_WRITE_TIMEOUT="5000"
# ETCD_PROXY_READ_TIMEOUT="0"
#
# [security]
# ETCD_CERT_FILE=""
# ETCD_KEY_FILE=""
# ETCD_CLIENT_CERT_AUTH="false"
# ETCD_TRUSTED_CA_FILE=""
# ETCD_PEER_CERT_FILE=""
# ETCD_PEER_KEY_FILE=""
# ETCD_PEER_CLIENT_CERT_AUTH="false"
# ETCD_PEER_TRUSTED_CA_FILE=""
# [logging]
# ETCD_DEBUG="false"
# examples for -log-package-levels etcdserver=WARNING,security=DEBUG
# ETCD_LOG_PACKAGE_LEVELS=""
```

vim /usr/lib/systemd/system/etcd.service

```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/opt/etcd/
EnvironmentFile=-/opt/etcd/conf/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /opt/etcd/bin/etcd --name=\"${ETCD_NAME}\" --data-dir=\"${ETCD_DATA_DIR}\" --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\""
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

启动并检验

```
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd
systemctl status etcd
/opt/etcd/bin/etcdctl ls
```



