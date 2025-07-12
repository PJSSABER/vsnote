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

export KUBERNETES_PROVIDER=local

cluster/kubectl.sh config set-cluster local --server=https://localhost:6443 --certificate-authority=/var/run/kubernetes/server-ca.crt
cluster/kubectl.sh config set-credentials myself --client-key=/var/run/kubernetes/client-admin.key --client-certificate=/var/run/kubernetes/client-admin.crt
cluster/kubectl.sh config set-context local --cluster=local --user=myself
cluster/kubectl.sh config use-context local
cluster/kubectl.sh
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

##  _ 占位符

var _ PodAdminHandler = (*podAdminHandler)(nil)

用于通过编译， 表明申明的该变量在后续中并未使用
The code is checking whether podAdminHandler implements PodAdminHandler

### 在import 中使用
import _ "github.com/mattn/go-sqlite3"
The blank identifier means the package's exported symbols (functions, types, etc.) are not directly accessible in the importing code

import _ "github.com/mattn/go-sqlite3" ensures the SQLite driver is registered with database/sql

When import _ is used, the Go compiler:
Loads the package.
Executes any init() functions in that package.
For github.com/mattn/go-sqlite3, this includes registering the SQLite driver.
the init() function is a special function used for initialization. It is automatically called by the Go runtime before the main() function is executed
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

# interface && 反射

# struct tag & 反射

```golang
// an example using reflect 
package main

import (
	"encoding/json"
	"fmt"
	"reflect"
)

type User struct {
	ID    int    `json:"id" db:"user_id" binding:"required"`
	Name  string `json:"name" db:"full_name"`
	Email string `json:"email,omitempty" db:"email_address"`
}

/*
    User 的 struct tag 事例
    当encode到json格式的时候，id 作为 json 的 key
    binding:"required" → Ensures the field is not empty.
    binding:"dive,required" → Ensures each element inside Tags is non-empty.
*/

func main() {
	user := User{ID: 1, Name: "Alice", Email: "alice@example.com"}

	// JSON Serialization using struct tags
	jsonData, _ := json.Marshal(user)
	fmt.Println(string(jsonData)) // Output: {"id":1,"name":"Alice","email":"alice@example.com"}

	// Reflection: Read Struct Tags
	t := reflect.TypeOf(user)       // get types of variable during runtime
	for i := 0; i < t.NumField(); i++ {
		field := t.Field(i)
		fmt.Printf("Field: %s, JSON Tag: %s, DB Tag: %s\n",
			field.Name, field.Tag.Get("json"), field.Tag.Get("db"))
	}
    /*
    output example:
        Field: ID, JSON Tag: id, DB Tag: user_id
        Field: Name, JSON Tag: name, DB Tag: full_name
        Field: Email, JSON Tag: email,omitempty, DB Tag: email_address
    */

/************* runtime get & modify value by reflect **********/
	v := reflect.ValueOf(&user).Elem() // Get the actual struct

	// Modify the Name field
	nameField := v.FieldByName("ID")
	if nameField.CanSet() {
		nameField.SetInt(2)
	}

	jsonData, _ = json.Marshal(user)
	fmt.Println(string(jsonData)) // Output: {"id":2,"name":"Alice","email":"alice@example.com"}

}
```

#
if a variable is exported (starts with an uppercase letter), you can use it from another package.

https://github.com/PJSSABER/virtual-kubelet


# go defer 实现
Go 语言的 defer 会在当前函数返回前执行传入的函数，defer 的执行发生在 return 语句将返回值写入返回值变量之后，并且在函数真正退出之前。当发生 panic 时，defer 语句仍然会被执行，这使得 defer 成为处理错误和恢复状态的理想选择

## defer 关键字的调用时机以及多次调用 defer 时执行顺序是如何确定的； 如果有多个 defer 语句，它们会按照后进先出的顺序执行，最晚定义的 defer 语句最先执行, 类似栈
```golang
func main() {
    for i := 0; i < 5; i++ {
        defer fmt.Println(i)
    } 
}

// $ go run main.go
// 4
// 3
// 2
// 1
// 0
```
## defer 关键字使用传值的方式传递参数时会进行预计算，导致不符合预期的结果；

## defer 实现

### 结构体 与 机制
```golang
// runtime._defer
type _defer struct {
	siz       int32     // 内存大小
	started   bool
	openDefer bool  // 开放编码优化
	sp        uintptr // stack pointer
	pc        uintptr // pc counter
	fn        *funcval
	_panic    *_panic
	link      *_defer  // 单向链表的指针
}

// 中间代码生成阶段的 cmd/compile/internal/gc.state.stmt 会负责处理程序中的 defer，该函数会根据条件的不同，使用三种不同的机制处理该关键字
func (s *state) stmt(n *Node) {
	...
	switch n.Op {
	case ODEFER:
		if s.hasOpenDefers {
			s.openDeferRecord(n.Left) // 开放编码
		} else {
			d := callDefer // 堆分配
			if n.Esc == EscNever {
				d = callDeferStack // 栈分配
			}
			s.callResult(n.Left, d)
		}
	}
}
```

