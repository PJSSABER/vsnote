# K8S
## Using K8S in new user
    ```shell
    a.  $ mkdir -p $HOME/.kube
    b.  $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    c.  $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

- 获取Pod 中的进程PID (K8S+docker为例)
    ```shell
    kubectl -n test describe pod debug-685b48bcf5-ggn5d
    #get docker id
    Container ID:   docker://e64939086488a9302821566b0c1f193b755c805f5ff5370d5ce5e6f154ffc648 
    #get runtime pid
    docker inspect e64939086488a9302821566b0c1f193b755c805f5ff5370d5ce5e6f154ffc648 | grep -i pid
    ```

## 删除namespace卡住
可以检查一下apiservice,如果有available是false的，删掉
    ```shell
    kubectl get apiservice
    ```

## 重启K8Spod
    ```shell
    #1.
    kubectl get pod {podname} -n {namespace} -o yaml | kubectl replace --force -f -
    #2.如果是deployment对象，将ReplicaSet 的数量 scale 到 0，然后又 scale 到 1，那么 Pod 也就重启了
    kubectl scale deployment esb-admin --replicas=0 -n {namespace}
    kubectl scale deployment esb-admin --replicas=1 -n {namespace}
    ```
## taint node
    ```shell
    #taint
    kubectl taint node wsf-spr4-sh node-role.kubernetes.io/master:NoSchedule
    # untaint
    kubectl taint node wsf-spr4-sh node-role.kubernetes.io/master:NoSchedule-
    ```
## K8S Qos
    Guaranteed：Pod 中每个容器都必须有内存/CPU 限制和请求，而且值必须相等。如果一个容器只指明limit而未设定request，则request的值等于limit值。
    Burstable：Pod 至少有一个容器有内存或者 CPU 请求且不满足 Guarantee 等级的要求，即内存/CPU 的值设置的不同。
    BestEffort：容器必须没有任何内存或者 CPU 的限制或请求。
    Guaranteed --> Burstable --> BestEffort

## 获取POD/deployment/service的yaml
    ```shell
        kubectl get deploy deploymentname -o yaml 
    ```

## 重启calico
    ```shell
        kubectl delete -f calico.yaml && kubectl apply -f calico.yaml 
    ```

## 删除节点 - 重新加入节点
```shell 
### 删除节点
# 设置为维护模式 在master操作
kubectl drain <node-to-delete> --delete-local-data --force --ignore-daemonsets node/<node-to-delete>

# 删除节点 在master操作
kubectl delete node <node-to-delete>

### node 节点重新加入集群
# master节点生成token
kubeadm token create --print-join-command

# optional : 在node 节点 删除之前相关文件
sudo kubeadm reset
systemctl stop kubelet
sudo rm -rf /etc/kubernetes/*

# 重新加入集群， 复制master生成的token： <token>
kubeadm join ×.×.×.×:6443 --token <token>
```

## k8s 1.2.4版本后自行管理container的工具
```shell
# show images
    crictl images
```

## name service not known
kubectl get svc <service-name> -o wide

## 重装k8s

### issue Conflicting values set for option Signed-By regarding source http://apt.kubernetes.io/ kubernetes-xenial: /usr/share/keyrings/k8s.gpg !=
- /usr/share/keyrings/k8s.gpg should match /etc/apt/sources.list.d/apt_kubernetes_io.list
  try apt-get update to check if fixed. 
- copy from other setups

# docker

## 修改dockers配置\ insecure registry
    ```shell
    vim /etc/docker/daemon.json
    ```
    
## containerd insecure registry
    ``` shell
    # 
    # change <IP>:5000 to your registry url
    sudo vim /etc/containerd/config.toml
    [plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."<IP>:5000"]
        endpoint = ["http://<IP>:5000"]
    [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."<IP>:5000".tls]
        insecure_skip_verify = true
    ```      

        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."10.112.107.160:30003"]
        endpoint = ["http://10.112.107.160:30003"]

## cp from container to host
    ```shell
    docker cp $container-name:$path $host_path
    ```
## docker run option
1. --pid=host  
Sets the PID mode to the host PID mode. This turns on sharing between container and the host operating system the PID address space. Containers launched with this flag can access and manipulate other containers in the bare-metal machine’s nam/etc/containerd/config.tomlespace and vice versa.

## 修改dockers tmp文件夹
    ```shell 
    在/etc/docker/daemon.json中增加一行 "graph": "/home/docker_data";systemctl restart docker
    ```

## clear docker
-  clear the disk space used by containers: docker container prune -f
-  remove unused images from the system(dangling): docker image prune -f
-  volume: docker volume prune -f
-  build cache: docker buildx prune -f
-  network: docker network prune -f
-  all: docker system prune 
