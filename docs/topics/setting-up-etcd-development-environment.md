# 搭建Etcd开发环境

<show-structure depth="3"/>

通过`Goreman`搭建单机3实例的集群环境，对集群中一个实例进行debug，效果可以查看`4. debugging etcdctl put`部分。

<procedure>
<img src="debug-etcd-put-1.png"  thumbnail="true"/>
</procedure>

## 1. 源码编译

### 1.1 编译命令

修改build.sh以支持debuginfo。
```Bash
kangxiaoning@localhost:~/workspace/etcd$ git diff build.sh
diff --git a/build.sh b/build.sh
index 05d916ea3..1d14ea5ca 100755
--- a/build.sh
+++ b/build.sh
@@ -20,7 +20,8 @@ CGO_ENABLED="${CGO_ENABLED:-0}"
 # Set GO_LDFLAGS="-s" for building without symbols for debugging.
 # shellcheck disable=SC2206
 GO_LDFLAGS=(${GO_LDFLAGS:-} "-X=${VERSION_SYMBOL}=${GIT_SHA}")
-GO_BUILD_ENV=("CGO_ENABLED=${CGO_ENABLED}" "GO_BUILD_FLAGS=${GO_BUILD_FLAGS:-}" "GOOS=${GOOS}" "GOARCH=${GOARCH}")
+# GO_BUILD_ENV=("CGO_ENABLED=${CGO_ENABLED}" "GO_BUILD_FLAGS=${GO_BUILD_FLAGS:-}" "GOOS=${GOOS}" "GOARCH=${GOARCH}")
+GO_BUILD_ENV=("CGO_ENABLED=${CGO_ENABLED}" "GO_BUILD_FLAGS=${GO_BUILD_FLAGS:--gcflags=\"all=-N -l\"}" "GOOS=${GOOS}" "GOARCH=${GOARCH}")
 
 GOFAIL_VERSION=$(cd tools/mod && go list -m -f '{{.Version}}' go.etcd.io/gofail)
 # enable/disable failpoints
@@ -65,8 +66,7 @@ etcd_build() {
     cd ./server
     # Static compilation is useful when etcd is run in a container. $GO_BUILD_FLAGS is OK
     # shellcheck disable=SC2086
-    run env "${GO_BUILD_ENV[@]}" go build ${GO_BUILD_FLAGS:-} \
-      -trimpath \
+    run env "${GO_BUILD_ENV[@]}" go build -v ${GO_BUILD_FLAGS:-} \
       -installsuffix=cgo \
       "-ldflags=${GO_LDFLAGS[*]}" \
       -o="../${out}/etcd" . || return 2
@@ -76,8 +76,7 @@ etcd_build() {
   # shellcheck disable=SC2086
   (
     cd ./etcdutl
-    run env GO_BUILD_FLAGS="${GO_BUILD_FLAGS:-}" "${GO_BUILD_ENV[@]}" go build ${GO_BUILD_FLAGS:-} \
-      -trimpath \
+    run env GO_BUILD_FLAGS="${GO_BUILD_FLAGS:-}" "${GO_BUILD_ENV[@]}" go build -v ${GO_BUILD_FLAGS:-} \
       -installsuffix=cgo \
       "-ldflags=${GO_LDFLAGS[*]}" \
       -o="../${out}/etcdutl" . || return 2
@@ -87,8 +86,7 @@ etcd_build() {
   # shellcheck disable=SC2086
   (
     cd ./etcdctl
-    run env GO_BUILD_FLAGS="${GO_BUILD_FLAGS:-}" "${GO_BUILD_ENV[@]}" go build ${GO_BUILD_FLAGS:-} \
-      -trimpath \
+    run env GO_BUILD_FLAGS="${GO_BUILD_FLAGS:-}" "${GO_BUILD_ENV[@]}" go build -v ${GO_BUILD_FLAGS:-} \
       -installsuffix=cgo \
       "-ldflags=${GO_LDFLAGS[*]}" \
       -o="../${out}/etcdctl" . || return 2
@@ -118,8 +116,7 @@ tools_build() {
     echo "Building" "'${tool}'"...
     run rm -f "${out}/${tool}"
     # shellcheck disable=SC2086
-    run env GO_BUILD_FLAGS="${GO_BUILD_FLAGS:-}" CGO_ENABLED=${CGO_ENABLED} go build ${GO_BUILD_FLAGS:-} \
-      -trimpath \
+    run env GO_BUILD_FLAGS="${GO_BUILD_FLAGS:-}" CGO_ENABLED=${CGO_ENABLED} go build -v ${GO_BUILD_FLAGS:-} \
       -installsuffix=cgo \
       "-ldflags=${GO_LDFLAGS[*]}" \
       -o="${out}/${tool}" "./${tool}" || return 2
@@ -142,7 +139,7 @@ tests_build() {
       run rm -f "../${out}/${tool}"
 
       # shellcheck disable=SC2086
-      run env CGO_ENABLED=${CGO_ENABLED} GO_BUILD_FLAGS="${GO_BUILD_FLAGS:-}" go build ${GO_BUILD_FLAGS:-} \
+      run env CGO_ENABLED=${CGO_ENABLED} GO_BUILD_FLAGS="${GO_BUILD_FLAGS:-}" go build -v ${GO_BUILD_FLAGS:-} \
         -installsuffix=cgo \
         "-ldflags=${GO_LDFLAGS[*]}" \
         -o="../${out}/${tool}" "./${tool}" || return 2
kangxiaoning@localhost:~/workspace/etcd$
```
{collapsible="true" collapsed-title="build.sh" default-state="collapsed"}