### 堆分配
根据 cmd/compile/internal/gc.state.stmt 方法对 defer 的处理我们可以看出，堆上分配的 runtime._defer 结构体是默认的兜底方案，当该方案被启用时，编译器会调用 cmd/compile/internal/gc.state.callResult 和 cmd/compile/internal/gc.state.call，这表示 defer 在编译器看来也是函数调用。

cmd/compile/internal/gc.state.call 会负责为所有函数和方法调用生成中间代码，它的工作包括以下内容：

获取需要执行的函数名、闭包指针、代码指针和函数调用的接收方；
获取栈地址并将函数或者方法的参数写入栈中；
使用 cmd/compile/internal/gc.state.newValue1A 以及相关函数生成函数调用的中间代码；
如果当前调用的函数是 defer，那么会单独生成相关的结束代码块；
获取函数的返回值地址并结束当前调用；
```golang
func (s *state) call(n *Node, k callKind, returnResultAddr bool) *ssa.Value {
	...
	var call *ssa.Value
	if k == callDeferStack {
		// 在栈上初始化 defer 结构体
		...
	} else {
		...
		switch {
		case k == callDefer:
			aux := ssa.StaticAuxCall(deferproc, ACArgs, ACResults)
			call = s.newValue1A(ssa.OpStaticCall, types.TypeMem, aux, s.mem())
		...
		}
		call.AuxInt = stksize
	}
	s.vars[&memVar] = call
	...
}
```

compile:
deferproc 让runtime的时候，_defer 结构体创建在heap上
将 defer 转换成了 runtime.deferproc，还在所有调用 defer 的函数结尾插入了 runtime.deferreturn

cmd/compile/internal/gc.walkstmt 在遇到 ODEFER 节点时会执行 Curfn.Func.SetHasDefer(true) 设置当前函数的 hasdefer 属性；
cmd/compile/internal/gc.buildssa 会执行 s.hasdefer = fn.Func.HasDefer() 更新 state 的 hasdefer；
cmd/compile/internal/gc.state.exit 会根据 state 的 hasdefer 在函数返回之前插入 runtime.deferreturn 的函数调用；

runtime:
从调度器的延迟调用缓存池 sched.deferpool 中取出结构体并将该结构体追加到当前 Goroutine 的缓存池中；
从 Goroutine 的延迟调用缓存池 pp.deferpool 中取出结构体；
通过 runtime.mallocgc 在堆上创建一个新的结构体；

runtime.deferreturn 会多次判断当前 Goroutine 的 _defer 链表中是否有未执行的结构体，该函数只有在所有延迟函数都执行后才会返回。

### stack 

```golang
func (s *state) call(n *Node, k callKind) *ssa.Value {
	...
	var call *ssa.Value
	if k == callDeferStack {
		// 在栈上创建 _defer 结构体
		t := deferstruct(stksize)
		...

		ACArgs = append(ACArgs, ssa.Param{Type: types.Types[TUINTPTR], Offset: int32(Ctxt.FixedFrameSize())})
		aux := ssa.StaticAuxCall(deferprocStack, ACArgs, ACResults) // 调用 deferprocStack
		arg0 := s.constOffPtrSP(types.Types[TUINTPTR], Ctxt.FixedFrameSize())
		s.store(types.Types[TUINTPTR], arg0, addr)
		call = s.newValue1A(ssa.OpStaticCall, types.TypeMem, aux, s.mem())
		call.AuxInt = stksize
	} else {
		...
	}
	s.vars[&memVar] = call
	...
}
```
在编译期间 就在stack上创建了_defer

所以在运行期间 runtime.deferprocStack 只需要设置一些未在编译期间初始化的字段，就可以将栈上的 runtime._defer 追加到函数的链表上

### open
Go 语言在 1.14 中通过开放编码（Open Coded）实现 defer 关键字，该设计使用代码内联优化 defer 关键的额外开销并引入函数数据 funcdata 管理 panic 的调用3，该优化可以将 defer 的调用开销从 1.13 版本的 ~35ns 降低至 ~6ns 左右

编译期间判断 defer 关键字、return 语句的个数确定是否开启开放编码优化；
通过 deferBits 和 cmd/compile/internal/gc.openDeferInfo 存储 defer 关键字的相关信息；
如果 defer 关键字的执行可以在编译期间确定，会在函数返回前直接插入相应的代码，否则会由运行时的 runtime.deferreturn 处理；
# gin & casbin





1. pod 重启 确认存在 
2. 