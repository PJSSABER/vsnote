# 环境准备
## GNU
```shell
sudo apt install build-essential  
```

## GO
```shell 
wget https://golang.google.cn/dl/go1.23.4.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.23.4.linux-amd64.tar.gz  

mkdir ~/go  
mkdir ~/go/src  
mkdir ~/go/bin  
 
# 在 ~/.bashrc 最后加入
export GOPATH="/home/<用户名>/go"  
export GOBIN="/home/<用户名>/go/bin"  
export PATH="/usr/local/go/bin:$GOPATH/bin:${PATH}"  

rm go1.23.4.linux-amd64.tar.gz
```

## Docker
```shell
sudo apt-get remove docker docker-engine docker.io containerd runc

sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

```
## etcd
see

https://etcd.io/docs/v3.5/install/

```shell
ETCD_VER=v3.5.4  
curl -L https://storage.googleapis.com/etcd/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz  
mkdir ~/etcd  
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C ~/etcd --strip-components=1  
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz  

# 添加环境变量
sudo nano ~/.bashrc  
```

## kubernetes
```shell
mkdir $GOPATH/src/k8s.io  && cd $GOPATH/src/k8s.io
git clone https://github.com/kubernetes/kubernetes.git  
git checkout -b kube1.24 v1.24.0  
```

### 编译

```shell
cd $GOPATH/src/k8s.io/kubernetes

# 编译单个组建：
sudo make WHAT="cmd/kube-apiserver"  
# 编译所有组件：
sudo make all  
# 启动本地单节点集群： 
sudo ./hack/local-up-cluster.sh  

```
#### 可能存在的问题
- 查看log, /var/kubelet.log
- 跑local-up-cluster.sh，集群起不来
  可能是https://blog.csdn.net/HYZX_9987/article/details/137835480

### Enable debug

```shell
cd $GOPATH/src/k8s.io/kubernetes
# kubernetes go编译文件
sudo vi ./hack/lib/golang.sh
# 查找build_binaries()函数 注意注释
# 保证 变量 gogcflags="${gogcflags} -N -l" 即可
```

# Golang standard layout
├── build
│   ├── ci
│   └── package
├── cmd             （运行的二进制）
│   └── _your_app_
├── configs         （配置文件 模板）
├── deployments      （IaaS，PaaS，系统和容器编排部署配置和模板）
├── docs
├── examples
├── init
├── internal
│   ├── app
│   │   └── _your_app_
│   └── pkg
│       └── _your_private_lib_
├── pkg                             (外部程序可以使用的库代码)
│   └── _your_public_lib_
├── scripts
├── test
├── .gitignore
├── LICENSE.md
├── Makefile
├── README.md
└── go.mod

# Go module

go mod init <project_name>

# channel 
通道是引用类型，通道类型的空值是nil
声明的通道后需要使用make函数初始化之后才能使用

创建channel的格式如下：
make(chan 元素类型, [缓冲大小])

关闭
close(ch)

1.对一个关闭的通道再发送值就会导致panic。
2.对一个关闭的通道进行接收会一直获取值直到通道为空。
3.对一个关闭的并且没有值的通道执行接收操作会得到对应类型的零值。
4.关闭一个已经关闭的通道会导致panic。

## channel 符号 <-

```Go
var v

ch <- v    // 发送值v到Channel ch中  
v := <-ch  // 从Channel ch中接收数据，并将数据赋值给v  

```
## channel 类型
```Go
chan T          // 可以接收和发送类型为 T 的数据  
chan<- float64  // 只可以用来发送 float64 类型的数据 到 channel
<-chan int      // 只可以用来接收 来自channel的 int 类型的数据 

v, ok := <-ch // 检测是否关闭
```

## 无缓冲channel
无缓冲通道上的发送操作会阻塞，直到另一个goroutine在该通道上执行接收操作，这时值才能发送成功，两个goroutine将继续执行。相反，如果接收操作先执行，接收方的goroutine将阻塞，直到另一个goroutine在该通道上发送一个值。

使用无缓冲通道进行通信将导致发送和接收的goroutine同步化。因此，无缓冲通道也被称为同步通道

```Go
func recv(c chan int) {
    ret := <-c
    fmt.Println("接收成功", ret)
}

func main() {
    ch := make(chan int)
    go recv(ch) // 启用goroutine从通道接收值
    ch <- 10
    fmt.Println("发送成功")
}

// 若没有 go recv(ch) 一句 则会形成死锁
```

## 有缓冲
```golang
func main() {
    ch := make(chan int, 1) // 创建一个容量为1的有缓冲区通道
    ch <- 10
    fmt.Println("发送成功")
}
```

## select 多路复用

select的使用类似于switch语句，它有一系列case分支和一个默认的分支。每个case会对应一个通道的通信（接收或发送）过程。select会一直等待，直到某个case的通信操作完成时，就会执行case分支对应的语句。

```golang
select {
  case <-chan1:
      // 如果chan1成功读到数据，则进行该case处理语句
  case chan2 <- 1:
      // 如果成功向chan2写入数据，则进行该case处理语句
  default:
      // 如果上面都没有成功，则进入default处理流程
}
```