```Shell
git clone https://github.com/etcd-io/etcd.git
cd etcd
git checkout -b debug-v3.5.6 v3.5.6
./build.sh
```

### 1.2 报错解决

编译报错`github.com/myitcv/gobin`下载失败，可以参考如下方法解决。

- 报错信息

```Shell
kangxiaoning@localhost:~/workspace/etcd$ make build
GO_BUILD_FLAGS="-v" ./build.sh
go: downloading github.com/json-iterator/go v1.1.11
go: downloading golang.org/x/time v0.0.0-20210220033141-f8bda1e9f3ba
go: downloading google.golang.org/grpc v1.41.0
go: downloading gopkg.in/cheggaaa/pb.v1 v1.0.28
go: downloading golang.org/x/crypto v0.0.0-20220411220226-7b82a4e95df4
go: downloading golang.org/x/sys v0.0.0-20210615035016-665e8c7367d1
go: downloading google.golang.org/genproto v0.0.0-20210602131652-f16073e35f0c
go: downloading golang.org/x/net v0.0.0-20211112202133-69e39bad7dc2
go: downloading go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc v0.25.0
go: downloading go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc v1.0.1
go: downloading go.opentelemetry.io/otel v1.0.1
go: downloading go.opentelemetry.io/otel/sdk v1.0.1
go: downloading go.opentelemetry.io/otel/exporters/otlp/otlptrace v1.0.1
go: downloading github.com/prometheus/common v0.26.0
go: downloading github.com/prometheus/procfs v0.6.0
go: downloading github.com/sirupsen/logrus v1.7.0
go: downloading go.opentelemetry.io/otel/trace v1.0.1
go: downloading go.opentelemetry.io/proto/otlp v0.9.0
go: downloading github.com/golang-jwt/jwt/v4 v4.4.2
go: downloading golang.org/x/text v0.3.6
go: downloading github.com/cenkalti/backoff/v4 v4.1.1
% 'env' 'GO111MODULE=off' 'go' 'get' 'github.com/myitcv/gobin'
stderr: go: modules disabled by GO111MODULE=off; see 'go help modules'
FAIL: (code:1):
  % 'env' 'GO111MODULE=off' 'go' 'get' 'github.com/myitcv/gobin'
make: *** [Makefile:25: build] Error 1
kangxiaoning@localhost:~/workspace/etcd$
```
{collapsible="true" collapsed-title="stderr: go: modules disabled by GO111MODULE=off; see 'go help modules'" default-state="collapsed"}

- 解决方法

修改如下脚本后重新build。 
 
```Shell
kangxiaoning@localhost:~/workspace/etcd$ git diff scripts/test_lib.sh
diff --git a/scripts/test_lib.sh b/scripts/test_lib.sh
index 9053f9ce8..92d8b9edd 100644
--- a/scripts/test_lib.sh
+++ b/scripts/test_lib.sh
@@ -303,7 +303,7 @@ function tool_exists {
 
 # Ensure gobin is available, as it runs majority of the tools
 if ! command -v "gobin" >/dev/null; then
-    run env GO111MODULE=off go get github.com/myitcv/gobin || exit 1
+    run env GO111MODULE=on go get github.com/myitcv/gobin || exit 1
 fi
 
 # tool_get_bin [tool] - returns absolute path to a tool binary (or returns error)
kangxiaoning@localhost:~/workspace/etcd$
```
{collapsible="true" collapsed-title="git diff scripts/test_lib.sh" default-state="collapsed"}

### 1.3 检查debug info

通过`file`命令检查二进制文件，输出中有`with debug_info, not stripped`则表示生成的二进制文件带有debug信息，这样才可以使用`dlv`进行远程调试。

```Shell
kangxiaoning@localhost:~/workspace/etcd$ file bin/etcd
bin/etcd: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, Go BuildID=FUUsnOg_y9TblbydkZxe/RV6n4-rQlTH7fac3UXTk/hLpXIr4OsqZyhyvQtGZg/-z68ihpaGGcUc_t8tY0p, with debug_info, not stripped
kangxiaoning@localhost:~/workspace/etcd$
```

## 2. 启动脚本

编译完成后，运行如下命令即可启动集群。

```Shell
cd /home/kangxiaoning/workspace/etcd
goreman -f ./Procfile start
```

