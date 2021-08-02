# cgroup
+ 查看支持的subsystem类型
    ```text
    [root@10 cgroup]# lssubsys -a
    cpuset
    cpu,cpuacct
    memory
    devices
    freezer
    net_cls,net_prio
    blkio
    perf_event
    hugetlb
    pids
    ```

+ 未关联subsystem的cgroup组：
    ```text
    mkdir cgroup-test
    sudo mount -t cgroup -o none,name=cgroup-test cgroup-test ./cgroup-test #挂载 一个 hierarchy
    ls ./cgroup test
    
    [cgroup-test] sudo mkdir cgroup-1 #创建子 cgroup ”cgroup-1”
    ```
  
+ subsystem路径  
在此路径下，均为可以进行关联的subsystem类型，与lssubsys结果相同
    ```text
    /sys/fs/cgroup/
    
    [root@10 cgroup]# ls
    blkio  cpuacct      cpuset   freezer  memory   net_cls,net_prio  perf_event  systemd
    cpu    cpu,cpuacct  devices  hugetlb  net_cls  net_prio          pids
    ```
  
+ docker新版本cgroup driver  
  较新版本docker使用cgroup driver是systemd。具体cgroup路径
  ```text
    /sys/fs/cgroup/memory/system.slice
  ```


