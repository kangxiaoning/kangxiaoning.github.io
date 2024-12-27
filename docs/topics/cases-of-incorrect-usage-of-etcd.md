# Etcd错误使用案例

Start typing here...

## 只连接1个Etcd实例

如下架构中，Master节点的APIServer只访问本地Etcd，当该节点有I/O性能问题导致Etcd写入异常，但是kube-apiserver、kube-controller-manager还能工作时，会导致连接到该节点的Node出现NotReady。

原因是连接到该节点的kubelet上报的状态**丢失**，但是其它2个节点组成的集群是正常的，控制面还能正常工作，因此可以将Node标记为NotReady。

<procedure>
<img src="kubernetes-bad-arch.svg"  thumbnail="true"/>
</procedure>

### 1. 集群正常运行

```Bash
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --write-out=table --endpoints=localhost:12379,localhost:22379,localhost:32379 endpoint status
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| localhost:12379 | 8211f1d0f64f3269 |   3.5.6 |   25 kB |     false |      false |         5 |         56 |                 56 |        |
| localhost:22379 | 91bc3c398fb3c146 |   3.5.6 |   20 kB |      true |      false |         5 |         56 |                 56 |        |
| localhost:32379 | fd422379fda50e48 |   3.5.6 |   25 kB |     false |      false |         5 |         56 |                 56 |        |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
kangxiaoning@localhost:~/workspace/etcd$
```

下面是对这个场景的简单模拟。

### 2. 单个节点IO异常

通过断点模拟IO异常，写入不了。

<procedure>
<img src="etcd-io-pause.png"  thumbnail="true"/>
</procedure>

此时集群整体还可用。

```Bash
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --write-out=table --endpoints=localhost:12379,localhost:22379,localhost:32379 endpoint status
{"level":"warn","ts":"2024-12-27T11:29:14.486+0800","logger":"etcd-client","caller":"v3@v3.5.6/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0x40002fefc0/localhost:12379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
Failed to get the status of endpoint localhost:12379 (context deadline exceeded)
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| localhost:22379 | 91bc3c398fb3c146 |   3.5.6 |   20 kB |      true |      false |         5 |         56 |                 56 |        |
| localhost:32379 | fd422379fda50e48 |   3.5.6 |   25 kB |     false |      false |         5 |         56 |                 56 |        |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
kangxiaoning@localhost:~/workspace/etcd$
```

### 3. 向异常节点写入数据

向异常节点写入数据会失败。

```Bash
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --endpoints=localhost:12379,localhost:22379,localhost:32379 get "hello"
kangxiaoning@localhost:~/workspace/etcd$ date;time bin/etcdctl --endpoints=localhost:12379 put "hello" "world";date
Fri Dec 27 11:34:24 CST 2024
{"level":"warn","ts":"2024-12-27T11:34:29.585+0800","logger":"etcd-client","caller":"v3@v3.5.6/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0x40004de1c0/localhost:12379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
Error: context deadline exceeded

real	0m5.037s
user	0m0.028s
sys	0m0.016s
Fri Dec 27 11:34:29 CST 2024
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --endpoints=localhost:12379,localhost:22379,localhost:32379 get "hello"
kangxiaoning@localhost:~/workspace/etcd$
```

### 4. 向集群写入数据

向集群写入数据成功。

```Bash
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --write-out=table --endpoints=localhost:12379,localhost:22379,localhost:32379 endpoint status
{"level":"warn","ts":"2024-12-27T11:36:00.219+0800","logger":"etcd-client","caller":"v3@v3.5.6/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0x4000440700/localhost:12379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
Failed to get the status of endpoint localhost:12379 (context deadline exceeded)
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| localhost:22379 | 91bc3c398fb3c146 |   3.5.6 |   20 kB |      true |      false |         5 |         58 |                 58 |        |
| localhost:32379 | fd422379fda50e48 |   3.5.6 |   25 kB |     false |      false |         5 |         58 |                 58 |        |
+-----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
kangxiaoning@localhost:~/workspace/etcd$
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --endpoints=localhost:12379,localhost:22379,localhost:32379 get "hello"
kangxiaoning@localhost:~/workspace/etcd$ 
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --endpoints=localhost:12379,localhost:22379,localhost:32379 put "hello" "world"
OK
kangxiaoning@localhost:~/workspace/etcd$ bin/etcdctl --endpoints=localhost:12379,localhost:22379,localhost:32379 get "hello"
hello
world
kangxiaoning@localhost:~/workspace/etcd$
```