# Go delve 调试 
GDB/G++ 等支持不是特别足够，特别对于go-routines使用DELVE调试更优

## 安装 与 基本命令
1. https://github.com/go-delve/delve/tree/master/Documentation/installation
   to work well with golang1.23, use 
   ```bash
    # Install the latest release:
    $ go install github.com/go-delve/delve/cmd/dlv@latest
   ```

2. 基本命令
   很象GDB， 特殊的如 routine 等 see
   https://github.com/go-delve/delve/tree/master/Documentation/cli

## 远程调试

1. remote
--headless
```shell
# for launch
dlv dap --listen=:12345

# for attach 貌似现在不支持 dlv dap???

dlv exec ./examples --listen=:12345 --headless
```
2. vscode
https://github.com/golang/vscode-go/wiki/debugging#remote-debugging

```shell
{
    "name": "Connect and launch",
    "type": "go",
    "debugAdapter": "dlv-dap", // the default
    "request": "launch",
    "port": 12345,
    "host": "172.21.10.197", // can skip for localhost
    "mode": "exec",
    "program": "/home/ubuntu/yyp/examples/examples",
    "substitutePath": [
        { "from": "${workspaceFolder}", "to": "/home/ubuntu/yyp/examples" },
    ]
}
```

# POSTMAN 调试 API server

1. 启动 cluster, 正确得单机集群
```
ubuntu@ip-172-21-10-197:~/yyp/examples$ ps -a | grep kube
 290388 pts/3    00:00:06 kube-apiserver
 290562 pts/3    00:00:01 kube-controller
 290564 pts/3    00:00:01 kube-scheduler
 290689 pts/4    00:00:01 kubelet
 291011 pts/5    00:00:00 kube-proxy
```
2. 记录 kube-apiserver 得启动命令，
   ```shell
/home/ubuntu/go/src/k8s.io/kubernetes/_output/local/bin/linux/amd64/kube-apiserver --authorization-mode=Node,RBAC --authorization-webhook-config-file= --authentication-token-webhook-config-file=  --cloud-provider= --cloud-config=   --v=3 --vmodule= --audit-policy-file=/tmp/local-up-cluster.sh.lUKI40/kube-audit-policy-file --audit-log-path=/tmp/kube-apiserver-audit.log --cert-dir=/var/run/kubernetes --egress-selector-config-file=/tmp/local-up-cluster.sh.lUKI40/kube_egress_selector_configuration.yaml --client-ca-file=/var/run/kubernetes/client-ca.crt --kubelet-client-certificate=/var/run/kubernetes/client-kube-apiserver.crt --kubelet-client-key=/var/run/kubernetes/client-kube-apiserver.key --service-account-key-file=/tmp/local-up-cluster.sh.lUKI40/kube-serviceaccount.key --service-account-lookup=true --service-account-issuer=https://kubernetes.default.svc --service-account-jwks-uri=https://kubernetes.default.svc/openid/v1/jwks --service-account-signing-key-file=/tmp/local-up-cluster.sh.lUKI40/kube-serviceaccount.key --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,Priority,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,NodeRestriction --disable-admission-plugins= --admission-control-config-file= --bind-address=0.0.0.0 --secure-port=6443 --tls-cert-file=/var/run/kubernetes/serving-kube-apiserver.crt --tls-private-key-file=/var/run/kubernetes/serving-kube-apiserver.key --storage-backend=etcd3 --storage-media-type=application/vnd.kubernetes.protobuf --etcd-servers=http://127.0.0.1:2379 --service-cluster-ip-range=10.0.0.0/24 --feature-gates=AllAlpha=false --emulated-version= --external-hostname=localhost --requestheader-username-headers=X-Remote-User --requestheader-group-headers=X-Remote-Group --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-client-ca-file=/var/run/kubernetes/request-header-ca.crt --requestheader-allowed-names=system:auth-proxy --proxy-client-cert-file=/var/run/kubernetes/client-auth-proxy.crt --proxy-client-key-file=/var/run/kubernetes/client-auth-proxy.key --cors-allowed-origins=//127.0.0.1(:[0-9]+)?$,//localhost(:[0-9]+)?$
   ```
   kill掉并使用dlv attach重新启动
3. 进行调试


# Go rountine

关键字

go func()

GPM是Go语言运行时（runtime）层面的实现，是go语言自己实现的一套调度系统。区别于操作系统调度OS线程。

1.G很好理解，就是个goroutine的，里面除了存放本goroutine信息外 还有与所在P的绑定等信息。
2.P管理着一组goroutine队列，P里面会存储当前goroutine运行的上下文环境（函数指针，堆栈地址及地址边界），P会对自己管理的goroutine队列做一些调度（比如把占用CPU时间较长的goroutine暂停、运行后续的goroutine等等）当自己的队列消费完了就去全局队列里取，如果全局队列里也消费完了会去其他P的队列里抢任务。
3.M（machine）是Go运行时（runtime）对操作系统内核线程的虚拟， M与内核线程一般是一一映射的关系， 一个groutine最终是要放到M上执行的；

