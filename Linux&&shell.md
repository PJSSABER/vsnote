# Emon/Vtune
直接工作在硬件层上的一层很薄的Driver ->  在workload 运行时，监测cpu上各个counter 的数据 -> 汇总出excel报告
Emon 通过读取CPU内部的Monitor来获取当前CPU的性能参数，因此emon对性能的影响很小。

## 关键概念
- Performance counter
Hardware register used to count a performance event 计数的硬件

- Event
Measurable occurrence or condition in the CPU cpu事件
例如
L2 misses
L2 references
Instruction retired

- Metric
Calculation of from >= 2 measured performance events 即计算出来的一些性能指标
例如
L2 Miss ratio
CPI
GB/s throughput
Pathlength

Emon将各个counter采集到的值作为输入，根据Emon的公式对metric进行计算
各个采样点的值可以直接使用，不需要与上一采样点的数据进行联合计算

## Emon 数据
CPU 执行流程

!["2333"](/images/TopDown.png)

## 重要metric
根据看 system socket core view, 统计的角度不同
- cpu utilization
- CPI clock per instruction, 平均每条指令的周期数
- IPC instruction per clock， 每个周期的运行指令数
- MIPS 每秒执行百万条指令数, = freq * IPC / 1E6
- MPI miss per instruction
- Numa Local Dram 读取Numa 近端内存 与 远端内存的比例
- Memory Bandwith & latency 内存速率与延迟
- Io 读写latency metric_IO_bandwidth_disk_or_network_writes 
- metric_TMA_Frontend_Bound(%) 前端bound，取指操作
- metric_TMA_Backend_Bound(%) 后端bound, 计算操作
    - metric_TMA_..Core_Bound(%)
    - metric_TMA_..Memory_Bound(%)
        - L1 cache, L2 cache, L3 cache, DRAM...
- metric_branch mispredict ratio 分支预测失误
- INST_RETIRED.ANY retire的instruction(即最终成功执行的instruction) Pathlength = INST_RETIRED.ANY*execution_time/3600
    
## TMA 分析

Top-down Microarchitecture Analysis (TMA) Method 
结合Emon/VTUNE, 从顶置下的进行分析

- TMA 分层， 可能根据 metric_TMA_Metrics_Version 的不同而改变，但主体基本一致

- CPU Bound = FrontendBound(取指) + BadSpeculation(预测错误&重装指令) + Retiring(正确执行消耗) + BackendBound(内存读写 & core调度)


## IO die 与 compute die
1. IO Die Frequency (Uncore Frequency):
The IO Die (input/output die) typically includes components like memory controllers, PCIe controllers, and other IO interfaces.
Its frequency would generally be considered part of the uncore frequency because it handles functions outside of the direct computation performed by the CPU cores.
The uncore frequency impacts how quickly data can be moved between the CPU cores and external components (like memory and peripherals). So, a higher IO die frequency can improve the performance of memory access, IO operations, and communication with other devices.
2. Compute Die Frequency (Core Frequency):
The compute die contains the actual CPU cores that execute instructions.
Its frequency is directly related to the core frequency, which governs the speed at which the cores execute tasks.
This frequency affects how fast the processor can carry out computations, like processing a loop, running an algorithm, or performing calculations.
In a multi-die architecture like AMD’s EPYC processors or some Intel chips, these distinctions become more pronounced:

Compute die focuses on computation and the core frequency defines how quickly the CPU can perform those calculations.
IO die focuses on interfacing with external devices and memory, and its speed is tied to the uncore frequency.
Understanding this separation helps in fine-tuning performance for workloads that are either compute-bound (requiring higher core frequency) or memory/IO-bound (benefiting from a higher uncore frequency).

# Useful Linux
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
    sudo cpupower frequency-set -g performance 
    cpupower frequency-set -u 3.4GHz  
    cpupower frequency-set -d 3.4GHz 
    cpupower frequency-info
    cpupower idle-set -D 3 // disable C3 and Higer

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

# system logs 
```shell
    dmesg

    cat /proc/sys/kernel/core_pattern
    vim /etc/sysctl.conf
添加一行：kernel.core_pattern=core-%e-%p-%t
ulimit -c
```

# sar usage
```shell
    sar -m CPU -P <core> 1 1000  # 指定core的频率抓取
    sar -P 0 1 100  # 指定core的cpu使用率抓取
    sar -P ALL 1 100  # 指定core的cpu使用率抓取
    sar -u 1 600 > CPU_UTIL & # 全核使用率
    sar -m CPU -u 1 600 > CPU_FREQ &  # 指全核频率
```
# Shell/bash script


sudo cpupower frequency-set -u 2.0GHz  
sudo cpupower frequency-set -d 2.0GHz 

# << && <<<

## << 
here document, allows you to provide a block of text to a command, treating the block as if it were coming from a file or standard input

```shell
command << EOF
line1
line2
line3
EOF
```

## <<< 
here string, allows you to pass a string directly to a command as standard input

```shell
grep "pattern" <<< "search this string for the pattern"
```

##
if what's inside there is a carriage and return, should use << to treat as a doc.

# 检查是否为file文件

```shell

#### -d
if [ -e "path/to/target" ] && [ ! -d "path/to/target" ]; then
  echo "It is a file"
else
  echo "It is not a file"
fi

#### stat 查看 metadata
if stat --printf="%F\n" "path/to/target" | grep -q "regular file"; then
  echo "It is a file"
else
  echo "It is not a file"
fi

#### file
if file "path/to/target" | grep -q "regular file"; then
  echo "It is a file"
else
  echo "It is not a file"
fi
```


