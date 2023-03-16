### Useful Linux
- 检查打开文件、端口
    ```shell
    lsof -i tcp # root priv may need 
    lsof -i :80
    fuser -v -n tcp 8002
    netstat -lnt(tcp) -lnu
    ```

- 修改网卡中断数
    ```shell
    ethtool -l|--show-channels devname

    ethtool -L|--set-channels devname [rx N] [tx N] [other N]
            [combined N]
            
    -l --show-channels
            Queries the specified network device for the numbers of
            channels it has.  A channel is an IRQ and the set of queues
            that can trigger that IRQ.

    -L --set-channels
            Changes the numbers of channels of the specified network
            device.

        rx N   Changes the number of channels with only receive queues.

        tx N   Changes the number of channels with only transmit queues.

        other N
                Changes the number of channels used only for other
                purposes e.g. link interrupts or SR-IOV co-ordination.

        combined N
                Changes the number of multi-purpose channels.
    ```

- 设置 CPU frequency
    ```shell
    cpupower frequency-set -g performance 
    cpupower frequency-set -u 3.4GHz  
    cpupower frequency-set -d 3.4GHz 
    cpupower frequency-info
    ```

- 设置大页内存与清理缓存

    ```shell
    echo 4096 | sudo tee /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
    or 
    /sys/device/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages
    ### 清理缓存
    To free pagecache, dentries and inodes:
    echo 3 > /proc/sys/vm/drop_caches
    ```

### system logs 
```shell
    dmesg

    cat /proc/sys/kernel/core_pattern
    vim /etc/sysctl.conf
添加一行：kernel.core_pattern=core-%e-%p-%t
ulimit -c
```

### sar usage
```shell
    sar -m CPU -P <core> 1 1000  # 指定core的频率抓取
    sar -P 0 1 100  # 指定core的cpu使用率抓取
    sar -u 1 600 > CPU_UTIL & # 全核使用率
    sar -m CPU -u 1 600 > CPU_FREQ &  # 指全核频率
```
### Shell/bash script