P与M一般也是一一对应的。他们关系是： P管理着一组G挂载在M上运行。当一个G长久阻塞在一个M上时，runtime会新建一个M，阻塞G所在的P会把其他的G 挂载在新建的M上。当旧的G阻塞完成或者认为其已经死掉时 回收旧的M。

P的个数是通过runtime.GOMAXPROCS设定（最大256），Go1.5版本之后默认为物理线程数。 在并发量大的时候会增加一些P和M，但不会太多，切换太频繁的话得不偿失。

## runtime
控制协程得库

1. runtime.Gosched() 让出当前时间片
2. runtime.Goexit() 退出当前go-routine
3. runtime.GOMAXPROCS 确定需要使用多少个OS线程来同时执行Go代码

## 锁

### 互斥锁
```golang

var x int64
var wg sync.WaitGroup
var lock sync.Mutex

func add() {
    for i := 0; i < 5000; i++ {
        lock.Lock() // 加锁
        x = x + 1
        lock.Unlock() // 解锁
    }
    wg.Done()
}
func main() {
    wg.Add(2)
    go add()
    go add()
    wg.Wait()
    fmt.Println(x)
}

```

### 读写锁
```go
var (
    x      int64
    wg     sync.WaitGroup
    lock   sync.Mutex
    rwlock sync.RWMutex
)

func write() {
    // lock.Lock()   // 加互斥锁
    rwlock.Lock() // 加写锁
    x = x + 1
    time.Sleep(10 * time.Millisecond) // 假设读操作耗时10毫秒
    rwlock.Unlock()                   // 解写锁
    // lock.Unlock()                     // 解互斥锁
    wg.Done()
}

func read() {
    // lock.Lock()                  // 加互斥锁
    rwlock.RLock()               // 加读锁
    time.Sleep(time.Millisecond) // 假设读操作耗时1毫秒
    rwlock.RUnlock()             // 解读锁
    // lock.Unlock()                // 解互斥锁
    wg.Done()
}

func main() {
    start := time.Now()
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go write()
    }

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go read()
    }

    wg.Wait()
    end := time.Now()
    fmt.Println(end.Sub(start))
}
```

## Sync

有点类似 条件变量
sync.WaitGroup有以下几个方法：

方法名	功能
(wg * WaitGroup) Add(delta int)	计数器+delta
(wg *WaitGroup) Done()	计数器-1
(wg *WaitGroup) Wait()	阻塞直到计数器变为0

```golang
var wg sync.WaitGroup

func hello() {
    defer wg.Done()
    fmt.Println("Hello Goroutine!")
}
func main() {
    wg.Add(1)
    go hello() // 启动另外一个goroutine去执行hello函数
    fmt.Println("main goroutine done!")
    wg.Wait()
}
```

### sync.Once
在编程的很多场景下我们需要确保某些操作在高并发的场景下只执行一次，例如只加载一次配置文件、只关闭一次通道等。

### sync.Map
```go
var m = sync.Map{}

func main() {
    wg := sync.WaitGroup{}
    for i := 0; i < 20; i++ {
        wg.Add(1)
        go func(n int) {
            key := strconv.Itoa(n)
            m.Store(key, n)
            value, _ := m.Load(key)
            fmt.Printf("k=:%v,v:=%v\n", key, value)
            wg.Done()
        }(i)
    }
    wg.Wait()
}
```

## 


# Go runtime GC

## force gc
Goroutine 2 - User: /usr/local/go/src/runtime/proc.go:425 runtime.gopark (0x46dd5c) [force gc (idle)]
Description: This goroutine triggers garbage collection (GC) when the runtime detects that the application is idle.
Location: /usr/local/go/src/runtime/proc.go:425 runtime.gopark
Purpose: Monitors the application and forces a garbage collection cycle during idle times to clean up unused memory.

## sweep wait
Goroutine 3 - User: /usr/local/go/src/runtime/proc.go:425 runtime.gopark (0x46dd5c) [GC sweep wait]
Description: This goroutine is responsible for sweeping memory that is no longer used (garbage collection phase). It waits until a GC cycle occurs.
Location: /usr/local/go/src/runtime/proc.go:425 runtime.gopark
Purpose: Helps manage memory by reclaiming unused resources.

## scavenge wait
Goroutine 4 - User: /usr/local/go/src/runtime/proc.go:425 runtime.gopark (0x46dd5c) [GC scavenge wait]
Description: This goroutine is part of the garbage collector and handles scavenging unused memory pages for reuse.
Location: /usr/local/go/src/runtime/proc.go:425 runtime.gopark
Purpose: Optimizes memory usage by reusing pages and reducing fragmentation.

## finalizer wait
Goroutine 5 - User: /usr/local/go/src/runtime/proc.go:425 runtime.gopark (0x46dd5c) [finalizer wait]
Description: This goroutine waits for objects with finalizers to be ready for cleanup. Finalizers are special functions that run when an object is garbage collected.
Location: /usr/local/go/src/runtime/proc.go:425 runtime.gopark
Purpose: Ensures finalizer functions are called on objects that require special cleanup before being garbage collected.

# 
https://github.com/PJSSABER/virtual-kubelet