```Shell
kangxiaoning@localhost:~/workspace/etcd$ more Procfile
# Use goreman to run `go get github.com/mattn/goreman`
# Change the path of bin/etcd if etcd is located elsewhere

etcd1: bin/etcd --name infra1 --listen-client-urls http://127.0.0.1:12379 --advertise-client-urls http://127.0.0.1:12379 --listen-peer-urls http://127.0.0.1:12380 --initial-advertise-peer-urls http://127.0.0.1:1238
0 --initial-cluster-token etcd-cluster-1 --initial-cluster 'infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32380' --initial-cluster-state new --enable-pprof --logger=zap --log-o
utputs=stderr
etcd2: bin/etcd --name infra2 --listen-client-urls http://127.0.0.1:22379 --advertise-client-urls http://127.0.0.1:22379 --listen-peer-urls http://127.0.0.1:22380 --initial-advertise-peer-urls http://127.0.0.1:2238
0 --initial-cluster-token etcd-cluster-1 --initial-cluster 'infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32380' --initial-cluster-state new --enable-pprof --logger=zap --log-o
utputs=stderr
etcd3: bin/etcd --name infra3 --listen-client-urls http://127.0.0.1:32379 --advertise-client-urls http://127.0.0.1:32379 --listen-peer-urls http://127.0.0.1:32380 --initial-advertise-peer-urls http://127.0.0.1:3238
0 --initial-cluster-token etcd-cluster-1 --initial-cluster 'infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32380' --initial-cluster-state new --enable-pprof --logger=zap --log-o
utputs=stderr
#proxy: bin/etcd grpc-proxy start --endpoints=127.0.0.1:2379,127.0.0.1:22379,127.0.0.1:32379 --listen-addr=127.0.0.1:23790 --advertise-client-url=127.0.0.1:23790 --enable-pprof

# A learner node can be started using Procfile.learner
kangxiaoning@localhost:~/workspace/etcd$
```
{collapsible="true" collapsed-title="Procfile" default-state="collapsed"}

后了后续更方便的debugging etcd，准备了如下脚本。

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

1. 运行`goreman`命令启动集群
2. 运行`./debug-etcd.sh`脚本启动`dlv`进程
3. 通过GoLand连接

```Shell
cd /home/kangxiaoning/workspace/etcd
goreman -f ./Procfile start
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

## 4. debugging `etcdctl put`

在`EtcdServer.applyEntries()`函数中打个断点，然后执行一个`etcdctl put "hello" "world"`，在断点处一步一步跟踪执行过程。

```Go
func (s *EtcdServer) applyEntries(ep *etcdProgress, apply *apply) {
	if len(apply.entries) == 0 {
		return
	}
	firsti := apply.entries[0].Index
	if firsti > ep.appliedi+1 {
		lg := s.Logger()
		lg.Panic(
			"unexpected committed entry index",
			zap.Uint64("current-applied-index", ep.appliedi),
			zap.Uint64("first-committed-entry-index", firsti),
		)
	}
	var ents []raftpb.Entry
	if ep.appliedi+1-firsti < uint64(len(apply.entries)) {
		ents = apply.entries[ep.appliedi+1-firsti:]
	}
	if len(ents) == 0 {
		return
	}
	var shouldstop bool
	if ep.appliedt, ep.appliedi, shouldstop = s.apply(ents, &ep.confState); shouldstop {
		go s.stopWithDelay(10*100*time.Millisecond, fmt.Errorf("the member has been permanently removed from the cluster"))
	}
}
```
{collapsible="true" collapsed-title="EtcdServer.applyEntries()" default-state="collapsed"}

```Shell
bin/etcdctl --endpoints=localhost:12379,localhost:22379,localhost:32379 put "hello" "world"
```

如下是部分示例。

从下图可以看到`hello world`关键字。

<procedure>
<img src="debug-etcd-put-1.png"  thumbnail="true"/>
</procedure>

<procedure>
<img src="debug-etcd-put-2.png"  thumbnail="true"/>
</procedure>

<procedure>
<img src="debug-etcd-put-3.png"  thumbnail="true"/>
</procedure>

<procedure>
<img src="debug-etcd-put-4.png"  thumbnail="true"/>
</procedure>

<procedure>
<img src="debug-etcd-put-5.png"  thumbnail="true"/>
</procedure>
<procedure>
<img src="debug-etcd-put-6.png"  thumbnail="true"/>
</procedure>

<procedure>
<img src="debug-etcd-put-7.png"  thumbnail="true"/>
</procedure>

<procedure>
<img src="debug-etcd-put-8.png"  thumbnail="true"/>
</procedure>

<procedure>
<img src="debug-etcd-put-9.png"  thumbnail="true"/>
</procedure>

<procedure>
<img src="debug-etcd-put-10.png"  thumbnail="true"/>
</procedure>

<procedure>
<img src="debug-etcd-put-11.png"  thumbnail="true"/>
</procedure>
