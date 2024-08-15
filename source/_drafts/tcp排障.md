---
title: tcp排障
tags: tcp
---


* server listen(sockfd,backlog). // net.core.somexconn 
* client connect():  
  * client[SYN_SENT] -(syn)> server[SYN_RCVD]. // sync
    * client: tcp_syn_retries=2
    * server: net.ipv4.tcp_max_syn_backlog=. // 控制半链接队列长度.
    * server: net.ipv4.tcp_syncookies // 对抗SYN Flood攻击. 半连接过多无法建立链接
  * server[SYN_RCVD] -(synack)> client[ESTABLISHED]. // synack
    * server: tcp_synack_retries=2
  * client[ESTABLISHED]-(ack)> server[SYN_RCVD]. // 进入accept queue. 全链接
    * server: net.core.somaxconn=16384
    * net.ipv4.tcp_abort_on_overflow = 0 // 控制accept queue满的行为，是RESET还是丢弃.
* server accept(): // 建联成功
client[ESTABLISHED]-(data)> server[ESTABLISHED]
* write()/read()
* client active close():
  * client[FIN_WAIT_1] -(FIN)> server[ESTABLISHED]
  * server[ESTABLISHED] -(ACK)> client[FIN_WAIT_2]
* server passive close():
  * server[CLOSE_WAIT ]-(FIN)> client[FIN_WAIT_2]
  * 