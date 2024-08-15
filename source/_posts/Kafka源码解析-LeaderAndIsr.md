---
title: Kafka源码解析-LeaderAndIsr
date: 2024-08-13 14:35:39
tags: kafka
---

```plantuml
@startuml
Controller->Broker: LeaderAndIsrRequest
Broker->ReplicaManager: becomeLeaderOrFollower

@enduml
```
