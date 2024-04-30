# 搭建Etcd开发环境

<show-structure depth="3"/>

通过`Goreman`搭建单机3实例的集群环境，对集群中一个实例进行debug。

## 1. 源码编译

```Shell
git clone https://github.com/etcd-io/etcd.git
cd etcd
git checkout -b debug-v3.5.6 v3.5.6
make build
goreman -f ./Procfile start
```

## 2. 启动脚本

### 2.1 debug-etcd.sh

```Shell
#!/bin/bash

PWD=/home/kangxiaoning/cmd

if ps -ef | grep -v grep | grep -q 12379; then
    echo "etcd is running under goreman"
    goreman run stop etcd1
else
    echo "etcd is not running under goreman"
fi

if ps -ef|grep -v grep|grep dlv|grep -q bin/etcd; then
    echo "etcd is running under dlv"
    ps -ef|grep -v grep|grep dlv|grep -q bin/etcd|awk '{print $2}'|xargs sudo kill -9
else
    echo "etcd is not running under dlv"
fi

sleep 1

sudo pwd
${PWD}/lib/start-etcd.sh
```

### 2.2 start-etcd.sh

```Shell
#!/bin/bash

PWD=/home/kangxiaoning/workspace/etcd
cd ${PWD}
dlv exec bin/etcd --headless --listen=:22355 --api-version=2 --accept-multiclient -- --name infra1 --listen-client-urls http://127.0.0.1:12379 --advertise-client-urls http://127.0.0.1:12379 --listen-peer-urls http:
//127.0.0.1:12380 --initial-advertise-peer-urls http://127.0.0.1:12380 --initial-cluster-token etcd-cluster-1 --initial-cluster infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32
380 --initial-cluster-state new --enable-pprof --logger=zap --log-outputs=stderr
```

## 3. 使用方法

运行`./debug-etcd.sh`脚本，启动`dlv`进程，然后通过GoLand连接即可。

```Shell
./debug-etcd.sh
```

<procedure>
<img src="debug-etcd.png"  thumbnail="true"/>
</procedure>

```Shell
kangxiaoning@localhost:~/workspace/etcd$ ps -ef|grep bin/etcd|grep -v grep
kangxia+ 2113025 2113013  0 22:38 pts/0    00:00:00 /bin/sh -c bin/etcd --name infra3 --listen-client-urls http://127.0.0.1:32379 --advertise-client-urls http://127.0.0.1:32379 --listen-peer-urls http://127.0.0.1:32380 --initial-advertise-peer-urls http://127.0.0.1:32380 --initial-cluster-token etcd-cluster-1 --initial-cluster 'infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32380' --initial-cluster-state new --enable-pprof --logger=zap --log-outputs=stderr
kangxia+ 2113027 2113025  1 22:38 pts/0    00:01:19 bin/etcd --name infra3 --listen-client-urls http://127.0.0.1:32379 --advertise-client-urls http://127.0.0.1:32379 --listen-peer-urls http://127.0.0.1:32380 --initial-advertise-peer-urls http://127.0.0.1:32380 --initial-cluster-token etcd-cluster-1 --initial-cluster infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32380 --initial-cluster-state new --enable-pprof --logger=zap --log-outputs=stderr
kangxia+ 2124754 2113013  0 22:46 pts/0    00:00:00 /bin/sh -c bin/etcd --name infra2 --listen-client-urls http://127.0.0.1:22379 --advertise-client-urls http://127.0.0.1:22379 --listen-peer-urls http://127.0.0.1:22380 --initial-advertise-peer-urls http://127.0.0.1:22380 --initial-cluster-token etcd-cluster-1 --initial-cluster 'infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32380' --initial-cluster-state new --enable-pprof --logger=zap --log-outputs=stderr
kangxia+ 2124755 2124754  1 22:46 pts/0    00:00:56 bin/etcd --name infra2 --listen-client-urls http://127.0.0.1:22379 --advertise-client-urls http://127.0.0.1:22379 --listen-peer-urls http://127.0.0.1:22380 --initial-advertise-peer-urls http://127.0.0.1:22380 --initial-cluster-token etcd-cluster-1 --initial-cluster infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32380 --initial-cluster-state new --enable-pprof --logger=zap --log-outputs=stderr
kangxia+ 2189610 2189609  0 23:30 pts/7    00:00:01 dlv exec bin/etcd --headless --listen=:22355 --api-version=2 --accept-multiclient -- --name infra1 --listen-client-urls http://127.0.0.1:12379 --advertise-client-urls http://127.0.0.1:12379 --listen-peer-urls http://127.0.0.1:12380 --initial-advertise-peer-urls http://127.0.0.1:12380 --initial-cluster-token etcd-cluster-1 --initial-cluster infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32380 --initial-cluster-state new --enable-pprof --logger=zap --log-outputs=stderr
kangxia+ 2189618 2189610  0 23:30 pts/7    00:00:10 /home/kangxiaoning/workspace/etcd/bin/etcd --name infra1 --listen-client-urls http://127.0.0.1:12379 --advertise-client-urls http://127.0.0.1:12379 --listen-peer-urls http://127.0.0.1:12380 --initial-advertise-peer-urls http://127.0.0.1:12380 --initial-cluster-token etcd-cluster-1 --initial-cluster infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32380 --initial-cluster-state new --enable-pprof --logger=zap --log-outputs=stderr
kangxiaoning@localhost:~/workspace/etcd$
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --write-out=table --endpoints=localhost:12379,localhost:22379,localhost:32379 member list
+------------------+---------+--------+------------------------+------------------------+------------+
|        ID        | STATUS  |  NAME  |       PEER ADDRS       |      CLIENT ADDRS      | IS LEARNER |
+------------------+---------+--------+------------------------+------------------------+------------+
| 8211f1d0f64f3269 | started | infra1 | http://127.0.0.1:12380 | http://127.0.0.1:12379 |      false |
| 91bc3c398fb3c146 | started | infra2 | http://127.0.0.1:22380 | http://127.0.0.1:22379 |      false |
| fd422379fda50e48 | started | infra3 | http://127.0.0.1:32380 | http://127.0.0.1:32379 |      false |
+------------------+---------+--------+------------------------+------------------------+------------+
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --write-out=table --endpoints=localhost:12379,localhost:22379,localhost:32379 endpoint status
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| localhost:12379 | 8211f1d0f64f3269 |   3.5.6 |   20 kB |     false |      false |         2 |         13 |                 13 |        |
| localhost:22379 | 91bc3c398fb3c146 |   3.5.6 |   20 kB |     false |      false |         2 |         13 |                 13 |        |
| localhost:32379 | fd422379fda50e48 |   3.5.6 |   25 kB |      true |      false |         2 |         13 |                 13 |        |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
kangxiaoning@localhost:~/workspace/etcd$ 
```