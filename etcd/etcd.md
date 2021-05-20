# etcd源码学习

## 启动配置
启动一个etcd的典型配置为
```text
cat << EOF | tee /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.0.52.13:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.0.52.13:2379,http://127.0.0.1:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.52.13:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.0.52.13:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://10.0.52.13:2380,etcd02=https://10.0.52.14:2380,etcd03=https://10.0.52.6:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

#[Security]
ETCD_CERT_FILE="/opt/etcd/ssl/server.pem"
ETCD_KEY_FILE="/opt/etcd/ssl/server-key.pem"
ETCD_TRUSTED_CA_FILE="/opt/etcd/ssl/ca.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_PEER_CERT_FILE="/opt/etcd/ssl/server.pem"
ETCD_PEER_KEY_FILE="/opt/etcd/ssl/server-key.pem"
ETCD_PEER_TRUSTED_CA_FILE="/opt/etcd/ssl/ca.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
EOF
```
需要注意的是：
+ ETCD_INITIAL_CLUSTER需要指定集群中的所有client，`包括自己`
+ ETCD_INITIAL_CLUSTER_TOKEN是指集群名称，全集群一样

## 预先了解
首先明确一下etcd的raft发送appendMsg等的方式。  
raft结构体下有msgs []pb.Message数据结构。用于存储raft需要向外发送的消息。  
<details>
<summary><mark><font color="darkred">pb.Message数据结构</font></mark></summary>
<p></p>
<pre><code>
type Message struct {
	Type             MessageType `protobuf:"varint,1,opt,name=type,enum=raftpb.MessageType" json:"type"`
	To               uint64      `protobuf:"varint,2,opt,name=to" json:"to"`
	From             uint64      `protobuf:"varint,3,opt,name=from" json:"from"`
	Term             uint64      `protobuf:"varint,4,opt,name=term" json:"term"`
	LogTerm          uint64      `protobuf:"varint,5,opt,name=logTerm" json:"logTerm"`
	Index            uint64      `protobuf:"varint,6,opt,name=index" json:"index"`
	Entries          []Entry     `protobuf:"bytes,7,rep,name=entries" json:"entries"`
	Commit           uint64      `protobuf:"varint,8,opt,name=commit" json:"commit"`
	Snapshot         Snapshot    `protobuf:"bytes,9,opt,name=snapshot" json:"snapshot"`
	Reject           bool        `protobuf:"varint,10,opt,name=reject" json:"reject"`
	RejectHint       uint64      `protobuf:"varint,11,opt,name=rejectHint" json:"rejectHint"`
	Context          []byte      `protobuf:"bytes,12,opt,name=context" json:"context,omitempty"`
	XXX_unrecognized []byte      `json:"-"`
}
</code></pre>
</details>
其中from、to是int型，用于保存该消息的的发送者、接收者的id。

## 启动流程
### member id
前面介绍了raft使用member id作为消息标识。member id则是在初始化流程中生成  
```text
etcdserver.NewServer(srvcfg)函数中，调用  
membership.NewClusterFromURLsMap(cfg.Logger, cfg.InitialClusterToken, cfg.InitialPeerURLsMap)

在该函数中，通过读取etcd的clustering配置文件的ETCD_INITIAL_CLUSTER参数，hash生成member id

具体的生成方式为：node连接地址 + ETCD_INITIAL_CLUSTER_TOKEN hash后即为member id
此时尚未与其他node建立连接。
```

除了每个node存在member id外，使用该集群的所有member id为cluster生成了`cluster id`  
即member id处理后hash  

由于集群内的 `ETCD_INITIAL_CLUSTER`与`ETCD_INITIAL_CLUSTER_TOKEN`要求完全相同，则所有raft node能保证相同的member id与cluster id  

### 建立集群间连接
peer listener、client listener
