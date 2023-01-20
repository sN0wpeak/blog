---
title: >-
  Kafka-KIP-101 - Alter Replication Protocol to use Leader Epoch rather than
  High Watermark for Truncation
date: 2023-01-16 17:21:56
tags:
---


https://cwiki.apache.org/confluence/display/KAFKA/KIP-101+-+Alter+Replication+Protocol+to+use+Leader+Epoch+rather+than+High+Watermark+for+Truncation

# Motivation
### Scenario 1: High Watermark Truncation followed by Immediate Leader Election
 
### Scenario 2: Replica Divergence on Restart after Multiple Hard Failures