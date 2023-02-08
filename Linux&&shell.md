### Useful Linux
- 检查打开文件、端口
    ```shell
    lsof -i tcp # root priv may need 
    fuser
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
### Shell/bash script