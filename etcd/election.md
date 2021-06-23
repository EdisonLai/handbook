# etcd实现northd主备选举

## 概述
业务场景：单进程任务使用主备部署提高可用性。
使用etcd封装的election能够轻松实现，且etcd的revision机制能够实现备进程按序升主，避免惊群。

## 原理
+ lease租约
+ revision单调递增
+ prefix get
+ watch机制

## demo地址
https://gist.github.com/EdisonLai/d3da19eb5d10c6260c91a59e6f645c53

### lease租约
设置ttl并设置租约。
1. 使用keepalive进行租约保活
2. etcd key在put时，可以关联租约，租约到期时，key同样删除

### revision
etcd维护一个全局单调递增的revision用于保存log的版本号。
1. 支持cmp revision操作。当key的版本号等于0，代表该key未创建。

### prefix get
etcd支持以 prefix 作为前缀获取所有该前缀的key-value
1. etcd所有的key是按续排列的
2. etcd会将 prefix 转换为 prefix - prefix0的范围，获取该范围内的所有key
3. 前缀获取支持参数：v3.WithFirstCreate(), v3.WithLastCreate(), v3.WithMaxCreateRev(maxCreateRev)

### watch机制
客户端可以监听一个key的删除

## 实现细节
1、申请lease获得返回的lease id。  
2、put key: /{electionName}/leaseID  value: candidateName，并与lease绑定  
3、get prefix(/{electionName}) v3.WithFirstCreate()，得到revision最小的 candidateName-Min  
4、若 candidateName-Min 与自己相同，则获取分布式锁成功。  
  
5、若 candidateName-Min 与自己不同，则get prefix(/{electionName}) v3.WithLastCreate(), v3.WithMaxCreateRev(maxCreateRev)  
6、第五步得到比自己创建的key小的最大revision的 key 假定名字为 key-last 。  
7、开始watch key-last
8、当 key-last 删除时，get prefix(/{electionName}) v3.WithLastCreate(), v3.WithMaxCreateRev(maxCreateRev)
9、判断第8步是否获取到值，若无，则本机为leader（最小revision的key）
