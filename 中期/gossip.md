# 目的

该算法主要解决的问题是，在GreatDB集群各节点间，通过ad-hoc、p2p网络，共享信息，并使数据传输时间最短，所需带宽最小。

Gossip传输的信息由key来识别，会被序列化成info对象。

单值info对象的值可以时任意类型。

一个Gossip实例维护一组info对象。单值info对象可以通过Gossip.AddInfo()方法添加。通过Gossip.GetInfo()，可以用key获取对应的info对象。


# 算法设计

Gossip协议是C/S架构。

Server端维护了一个数组，包含了一组已连接的节点。
Client端则是连接Server完成Info的发送和接收。


# 代码实现

整个Gossip协议的代码实现都放在同一个package中。

## Protobuf格式

Gossip算法传输信息是通过Protobuf框架。

```protobuf
// BootstrapInfo 包含了启动一个Gossip网络所必须的信息
message BootstrapInfo {
  // 集群里另一个节点的地址
  repeated util.UnresolvedAddr addresses = 1 [(gogoproto.nullable) = false];
  // 启动信息的时间戳
  util.hlc.Timestamp timestamp = 2 [(gogoproto.nullable) = false];
}

// Request 是Gossip RPC传输的Request结构体
message Request {
  // 发起请求的节点ID
  int32 node_id = 1 [(gogoproto.customname) = "NodeID",
      (gogoproto.casttype) = "github.com/cockroachdb/cockroach/pkg/roachpb.NodeID"];
  // 发起请求的客户端地址
  util.UnresolvedAddr addr = 2 [(gogoproto.nullable) = false];
  // 一组发起请求方所见到的，由其他节点生成的时间戳。
  map<int32, int64> high_water_stamps = 3 [(gogoproto.castkey) = "github.com/cockroachdb/cockroach/pkg/roachpb.NodeID", (gogoproto.nullable) = false];
  // Delta 由发送者生成.
  map<string, Info> delta = 4;
  // Cluster ID 提供了有一个合法的连接
  bytes cluster_id = 5 [(gogoproto.nullable) = false,
                        (gogoproto.customname) = "ClusterID",
                        (gogoproto.customtype) = "github.com/cockroachdb/cockroach/pkg/util/uuid.UUID"];
}

// Response由Gossip.Gossip RPC返回
message Response {
  // 正在回应的节点ID.
  int32 node_id = 1 [(gogoproto.customname) = "NodeID",
      (gogoproto.casttype) = "github.com/cockroachdb/cockroach/pkg/roachpb.NodeID"];
  // 正在回应的客户端地址
  util.UnresolvedAddr addr = 2 [(gogoproto.nullable) = false];
  util.UnresolvedAddr alternate_addr = 3;
  int32 alternate_node_id = 4 [(gogoproto.customname) = "AlternateNodeID",
      (gogoproto.casttype) = "github.com/cockroachdb/cockroach/pkg/roachpb.NodeID"];
  map<string, Info> delta = 5;
  // 一组发起回应方所见到的，由其他节点生成的时间戳。
  map<int32, int64> high_water_stamps = 6 [(gogoproto.castkey) = "github.com/cockroachdb/cockroach/pkg/roachpb.NodeID", (gogoproto.nullable) = false];
}

// InfoStatus 包含了当前infoStore的信息 
message InfoStatus {
  map<string, Info> infos = 1 [(gogoproto.nullable) = false];
}

// Info 是Gossip网络传输的基本单位
message Info {
  roachpb.Value value = 1 [(gogoproto.nullable) = false];
  int64 orig_stamp = 2;
  int64 ttl_stamp = 3 [(gogoproto.customname) = "TTLStamp"];
  uint32 hops = 4;
  int32 node_id = 5 [(gogoproto.customname) = "NodeID",
      (gogoproto.casttype) = "github.com/cockroachdb/cockroach/pkg/roachpb.NodeID"];
  int32 peer_id = 6 [(gogoproto.customname) = "PeerID",
      (gogoproto.casttype) = "github.com/cockroachdb/cockroach/pkg/roachpb.NodeID"];
}

// Gossip服务
service Gossip {
  rpc Gossip (stream Request) returns (stream Response) {}
}
```