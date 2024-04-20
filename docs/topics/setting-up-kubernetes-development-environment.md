# 搭建Kubernetes开发环境

<show-structure depth="3"/>

目的是**在MacOS上通过GoLand对Ubuntu中运行的Kubernetes执行Debug**，这里记录下在Ubuntu搭建Kubernetes开发环境的过程，效果如下图所示。

<procedure>
<img src="debug-example.png" alt="etcd service" thumbnail="true"/>
</procedure>

<procedure>
<img src="debug-example-02.png" alt="etcd service" thumbnail="true"/>
</procedure>

<procedure>
<img src="debug-kubelet.png" alt="etcd service" thumbnail="true"/>
</procedure>

## 1. Ubuntu配置

### 1.1 硬件要求

- 8GB of RAM
- 50GB of free disk space

### 1.2 镜像配置

使用阿里云Ubuntu镜像源，根据CPU架构选择下面指引，找到对应版本的配置进行修改。

- [x86架构使用的源](https://developer.aliyun.com/mirror/ubuntu)
- [Arm架构使用的源](https://developer.aliyun.com/mirror/ubuntu-ports)

### 1.3 时间设置

根据个人喜好设置，不影响使用。

```Shell
# 时区
timedatectl set-timezone Asia/Shanghai

# 24小时制
echo "LC_TIME=en_DK.UTF-8" >> /etc/default/locale && cat /etc/default/locale
```

## 2 依赖安装

### 2.1 GNU Development Tools

```Shell
sudo apt update
sudo apt install build-essential rsync jq python3-pip
pip install pyyaml
```

### 2.2 Docker

```Shell

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 2.3 Go

Kubernetes对Go的版本要求参考[Go Requirement](https://github.com/kubernetes/community/blob/master/contributors/devel/development.md#go)，社区建议安装最新版本的Go。

在[Go社区下载页面](https://golang.google.cn/dl/)找到需要的版本。

```Shell
# 根根据CPU架构修改包名
wget https://golang.google.cn/dl/go1.22.1.linux-arm64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.22.1.linux-arm64.tar.gz

mkdir -p $HOME/go/{src,bin}

echo 'export GOPATH=$HOME/go' >> .bashrc
echo 'export GOBIN=$HOME/go/bin' >> .bashrc
echo 'export PATH=/usr/local/go/bin:$GOPATH/bin:${PATH}' >> .bashrc
source ~/.bashrc

# 设置Go代理
go env -w GO111MODULE="on"  
go env -w GOPROXY="https://goproxy.cn,direct"  
```

### 2.4 Delve

```Shell
go install github.com/go-delve/delve/cmd/dlv@latest
```

### 2.5 Clone kubernetes

clone代码后切换到对应分支，后面需要使用对应版本的脚本。

```Shell
mkdir $GOPATH/src/k8s.io  && cd $GOPATH/src/k8s.io
git clone https://github.com/kubernetes/kubernetes.git 

cd $GOPATH/src/k8s.io/kubernetes

# 切换到v1.24.17
git tag | grep 1.24
git checkout -b debug-1.24.17 v1.24.17

```

### 2.6 Etcd

```Shell
cd $GOPATH/src/k8s.io/kubernetes
./hack/install-etcd.sh

echo 'export PATH=$GOPATH/src/k8s.io/kubernetes/third_party/etcd:${PATH}' >> ~/.bashrc
source ~/.bashrc
```

### 2.7 Sudo配置

执行sudo命令时，Ubuntu为了确保安全，会将环境变量重置为安全的环境变量，导致`sudo hack/local-up-cluster.sh`找不到etcd命令，进而启动失败。

```Shell
sudo visudo

# 添加相关环境变量，修改后如下
Defaults	secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/usr/local/go/bin:/home/kangxiaoning/go/bin:/home/kangxiaoning/go/src/k8s.io/kubernetes/third_party/etcd"
```


## 3. 启动集群

### 3.1 debug编译

默认的编译选项生成的二进制文件不包含debug需要的symbol信息，

```Shell
cd $GOPATH/src/k8s.io/kubernetes

# 查看编译选项
make help

# 执行Debug编译
sudo make all DBG=1
```


### 3.2 启动集群

切换到指定版本，然后运行脚本执行编译及启动，这里使用v1.24.17版本。

```Shell
# 确保已切换到v1.24.17
cd $GOPATH/src/k8s.io/kubernetes
git status

# 编译并启动单机集群
sudo hack/local-up-cluster.sh

# 验证进程
ps -a|egrep 'kube|etcd'
```

如果希望运行在后台，编写如下脚本，后续执行`./start-cluster.sh`即可。

```Shell
kangxiaoning@localhost:~/cmd$ more start-cluster.sh 
#!/bin/bash

nohup sudo -b $GOPATH/src/k8s.io/kubernetes/hack/local-up-cluster.sh -O

kangxiaoning@localhost:~/cmd$
```

正常启动的结果如下。

```Shell
kangxiaoning@localhost:~$ ps -a|egrep 'kube|etcd'
  75651 pts/1    00:00:01 etcd
  75860 pts/1    00:00:07 kube-apiserver
  76145 pts/1    00:00:02 kube-controller
  76147 pts/1    00:00:00 kube-scheduler
  76299 pts/3    00:00:02 kubelet
  76851 pts/4    00:00:00 kube-proxy
kangxiaoning@localhost:~$ 
```

### 3.3 配置kubectl

根据启动结果做下面设置。

```Shell
echo 'export KUBECONFIG=/var/run/kubernetes/admin.kubeconfig' >> ~/.bashrc
echo 'alias kubectl="/home/kangxiaoning/go/src/k8s.io/kubernetes/cluster/kubectl.sh"' >> ~/.bashrc
source ~/.bashrc
```

正常运行结果如下。

```Shell
kangxiaoning@localhost:~$ kubectl get node
NAME        STATUS   ROLES    AGE   VERSION
127.0.0.1   Ready    <none>   25m   v1.24.17-dirty
kangxiaoning@localhost:~$
```


## 4. 简化使用

### 4.1 启停集群

使用如下对倒对集群执行启动、停止操作。

- 启动集群： `kstart`
- 停止集群： `kstop`

通过如下命令配置`alias`。

```Shell
echo 'alias kstart="/home/kangxiaoning/cmd/start-cluster.sh"' >> ~/.bashrc
echo 'alias kstop="/home/kangxiaoning/cmd/stop-cluster.sh"' >> ~/.bashrc
source ~/.bashrc
```
对应的脚本分别如下。

#### 4.1.1 start-cluster.sh

启动集群的脚本。

```Shell
#!/bin/bash

PWD=/home/kangxiaoning/cmd
sudo -b $GOPATH/src/k8s.io/kubernetes/hack/local-up-cluster.sh -O > ${PWD}/log/start-cluster.log 2>&1
```

#### 4.1.2 stop-cluster.sh

停止集群的脚本。

```Shell
#!/bin/bash

sudo pkill -f $GOPATH/src/k8s.io/kubernetes/hack/local-up-cluster.sh
```

### 4.2 重启apiserver

使用dlv启动kube-apiserver，以支持debug。

**使用方法：** 运行`./debug-apiserver.sh`脚本，如果kube-apiserver已经在运行会先执行kill，然后再通过dlv启动kube-apiserver。

#### 4.2.1 debug-apiserver.sh
```Shell
#!/bin/bash

PWD=/home/kangxiaoning/cmd

if ps -ef | grep -v grep | grep -q kube-apiserver; then
    echo "kube-apiserver is running under dlv"
    ${PWD}/lib/stop-apiserver.sh
else
    echo "kube-apiserver is not running under dlv"
fi

sleep 1

sudo nohup ${PWD}/lib/start-apiserver.sh > /dev/null 2>&1 &
```

#### 4.2.2 start-apiserver.sh

```Shell
dlv --headless exec /home/kangxiaoning/go/src/k8s.io/kubernetes/_output/local/bin/linux/arm64/kube-apiserver \
--listen=:12306 --api-version=2 -- \
--authorization-mode=Node,RBAC \
--cloud-provider= \
--cloud-config= \
--v=3 \
--vmodule= \
--audit-policy-file=/tmp/kube-audit-policy-file \
--audit-log-path=/tmp/kube-apiserver-audit.log \
--authorization-webhook-config-file= \
--authentication-token-webhook-config-file= \
--cert-dir=/var/run/kubernetes \
--egress-selector-config-file=/tmp/kube_egress_selector_configuration.yaml \
--client-ca-file=/var/run/kubernetes/client-ca.crt \
--kubelet-client-certificate=/var/run/kubernetes/client-kube-apiserver.crt \
--kubelet-client-key=/var/run/kubernetes/client-kube-apiserver.key \
--service-account-key-file=/tmp/kube-serviceaccount.key \
--service-account-lookup=true \
--service-account-issuer=https://kubernetes.default.svc \
--service-account-jwks-uri=https://kubernetes.default.svc/openid/v1/jwks \
--service-account-signing-key-file=/tmp/kube-serviceaccount.key \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,Priority,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,NodeRestriction \
--disable-admission-plugins= \
--admission-control-config-file= \
--bind-address=0.0.0.0 \
--secure-port=6443 \
--tls-cert-file=/var/run/kubernetes/serving-kube-apiserver.crt \
--tls-private-key-file=/var/run/kubernetes/serving-kube-apiserver.key \
--storage-backend=etcd3 \
--storage-media-type=application/vnd.kubernetes.protobuf \
--etcd-servers=http://127.0.0.1:2379 \
--service-cluster-ip-range=10.0.0.0/24 \
--feature-gates=AllAlpha=false \
--external-hostname=localhost \
--requestheader-username-headers=X-Remote-User \
--requestheader-group-headers=X-Remote-Group \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-client-ca-file=/var/run/kubernetes/request-header-ca.crt \
--requestheader-allowed-names=system:auth-proxy \
--proxy-client-cert-file=/var/run/kubernetes/client-auth-proxy.crt \
--proxy-client-key-file=/var/run/kubernetes/client-auth-proxy.key
```

#### 4.2.3 stop-apiserver.sh

```Shell
#!/bin/bash

ps -ef|grep kube-apiserver|grep -v grep|awk '{print $2}'|xargs sudo kill -9
```

### 4.3 重启kubelet

>
> GoLand中打断点报错 **cannot find debugger path for ...**
>
> **问题原因：** [Remote debugging results in can not find debugger path for xxx/xxx/fobar.go](https://youtrack.jetbrains.com/issue/GO-12999)
>
> **解决方法一： 删除`_output/local/go/`目录**
> 
> **解决方法二： 在GoLand中选中`_output`目录，右击->Mark Directory as->Excluded**
> 
{style="warning"}

使用dlv启动kubelet，以支持debug。

使用方法： 运行`./debug-kubelet.sh`脚本，如果kubelet已经在运行会先执行kill，然后再通过dlv启动kubelet。

#### 4.3.1 保存kubelet配置

在集群运行的情况下执行如下命令，保存kubelet配置，后面dlv启动脚本需要使用。

```Shell
sudo cp /tmp/kubelet.yaml /var/lib/kubelet/config.yaml
```

配置下面脚本。

#### 4.3.2 debug-kubelet.sh

```Shell
#!/bin/bash

PWD=/home/kangxiaoning/cmd

if ps -ef | grep -v grep | grep -q bin/kubelet; then
    echo "kubelet is running under dlv"
    ${PWD}/lib/stop-kubelet.sh
else
    echo "kubelet is not running under dlv"
fi

sleep 1

sudo nohup ${PWD}/lib/start-kubelet.sh > /dev/null 2>&1 &
```

#### 4.3.3 start-kubelet.sh

```Shell
#!/bin/bash

dlv --headless exec /home/kangxiaoning/go/src/k8s.io/kubernetes/_output/bin/kubelet --listen=:2355 --api-version=2 -- --v=3 --vmodule= --container-runtime=remote --hostname-override=127.0.0.1 --cloud-provider= --cloud-config= --bootstrap-kubeconfig=/var/run/kubernetes/kubelet.kubeconfig --kubeconfig=/var/run/kubernetes/kubelet-rotated.kubeconfig --container-runtime-endpoint=unix:///run/containerd/containerd.sock --config=/var/lib/kubelet/config.yaml
```

#### 4.3.4 stop-kubelet.sh

```Shell
#!/bin/bash

ps -ef|grep bin/kubelet|grep -v grep|awk '{print $2}'|xargs sudo kill -9
```

## 5. GoLand debug

### 5.1 GoLand debug配置

接下来配置GoLand的debug，步骤如下。

Edit Configurations -> Add New Configuration -> Go Remote

下图只需要输入Host输入框的192.168.166.133，端口在dlv启动了使用了默认的2345，不需要修改。

<procedure>
<img src="debug-configuration.png" alt="etcd service" thumbnail="true"/>
</procedure>

### 5.2 debug kube-apiserver

假设现在对`kubernetes/cmd/kube-apiserver/apiserver.go`的`command := app.NewAPIServerCommand()`这一行进行debug，需要了解`command`结构体的具体内容，在这一行打个断点，然后点击debug图标，即得到如下debug内容。

<procedure>
<img src="debug-kube-apiserver.png" alt="etcd service" thumbnail="true"/>
</procedure>

### 5.2 debug kubelet

<procedure>
<img src="debug-kubelet.png" alt="etcd service" thumbnail="true"/>
</procedure>


如果需要debug其它Kubernetes组件，使用dlv启动对应进程，然后在GoLand做相应配置即可。

## 参考
- [](https://github.com/kubernetes/community/blob/master/contributors/devel/development.md)
- [](https://github.com/kubernetes/community/blob/master/contributors/devel/running-locally.md)
- [Remote debugging results in can not find debugger path for xxx/xxx/fobar.go](https://youtrack.jetbrains.com/issue/GO-12999)