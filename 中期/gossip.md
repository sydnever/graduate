# 目的

该算法主要解决的问题是，在GreatDB集群各节点间，通过ad-hoc、p2p网络，共享信息，并使数据传输时间最短，所需带宽最小。

Gossip传输的信息由key来识别，会被序列化成info对象。

单值info对象的值可以时任意类型。

一个Gossip实例维护一组info对象。单值info对象可以通过Gossip.AddInfo()方法添加。通过Gossip.GetInfo()，可以用key获取对应的info对象。


# 算法设计

Gossip协议是对等形式的C/S架构。

Server端维护了一个数组，包含了一组已连接的节点。

Client端则是连接Server完成Info的发送和接收。

每个节点尝试用最小的总跳数与对等节点联系，以收集系统中的所有信息。具体算法如下：

0. 启动：每个节点启动Gossip Server来接收gossip请求。

1. 加入Gossip网络：节点从启动列表里随即选择一个对等节点，来进行第一次对外连接，同时启动Gossip Client。

2. 节点向对等节点发送Gossip请求。在Gossip请求与回应中，都包含一组从节点ID到Info对象的映射，涵盖了在网络中的其他节点。每个节点都维护了自己和其他对等节点的映射。Info对象中包括最近生成Info对象的时间戳，和到达某节点的最少转发数。当请求节点经过了checkInterval长的时间，而没能收到回应，就会丢出超时异常，然后Client关闭并进行垃圾回收。如果节点没有对外连接，则会返回步骤1 。
    
   a. 当Gossip请求和回应被接收，相应的infostore会进行扩张。当新的Info对象被接收，在这次请求和回应中涉及的Client会被记录。如果节点没有对外连接，返回步骤1 。

   b. 如果Gossip报文被接收时，超过了最大转发数maxHops或者已连接的对等节点小于maxPeers()，则会随即选择对等节点启动，然后跳转到步骤2 。

   c. 如果KeySentinel标记的gossip丢失或者过期了，节点则被视为发生了脑裂，跳转到步骤1 。

3. 在连接状态下，如果节点有太多已连接客户端，gossip请求会立刻返回到一个随即选择的Client，并带有一组替换地址。


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
      (gogoproto.casttype) = "./pkg/roachpb.NodeID"];
  // 发起请求的客户端地址
  util.UnresolvedAddr addr = 2 [(gogoproto.nullable) = false];
  // 一组发起请求方所见到的，由其他节点生成的时间戳。
  map<int32, int64> high_water_stamps = 3 [(gogoproto.castkey) = "./pkg/roachpb.NodeID", (gogoproto.nullable) = false];
  // Delta 由发送者生成.
  map<string, Info> delta = 4;
  // Cluster ID 提供了有一个合法的连接
  bytes cluster_id = 5 [(gogoproto.nullable) = false,
                        (gogoproto.customname) = "ClusterID",
                        (gogoproto.customtype) = "./pkg/util/uuid.UUID"];
}

// Response由Gossip.Gossip RPC返回
message Response {
  // 正在回应的节点ID.
  int32 node_id = 1 [(gogoproto.customname) = "NodeID",
      (gogoproto.casttype) = "./pkg/roachpb.NodeID"];
  // 正在回应的客户端地址
  util.UnresolvedAddr addr = 2 [(gogoproto.nullable) = false];
  util.UnresolvedAddr alternate_addr = 3;
  int32 alternate_node_id = 4 [(gogoproto.customname) = "AlternateNodeID",
      (gogoproto.casttype) = "./pkg/roachpb.NodeID"];
  map<string, Info> delta = 5;
  // 一组发起回应方所见到的，由其他节点生成的时间戳。
  map<int32, int64> high_water_stamps = 6 [(gogoproto.castkey) = "./pkg/roachpb.NodeID", (gogoproto.nullable) = false];
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
      (gogoproto.casttype) = "./pkg/roachpb.NodeID"];
  int32 peer_id = 6 [(gogoproto.customname) = "PeerID",
      (gogoproto.casttype) = "./pkg/roachpb.NodeID"];
}

