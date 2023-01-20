---
title: Kafka源码解析-消费
date: 2023-01-05 23:37:00
tags:
---

# 消费组

# 开始消费

## broker端

首先找到我们的老朋友KafkaApis. 里面搜索FETCH, 现在调用的是`handleFetchRequest`
，代码太长了我们只看重点吧，里面实际是封装了实际调用的是`replicaManager.fetchMessages`
需要的结构，然后调用`replicaManager.fetchMessages`进行获取.

``` scala

  def handleFetchRequest(request: RequestChannel.Request): Unit = {
    val versionId = request.header.apiVersion
    val clientId = request.header.clientId
    val fetchRequest = request.body[FetchRequest]
    val fetchContext = fetchManager.newContext(
      fetchRequest.metadata,
      fetchRequest.fetchData,
      fetchRequest.toForget,
      fetchRequest.isFromFollower)
    val clientMetadata: Option[ClientMetadata] = if (versionId >= 11) {
      // Fetch API version 11 added preferred replica logic
      Some(new DefaultClientMetadata(
        fetchRequest.rackId,
        clientId,
        request.context.clientAddress,
        request.context.principal,
        request.context.listenerName.value))
    } else {
      None
    }
    
     
    // 构建合法的Topic和分区元数据.
    val interesting = mutable.ArrayBuffer[(TopicPartition, FetchRequest.PartitionData)]()
    
    // 省略: 验证鉴权代码
    // 省略: downConvert逻辑...
    // 省略: quota逻辑
   
    val fetchMaxBytes = Math.min(Math.min(fetchRequest.maxBytes, config.fetchMaxBytes), maxQuotaWindowBytes)
    val fetchMinBytes = Math.min(fetchRequest.minBytes, fetchMaxBytes)
    if (interesting.isEmpty)
      processResponseCallback(Seq.empty)
    else {
      // call the replica manager to fetch messages from the local replica
      replicaManager.fetchMessages(
        fetchRequest.maxWait.toLong,
        fetchRequest.replicaId,
        fetchMinBytes,
        fetchMaxBytes,
        versionId <= 2,
        interesting,
        replicationQuota(fetchRequest),
        processResponseCallback,
        fetchRequest.isolationLevel,
        clientMetadata)
    }
```

replicaManager class.

``` scala
  // Fetch messages from a replica, and wait until enough data can be fetched and return; the callback function will be triggered either when timeout or required fetch info is satisfied. ** Consumers may fetch from any replica, but followers can only fetch from the leader. **
  def fetchMessages(timeout: Long, // 客户端最大等待时间.
                    replicaId: Int, // 从消费者还是从哪个follwer来的. 0 < consumer
                    fetchMinBytes: Int,
                    fetchMaxBytes: Int,
                    hardMaxBytesLimit: Boolean, // 老版本参数忽略
                    fetchInfos: Seq[(TopicPartition, PartitionData)], // 获取的源信息
                    quota: ReplicaQuota, // 配额
                    responseCallback: Seq[(TopicPartition, FetchPartitionData)] => Unit, // 回调函数
                    isolationLevel: IsolationLevel,
                    clientMetadata: Option[ClientMetadata] // 客户端信息，用于处理rackId相关. 
                    ): Unit = {
    val isFromFollower = Request.isValidBrokerId(replicaId)
    val isFromConsumer = !(isFromFollower || replicaId == Request.FutureLocalReplicaId)
    
    val fetchOnlyFromLeader = isFromFollower || (isFromConsumer && clientMetadata.isEmpty)
    def readFromLog(): Seq[(TopicPartition, LogReadResult)] = {
      val result = readFromLocalLog(
        replicaId = replicaId,
        fetchOnlyFromLeader = fetchOnlyFromLeader,
        fetchIsolation = fetchIsolation,
        fetchMaxBytes = fetchMaxBytes,
        hardMaxBytesLimit = hardMaxBytesLimit,
        readPartitionInfo = fetchInfos,
        quota = quota,
        clientMetadata = clientMetadata)
      if (isFromFollower) updateFollowerFetchState(replicaId, result)
      else result
    }

    val logReadResults = readFromLog()

    // check if this fetch request can be satisfied right away
    var bytesReadable: Long = 0
    var errorReadingData = false
    var hasDivergingEpoch = false
    val logReadResultMap = new mutable.HashMap[TopicPartition, LogReadResult]
    logReadResults.foreach { case (topicPartition, logReadResult) =>
      brokerTopicStats.topicStats(topicPartition.topic).totalFetchRequestRate.mark()
      brokerTopicStats.allTopicsStats.totalFetchRequestRate.mark()

      if (logReadResult.error != Errors.NONE)
        errorReadingData = true
      if (logReadResult.divergingEpoch.nonEmpty)
        hasDivergingEpoch = true
      bytesReadable = bytesReadable + logReadResult.info.records.sizeInBytes
      logReadResultMap.put(topicPartition, logReadResult)
    }

    // respond immediately if 1) fetch request does not want to wait
    //                        2) fetch request does not require any data
    //                        3) has enough data to respond
    //                        4) some error happens while reading data
    //                        5) we found a diverging epoch
    if (timeout <= 0 || fetchInfos.isEmpty || bytesReadable >= fetchMinBytes || errorReadingData || hasDivergingEpoch) {
      val fetchPartitionData = logReadResults.map { case (tp, result) =>
        val isReassignmentFetch = isFromFollower && isAddingReplica(tp, replicaId)
        tp -> result.toFetchPartitionData(isReassignmentFetch)
      }
      responseCallback(fetchPartitionData)
    } else {
      // construct the fetch results from the read results
      val fetchPartitionStatus = new mutable.ArrayBuffer[(TopicPartition, FetchPartitionStatus)]
      fetchInfos.foreach { case (topicPartition, partitionData) =>
        logReadResultMap.get(topicPartition).foreach(logReadResult => {
          val logOffsetMetadata = logReadResult.info.fetchOffsetMetadata
          fetchPartitionStatus += (topicPartition -> FetchPartitionStatus(logOffsetMetadata, partitionData))
        })
      }
      val fetchMetadata: SFetchMetadata = SFetchMetadata(fetchMinBytes, fetchMaxBytes, hardMaxBytesLimit,
        fetchOnlyFromLeader, fetchIsolation, isFromFollower, replicaId, fetchPartitionStatus)
      val delayedFetch = new DelayedFetch(timeout, fetchMetadata, this, quota, clientMetadata,
        responseCallback)

      // create a list of (topic, partition) pairs to use as keys for this delayed fetch operation
      val delayedFetchKeys = fetchPartitionStatus.map { case (tp, _) => TopicPartitionOperationKey(tp) }

      // try to complete the request immediately, otherwise put it into the purgatory;
      // this is because while the delayed fetch operation is being created, new requests
      // may arrive and hence make this operation completable.
      delayedFetchPurgatory.tryCompleteElseWatch(delayedFetch, delayedFetchKeys)
    }
  }
```