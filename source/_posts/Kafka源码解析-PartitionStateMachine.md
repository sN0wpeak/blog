---
title: Kafka源码解析-PartitionStateMachine
date: 2024-07-20 23:40:34
tags: kafka
---
PartitionStateMachine用于管理Partition的状态。目前有以下几种状态：
1. NonExistentPartition. 表示Partition不存在或者是被删除了. 前置状态：OfflinePartition
2. NewPartition.  前置状态：NonExistentPartition
3. OnlinePartition. 前置状态: NewPartition, OnlinePartition, OfflinePartition
4. OfflinePartition. 前置状态: NewPartition, OnlinePartition, OfflinePartition
```

```

### When



```scala
class PartitionModificationsHandler(eventManager: ControllerEventManager, topic: String) extends ZNodeChangeHandler {
  override val path: String = TopicZNode.path(topic)

  override def handleDataChange(): Unit = eventManager.put(PartitionModifications(topic))
}
```

```scala

class ZkReplicaStateMachine(config: KafkaConfig,
                            stateChangeLogger: StateChangeLogger,
                            controllerContext: ControllerContext,
                            zkClient: KafkaZkClient,
                            controllerBrokerRequestBatch: ControllerBrokerRequestBatch)
        extends ReplicaStateMachine(controllerContext) with Logging {
   
}
```



```scala
class ZkPartitionStateMachine(config: KafkaConfig,
                              stateChangeLogger: StateChangeLogger,
                              controllerContext: ControllerContext,
                              zkClient: KafkaZkClient,
                              controllerBrokerRequestBatch: ControllerBrokerRequestBatch)
  extends PartitionStateMachine(controllerContext) {

   override def handleStateChanges(
                                          partitions: Seq[TopicPartition],
                                          targetState: PartitionState,
                                          partitionLeaderElectionStrategyOpt: Option[PartitionLeaderElectionStrategy]
                                  ): Map[TopicPartition, Either[Throwable, LeaderAndIsr]] = {
      if (partitions.nonEmpty) {
         try {
            // 新建一个Batch. 用于检查要构建的结构是否复位. 详见AbstractControllerBrokerRequestBatch
            controllerBrokerRequestBatch.newBatch()
            val result = doHandleStateChanges(
               partitions,
               targetState,
               partitionLeaderElectionStrategyOpt
            )
            // 
            controllerBrokerRequestBatch.sendRequestsToBrokers(controllerContext.epoch){
            result
         } catch {
            case e: ControllerMovedException =>
               error(s"Controller moved to another broker when moving some partitions to $targetState state", e)
               throw e
            case e: Throwable =>
               error(s"Error while moving some partitions to $targetState state", e)
               partitions.iterator.map(_ -> Left(e)).toMap
         }
      } else {
         Map.empty
      }
   }
   
   private def doHandleStateChanges(
                                           partitions: Seq[TopicPartition],
                                           targetState: PartitionState,
                                           partitionLeaderElectionStrategyOpt: Option[PartitionLeaderElectionStrategy]
                                   ): Map[TopicPartition, Either[Throwable, LeaderAndIsr]] = {
      val stateChangeLog = stateChangeLogger.withControllerEpoch(controllerContext.epoch)
      val traceEnabled = stateChangeLog.isTraceEnabled
      // 将所有不在controllerContext.PartitionState中的partition设置默认状态为NonExistentPartition
      partitions.foreach(partition => controllerContext.putPartitionStateIfNotExists(partition, NonExistentPartition))
      val (validPartitions, invalidPartitions) = controllerContext.checkValidPartitionStateChange(partitions, targetState)
      invalidPartitions.foreach(partition => logInvalidTransition(partition, targetState))

      targetState match {
         // 将Partition状态转化为NewPartition. 只更新controllerContext.
         case NewPartition =>
            validPartitions.foreach { partition =>
               stateChangeLog.info(s"Changed partition $partition state from ${partitionState(partition)} to $targetState with " +
                       s"assigned replicas ${controllerContext.partitionReplicaAssignment(partition).mkString(",")}")
               controllerContext.putPartitionState(partition, NewPartition)
            }
            Map.empty
         case OnlinePartition =>
            val uninitializedPartitions = validPartitions.filter(partition => partitionState(partition) == NewPartition)
            val partitionsToElectLeader = validPartitions.filter(partition => partitionState(partition) == OfflinePartition || partitionState(partition) == OnlinePartition)
            if (uninitializedPartitions.nonEmpty) {
               val successfulInitializations = initializeLeaderAndIsrForPartitions(uninitializedPartitions)
               successfulInitializations.foreach { partition =>
                  stateChangeLog.info(s"Changed partition $partition from ${partitionState(partition)} to $targetState with state " +
                          s"${controllerContext.partitionLeadershipInfo(partition).get.leaderAndIsr}")
                  controllerContext.putPartitionState(partition, OnlinePartition)
               }
            }
            if (partitionsToElectLeader.nonEmpty) {
               val electionResults = electLeaderForPartitions(
                  partitionsToElectLeader,
                  partitionLeaderElectionStrategyOpt.getOrElse(
                     throw new IllegalArgumentException("Election strategy is a required field when the target state is OnlinePartition")
                  )
               )

               electionResults.foreach {
                  case (partition, Right(leaderAndIsr)) =>
                     stateChangeLog.info(
                        s"Changed partition $partition from ${partitionState(partition)} to $targetState with state $leaderAndIsr"
                     )
                     controllerContext.putPartitionState(partition, OnlinePartition)
                  case (_, Left(_)) => // Ignore; no need to update partition state on election error
               }

               electionResults
            } else {
               Map.empty
            }
         case OfflinePartition | NonExistentPartition =>
            validPartitions.foreach { partition =>
               if (traceEnabled)
                  stateChangeLog.trace(s"Changed partition $partition state from ${partitionState(partition)} to $targetState")
               controllerContext.putPartitionState(partition, targetState)
            }
            Map.empty
      }
   }
}
```