// Gossip服务
service Gossip {
  rpc Gossip (stream Request) returns (stream Response) {}
}
```

## gossip.go

gossip.go文件是实现gossip算法的主体。

严格来说，go语言并不是面向对象语言，在go语言中只有type而没有class，只有组合而没有继承，所以以下代码实现的描述，尽量不会使用面向对象的术语。

### type Gossip

Gossip类型是一个gossip节点的实例，其中内嵌了一个gossip server。在启动期间，启动列表包含了进入gossip网络的候选者。

代码摘要如下：

```go
type Gossip struct {
	*server // 内嵌的gossip RPC server

  Connected     chan struct{}   
  ...   
	hasCleanedBS  bool

  
	clientsMu struct {// 客户端互斥体
		syncutil.Mutex
		clients []*client
		breakers map[string]*circuit.Breaker
	}

	disconnected chan *client  
	stalled      bool          
	stalledCh    chan struct{} 

	stallInterval     time.Duration
	bootstrapInterval time.Duration
	cullInterval      time.Duration

	systemConfig         config.SystemConfig
	systemConfigSet      bool
	systemConfigMu       syncutil.RWMutex
	systemConfigChannels []chan<- struct{}

	resolverIdx    int
	resolvers      []resolver.Resolver
	resolversTried map[int]struct{} 
	nodeDescs      map[roachpb.NodeID]*roachpb.NodeDescriptor
  storeMap map[roachpb.StoreID]roachpb.NodeID
  
	resolverAddrs  map[util.UnresolvedAddr]resolver.Resolver
	bootstrapAddrs map[util.UnresolvedAddr]roachpb.NodeID
}
```

### 函数New()

这个函数用于创建一个gossip节点的实例，在更高层面上管理ClusterIDContainer和 NodeIDContainer实例。

代码概要如下：

```go
func New(
	ambient log.AmbientContext,
	clusterID *base.ClusterIDContainer,
	nodeID *base.NodeIDContainer,
	rpcContext *rpc.Context,
	grpcServer *grpc.Server,
	stopper *stop.Stopper,
	registry *metric.Registry,
) *Gossip {
	ambient.SetEventLog("gossip", "gossip")
	g := &Gossip{
		server:            newServer(ambient, clusterID, nodeID, stopper, registry),
		Connected:         make(chan struct{}),
		rpcContext:        rpcContext,
		outgoing:          makeNodeSet(minPeers, metric.NewGauge(MetaConnectionsOutgoingGauge)),
		bootstrapping:     map[string]struct{}{},
		disconnected:      make(chan *client, 10),
		stalledCh:         make(chan struct{}, 1),
		stallInterval:     defaultStallInterval,
		bootstrapInterval: defaultBootstrapInterval,
		cullInterval:      defaultCullInterval,
		resolversTried:    map[int]struct{}{},
		nodeDescs:         map[roachpb.NodeID]*roachpb.NodeDescriptor{},
		storeMap:          make(map[roachpb.StoreID]roachpb.NodeID),
		resolverAddrs:     map[util.UnresolvedAddr]resolver.Resolver{},
		bootstrapAddrs:    map[util.UnresolvedAddr]roachpb.NodeID{},
	}
	stopper.AddCloser(stop.CloserFn(g.server.AmbientContext.FinishEventLog))

	registry.AddMetric(g.outgoing.gauge)
	g.clientsMu.breakers = map[string]*circuit.Breaker{}

	g.mu.Lock()
	g.mu.is.registerCallback(KeySystemConfig, g.updateSystemConfig)
	g.mu.is.registerCallback(MakePrefixPattern(KeyNodeIDPrefix), g.updateNodeAddress)
	g.mu.is.registerCallback(MakePrefixPattern(KeyStorePrefix), g.updateStoreMap)
	g.mu.Unlock()

	if grpcServer != nil {
		RegisterGossipServer(grpcServer, g.server)
	}
	return g
}
```
### 函数NewTest()

New函数的单元测试函数。

### 函数GetNodeMetrics()

调用底层API获取节点的各种相关信息。

### 函数SetNodeDescriptor()
在gossip网络中添加节点解释器。

### 其他简单工具函数


## server.go
gossip server的具体代码实现。

### type server

Server端维护了一个数组，包含了一组已连接的节点。

```go
type server struct {
	log.AmbientContext

	clusterID *base.ClusterIDContainer
	NodeID    *base.NodeIDContainer

	stopper *stop.Stopper

	mu struct {
		syncutil.Mutex
		is       *infoStore                         
		incoming nodeSet                            
		nodeMap  map[util.UnresolvedAddr]serverInfo 
		ready chan struct{}
	}
	tighten chan struct{} 

	nodeMetrics   Metrics
	serverMetrics Metrics

	simulationCycler *sync.Cond
}
```

### 函数NewServer()

这个函数负责创建server的实例。
```go
func newServer(
	ambient log.AmbientContext,
	clusterID *base.ClusterIDContainer,
	nodeID *base.NodeIDContainer,
	stopper *stop.Stopper,
	registry *metric.Registry,
) *server {
	s := &server{
		AmbientContext: ambient,
		clusterID:      clusterID,
		NodeID:         nodeID,
		stopper:        stopper,
		tighten:        make(chan struct{}, 1),
		nodeMetrics:    makeMetrics(),
		serverMetrics:  makeMetrics(),
	}

	s.mu.is = newInfoStore(s.AmbientContext, nodeID, util.UnresolvedAddr{}, stopper)
	s.mu.incoming = makeNodeSet(minPeers, metric.NewGauge(MetaConnectionsIncomingGauge))
	s.mu.nodeMap = make(map[util.UnresolvedAddr]serverInfo)
	s.mu.ready = make(chan struct{})

	registry.AddMetric(s.mu.incoming.gauge)
	registry.AddMetricStruct(s.nodeMetrics)

	return s
}
```

### 函数Gossip()
该函数负责接收对等节点发来的gossip信息。
```go
func (s *server) Gossip(stream Gossip_GossipServer) error {
	args, err := stream.Recv()
	if err != nil {
		return err
	}
	if (args.ClusterID != uuid.UUID{}) && args.ClusterID != s.clusterID.Get() {
		return errors.Errorf("gossip connection refused from different cluster %s", args.ClusterID)
	}

	ctx, cancel := context.WithCancel(s.AnnotateCtx(stream.Context()))
	defer cancel()
	syncChan := make(chan struct{}, 1)
	send := func(reply *Response) error {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case syncChan <- struct{}{}:
			defer func() { <-syncChan }()

			bytesSent := int64(reply.Size())
			infoCount := int64(len(reply.Delta))
			s.nodeMetrics.BytesSent.Inc(bytesSent)
			s.nodeMetrics.InfosSent.Inc(infoCount)
			s.serverMetrics.BytesSent.Inc(bytesSent)
			s.serverMetrics.InfosSent.Inc(infoCount)

			return stream.Send(reply)
		}
	}

	defer func() { syncChan <- struct{}{} }()

	errCh := make(chan error, 1)

	if err := s.stopper.RunTask(ctx, "gossip.server: receiver", func(ctx context.Context) {
		s.stopper.RunWorker(ctx, func(ctx context.Context) {
			errCh <- s.gossipReceiver(ctx, &args, send, stream.Recv)
		})
	}); err != nil {
		return err
	}

	reply := new(Response)

	for {
		s.mu.Lock()
		ready := s.mu.ready
		delta := s.mu.is.delta(args.HighWaterStamps)

		if infoCount := len(delta); infoCount > 0 {
			if log.V(1) {
				log.Infof(ctx, "returning %d info(s) to node %d: %s",
					infoCount, args.NodeID, extractKeys(delta))
			}

			*reply = Response{
				NodeID:          s.NodeID.Get(),
				HighWaterStamps: s.mu.is.getHighWaterStamps(),
				Delta:           delta,
			}

			s.mu.Unlock()
			if err := send(reply); err != nil {
				return err
			}
			s.mu.Lock()
		}

		s.mu.Unlock()

		select {
		case <-s.stopper.ShouldQuiesce():
			return nil
		case err := <-errCh:
			return err
		case <-ready:
		}
	}
}
```
### 其他简单工具函数

## client.go 

Client端则是连接Server完成Info的发送和接收。


### type client
client实例

```go
type client struct {
	log.AmbientContext

	createdAt             time.Time
	peerID                roachpb.NodeID           
	resolvedPlaceholder   bool                     
	addr                  net.Addr                 
	forwardAddr           *util.UnresolvedAddr     
	remoteHighWaterStamps map[roachpb.NodeID]int64 
	closer                chan struct{}            
	clientMetrics         Metrics
	nodeMetrics           Metrics
}

func extractKeys(delta map[string]*Info) string {
	keys := make([]string, 0, len(delta))
	for key := range delta {
		keys = append(keys, key)
	}
	return fmt.Sprintf("%s", keys)
}

func newClient(ambient log.AmbientContext, addr net.Addr, nodeMetrics Metrics) *client {
	return &client{
		AmbientContext: ambient,
		createdAt:      timeutil.Now(),
		addr:           addr,
		remoteHighWaterStamps: map[roachpb.NodeID]int64{},
		closer:                make(chan struct{}),
		clientMetrics:         makeMetrics(),
		nodeMetrics:           nodeMetrics,
	}
}
```

### 函数newClient()
负责创建Client实例。

### 函数startLocked()
负责启动远程连接，并在启动后开始gossip。

### 函数close()
关闭client，终止gossip循环。

### 其他工具函数

## *_test.go

各种功能模块的单元测试，如：client_test.go，gossip_test.go。

## info.go

info对象的具体实现。

## util.go

一些零散的难以归类的工具函数。