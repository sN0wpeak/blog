---
title: Sarama消费流程
date: 2022-03-30 21:40:30
tags:
---
```go
type consumerGroup struct {
	client Client // client struct

	config   *Config
	consumer Consumer // consumer struct
	groupID  string
	memberID string
	errors   chan error

	lock      sync.Mutex
	closed    chan none
	closeOnce sync.Once

	userData []byte
}

type consumer struct {
	conf            *Config
	children        map[string]map[int32]*partitionConsumer
	brokerConsumers map[*Broker]*brokerConsumer
	client          Client
	lock            sync.Mutex
}

type client struct {
	conf           *Config
	closer, closed chan none // for shutting down background metadata updater

	// the broker addresses given to us through the constructor are not guaranteed to be returned in
	// the cluster metadata (I *think* it only returns brokers who are currently leading partitions?)
	// so we store them separately
	seedBrokers []*Broker
	deadSeeds   []*Broker

	controllerID   int32                                   // cluster controller broker id
	brokers        map[int32]*Broker                       // maps broker ids to brokers
	metadata       map[string]map[int32]*PartitionMetadata // maps topics to partition ids to metadata
	metadataTopics map[string]none                         // topics that need to collect metadata
	coordinators   map[string]int32                        // Maps consumer group names to coordinating broker IDs

	// If the number of partitions is large, we can get some churn calling cachedPartitions,
	// so the result is cached.  It is important to update this value whenever metadata is changed
	cachedPartitionsResults map[string][maxPartitionIndex][]int32

	lock sync.RWMutex // protects access to the maps that hold cluster state.
}


type consumerGroupSession struct {
	parent       *consumerGroup
	memberID     string
	generationID int32
	handler      ConsumerGroupHandler

	claims  map[string][]int32
	offsets *offsetManager
	ctx     context.Context
	cancel  func()

	waitGroup       sync.WaitGroup
	releaseOnce     sync.Once
	hbDying, hbDead chan none // 控制心跳结束
}



type partitionOffsetManager struct {
	parent    *offsetManager
	topic     string
	partition int32

	lock     sync.Mutex
	offset   int64
	metadata string
	dirty    bool
	done     bool

	releaseOnce sync.Once
	errors      chan *ConsumerError
}


type partitionConsumer struct {
	highWaterMarkOffset int64 // must be at the top of the struct because https://golang.org/pkg/sync/atomic/#pkg-note-BUG

	consumer *consumer
	conf     *Config
	broker   *brokerConsumer // 用于向broker拉取数据
	messages chan *ConsumerMessage // responseFeeder将feeder处理为messages，然后就可以消费了. 
	errors   chan *ConsumerError
	feeder   chan *FetchResponse // 在brokerConsumer.subscriptionConsumer向broker拉取消息，然后由partitionConsumer.responseFeeder进行处理

	preferredReadReplica int32 // 

	trigger, dying chan none
	closeOnce      sync.Once
	topic          string
	partition      int32
	responseResult error
	fetchSize      int32
	offset         int64
	retries        int32

	paused int32
}



type brokerConsumer struct {
	consumer         *consumer
	broker           *Broker
	input            chan *partitionConsumer
	newSubscriptions chan []*partitionConsumer
	subscriptions    map[*partitionConsumer]none
	wait             chan none
	acks             sync.WaitGroup
	refs             int
}

type consumerGroupClaim struct {
	topic     string
	partition int32
	offset    int64
	PartitionConsumer
}
```

```go

for {
	// `Consume` should be called inside an infinite loop, when a
	// server-side rebalance happens, the consumer session will need to be
	// recreated to get the new claims
	client.Consume(ctx, strings.Split(topics, ","), &consumer)
		// Refresh metadata for requested topics
		c.client.RefreshMetadata(topics...)
		// Init session
		sess, err := c.newSession(ctx, topics, handler, c.config.Consumer.Group.Rebalance.Retry.Max)
			coordinator, err := c.client.Coordinator(c.groupID)
			// Join consumer group
			join, err := c.joinGroupRequest(coordinator, topics)
			switch join.Err // 出错重试
			// Prepare distribution plan if we joined as the leader
			if join.LeaderId == join.MemberId {
				members, err := join.GetMembers()
				plan, err = c.balance(members)
			}
			// Sync consumer group
			groupRequest, err := c.syncGroupRequest(coordinator, plan, join.GenerationId)
			switch groupRequest.Err  // 出错重试
			// Retrieve and sort claims
			var claims map[string][]int32
				claims = members.Topics
			return newConsumerGroupSession(ctx, c, claims, join.MemberId, join.GenerationId, handler)
				// init offset manager. 用于管理offset，
				offsets, err := newOffsetManagerFromClient(parent.groupID, memberID, generationID, parent.client)
					om := &offsetManager{
						...
						poms:   make(map[string]map[int32]*partitionOffsetManager), // topic -> partition -> offset
						...
					}
					// 如果开启conf.Consumer.Offsets.AutoCommit.Enable ，会启动异步定时提交. 频率: conf.Consumer.Offsets.AutoCommit.Interval
					return om
				// init context
				ctx, cancel := context.WithCancel(ctx)
				// init session
				sess := &consumerGroupSession{
					...
					handler:      handler, 
					offsets:      offsets, 
					claims:       claims, 
					ctx:          ctx,
					cancel:       cancel,
					...
				}
				// start heartbeat loop
				go sess.heartbeatLoop()
					defer close(s.hbDead)
					defer s.cancel() // trigger the end of the session on exit
					retries := s.parent.config.Metadata.Retry.Max
					// 重试获取Coordinator
					coordinator, err := s.parent.client.Coordinator(s.parent.groupID)
					// 重试发送心跳
					resp, err := s.parent.heartbeatRequest(coordinator, s.memberID, s.generationID)
					switch resp.Err {
							
						case ErrRebalanceInProgress: // 取消当前session，在handler里监听该事件退出消费.
							retries = s.parent.config.Metadata.Retry.Max
							s.cancel()
						case ErrUnknownMemberId, ErrIllegalGeneration:
							return
						default:
							s.parent.handleError(resp.Err, "", -1)
							return
					}
					select {
						case <-pause.C:
						case <-s.hbDying: // 消费退出后. release时调用
							return
					}
					
				// create a POM for each claim
				for topic, partitions := range claims {
					// 初始化分区分区的offset
					pom, err := offsets.ManagePartition(topic, partition)
					
				}
				// perform setup
				handler.Setup(sess)
				// start consuming
				for topic, partitions := range claims {
					sess.waitGroup.Add(1)
					for _, partition := range partitions {
						defer sess.waitGroup.Done()
						defer sess.cancel()
						go sess.consume(topic, partition)
							// get next offset
							// create new claim
							claim, err := newConsumerGroupClaim(s, topic, partition, offset)
								pcm, err := sess.parent.consumer.ConsumePartition(topic, partition, offset)
								child := &partitionConsumer{
									consumer:             c,
									conf:                 c.conf,
									topic:                topic,
									partition:            partition,
									messages:             make(chan *ConsumerMessage, c.conf.ChannelBufferSize), // c.conf.ChannelBufferSize = 256
									errors:               make(chan *ConsumerError, c.conf.ChannelBufferSize),
									feeder:               make(chan *FetchResponse, 1),
									preferredReadReplica: invalidPreferredReplicaID,
									trigger:              make(chan none, 1),
									dying:                make(chan none),
									fetchSize:            c.conf.Consumer.Fetch.Default,
								}
								child.chooseStartingOffset(offset)
								leader, err = c.client.Leader(child.topic, child.partition)
								c.addChild(child)
								// 如果child.trigger发生时. 重新建立 partitionConsumer 和 broker的关系发送input
								go withRecover(child.dispatcher)
								// 
								go withRecover(child.responseFeeder)
								
								child.broker = c.refBrokerConsumer(leader)
									bc := &brokerConsumer{
										consumer:         c,
										broker:           broker,
										input:            make(chan *partitionConsumer),
										newSubscriptions: make(chan []*partitionConsumer),
										wait:             make(chan none),
										subscriptions:    make(map[*partitionConsumer]none),
										refs:             0,
									}
								
									go withRecover(bc.subscriptionManager)
										// accepts new subscriptions on `bc.input` and bat to bc.newSubscriptions. 
										// bc.wait <- none{}. 触发subscriptionConsumer. new subscriptions is available
										// bc.newSubscriptions <- nil 触发subscriptionConsumer. if no new subscriptions are available
									go withRecover(bc.subscriptionConsumer)
										for newSubscriptions := range bc.newSubscriptions {
											bc.updateSubscriptions(newSubscriptions)
												for _, child := range newSubscriptions: bc.subscriptions[child] = none{}
												// remove subscription if subscription dying
											// 没有订阅
											if len(bc.subscriptions) == 0 { <-bc.wait continue }
											response, err := bc.fetchNewMessages()
											if err != nil { 
												bc.abort(err);
													bc.consumer.abandonBrokerConsumer(bc)
													for child := range bc.subscriptions { child.trigger <- none{} }
													for newSubscriptions := range bc.newSubscriptions { 
														if len(newSubscriptions) == 0 { <-bc.wait continue }
														for _, child := range newSubscriptions {
																child.trigger <- none{}
														}
													} 
												return
											}
											// 
											for child := range bc.subscriptions {
												child.feeder <- response
											}
											
											bc.acks.Wait()
											bc.handleResponses()
												for child := range bc.subscriptions {
													switch result {
														result := child.responseResult
														child.responseResult = nil
														// Discard any replica preference.
														child.preferredReadReplica = invalidPreferredReplicaID
												}
										}
		
								child.broker.input <- child
								return pcm
							return &consumerGroupClaim{
								topic:             topic,
								partition:         partition,
								offset:            offset,
								PartitionConsumer: pcm,
							}, nil
							
							// trigger close when session is done
							// start processing
							
		            // 真正消费的consumer
								
								
							// ensure consumer is closed & drained

							
					}
				}
				
				return sess
		// loop check topic partition numbers changed
		// will trigger rebalance when any topic partitions number had changed
		// avoid Consume function called again that will generate more than loopCheckPartitionNumbers coroutine
		// todo
		go c.loopCheckPartitionNumbers(topics, sess)
		// Wait for session exit signal
		<-sess.ctx.Done()
		// Gracefully release session claims
		return sess.release(true): func (s *consumerGroupSession) release(withCleanup bool) (err error)
			s.cancel()
			s.waitGroup.Wait()
			s.releaseOnce.Do
			s.handler.Cleanup(s)
				s.offsets.Close()
			close(s.hbDying)
			<-s.hbDead
}
```