# 深入理解 TCP（1）——连接管理

TCP 最权威的文档是[RFC 793 Transmission Control Protocol (TCP)](https://www.ietf.org/rfc/rfc793.txt)，有什么不明白的地方最好看这个文档。当然了，TCP/IP 是由一系列 RFC 文档所定义，不只是这一个。

可以说，**连接管理**是 TCP 协议中最重要的部分，它包含了很多重要的细节，包括三次握手机制、四次挥手机制、连接状态的变迁等。

先放出 TCP 报文的格式，这是以下所有讨论的基础。

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |          Source Port          |       Destination Port        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                        Sequence Number                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Acknowledgment Number                      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Data | Rese- |C|E|U|A|P|R|S|F|                               |
   | Offset| rved  |W|C|R|C|S|S|Y|I|            Window             |
   |       |       |R|E|G|K|H|T|N|N|                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           Checksum            |         Urgent Pointer        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options                    |    Padding    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                             data                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                            TCP Header Format

          Note that one tick mark represents one bit position.
```

特别注意那几个标志位：URG、ACK、PSH、RST、SYN、FIN，还有后面的 Window Size。（协议又添加了一些 Flag——NS、CWR、ECN，我们暂且不讨论。"ECN and CWR are related to bandwidth congestion，Congestion Window Reduced (CWR)。NS (experimental) - The nonce sum flag is still an experimental flag used to help protect against accidental malicious concealment of packets from the sender.）

URG：A control bit (urgent), occupying no sequence space, used to indicate that the receiving user should be notified to do urgent processing as long as there is data to be consumed with sequence numbers less than the value indicated in the urgent pointer.（该控制位已经不推荐使用了）

ACK：A control bit (acknowledge) occupying no sequence space, which indicates that the acknowledgment field of this segment specifies the next sequence number the sender of this segment is expecting to receive, hence acknowledging receipt of all previous sequence numbers.

PUSH：一种控制位（push），不占用序列号，指示该数据报包含了一定要推给接收者的数据。

RST：一种控制位（reset），不占用序列号，指示接收者删除连接且不要进一步交互。接收端可以根据数据报的 Sequence Number 和 Acknowledgment Number 字段来决定是执行 reset 命令还是忽略它。在任何情况下，接收到包含 RST 到数据报都不会引起任何包含 RST 的响应。

SYN：一种控制位（synchronization），占用一个序列号，被用在初始化一个连接，指示初始序列号。

FIN：一种控制位（finish），占用一个序列号，指示发送者不会再进一步发送数据或者发送占用序列号的控制位。

## TCP 状态的变迁

```
                              +---------+ ---------\      active OPEN
                              |  CLOSED |            \    -----------
                              +---------+<---------\   \   create TCB
                                |     ^              \   \  snd SYN
                   passive OPEN |     |   CLOSE        \   \
                   ------------ |     | ----------       \   \
                    create TCB  |     | delete TCB         \   \
                                V     |                      \   \
                              +---------+            CLOSE    |    \
                              |  LISTEN |          ---------- |     |
                              +---------+          delete TCB |     |
                   rcv SYN      |     |     SEND              |     |
                  -----------   |     |    -------            |     V
 +---------+      snd SYN,ACK  /       \   snd SYN          +---------+
 |         |<-----------------           ------------------>|         |
 |   SYN   |                    rcv SYN                     |   SYN   |
 |   RCVD  |<-----------------------------------------------|   SENT  |
 |         |                    snd ACK                     |         |
 |         |------------------           -------------------|         |
 +---------+   rcv ACK of SYN  \       /  rcv SYN,ACK       +---------+
   |           --------------   |     |   -----------
   |                  x         |     |     snd ACK
   |                            V     V
   |  CLOSE                   +---------+
   | -------                  |  ESTAB  |
   | snd FIN                  +---------+
   |                   CLOSE    |     |    rcv FIN
   V                  -------   |     |    -------
 +---------+          snd FIN  /       \   snd ACK          +---------+
 |  FIN    |<-----------------           ------------------>|  CLOSE  |
 | WAIT-1  |------------------                              |   WAIT  |
 +---------+          rcv FIN  \                            +---------+
   | rcv ACK of FIN   -------   |                            CLOSE  |
   | --------------   snd ACK   |                           ------- |
   V        x                   V                           snd FIN V
 +---------+                  +---------+                   +---------+
 |FINWAIT-2|                  | CLOSING |                   | LAST-ACK|
 +---------+                  +---------+                   +---------+
   |                rcv ACK of FIN |                 rcv ACK of FIN |
   |  rcv FIN       -------------- |    Timeout=2MSL -------------- |
   |  -------              x       V    ------------        x       V
    \ snd ACK                 +---------+delete TCB         +---------+
     ------------------------>|TIME WAIT|------------------>| CLOSED  |
                              +---------+                   +---------+

                      TCP Connection State Diagram
```

一个 TCP 连接的 11 个状态，如上图所示。分别为 CLOSED、LISTEN、SYN_RCVD、SYN_SENT、ESTABLISH、FIN_WAIT_1、FIN_WAIT_2、CLOSING、CLOSE_WAIT、LAST_ACK、TIME_WAIT。

上半部分是三次握手部分的状态变迁，下半部分是四次挥手的状态变迁。

下面会慢慢深入这些状态变迁的细节。

## 三次握手

三次握手的本质是什么？本质是通过报文来交换信息，使得两台计算机在某件事情上达成一致。

什么事情需要达成一致？

-   网络是否通畅？我的消息是否可以送达对方？
-   对方的接收能力如何？我应该最大发送多大的消息报？
-   连接的需求是不是即时的？还是过期的？
-   ……

我们先来看一个最基本的三次握手连接：

```
      TCP A                                                TCP B

  1.  CLOSED                                               LISTEN

  2.  SYN-SENT    --> <SEQ=100><CTL=SYN>               --> SYN-RECEIVED

  3.  ESTABLISHED <-- <SEQ=300><ACK=101><CTL=SYN,ACK>  <-- SYN-RECEIVED

  4.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK>       --> ESTABLISHED

  5.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK><DATA> --> ESTABLISHED

          Basic 3-Way Handshake for Connection Synchronization
```

可以看到初始阶段，一个状态是 CLOSED，一个状态是 LISTEN。

再来看看双方同时开启连接的情况：

```
      TCP A                                            TCP B

  1.  CLOSED                                           CLOSED

  2.  SYN-SENT     --> <SEQ=100><CTL=SYN>              ...

  3.  SYN-RECEIVED <-- <SEQ=300><CTL=SYN>              <-- SYN-SENT

  4.               ... <SEQ=100><CTL=SYN>              --> SYN-RECEIVED

  5.  SYN-RECEIVED --> <SEQ=100><ACK=301><CTL=SYN,ACK> ...

  6.  ESTABLISHED  <-- <SEQ=300><ACK=101><CTL=SYN,ACK> <-- SYN-RECEIVED

  7.               ... <SEQ=101><ACK=301><CTL=ACK>     --> ESTABLISHED

                Simultaneous Connection Synchronization
```

可以看到双方的初始状态都是 CLOSED。

再来看看如何从旧的 SYN 数据报中恢复连接：

```
      TCP A                                                TCP B

  1.  CLOSED                                               LISTEN

  2.  SYN-SENT    --> <SEQ=100><CTL=SYN>               ...

  3.  (duplicate) ... <SEQ=90><CTL=SYN>               --> SYN-RECEIVED

  4.  SYN-SENT    <-- <SEQ=300><ACK=91><CTL=SYN,ACK>  <-- SYN-RECEIVED

  5.  SYN-SENT    --> <SEQ=91><CTL=RST>               --> LISTEN


  6.              ... <SEQ=100><CTL=SYN>               --> SYN-RECEIVED

  7.  SYN-SENT    <-- <SEQ=400><ACK=101><CTL=SYN,ACK>  <-- SYN-RECEIVED

  8.  ESTABLISHED --> <SEQ=101><ACK=401><CTL=ACK>      --> ESTABLISHED

                    Recovery from Old Duplicate SYN
```

可以看到，主动开启连接的 TCP 发现有一个报文`<SEQ=90><CTL=SYN>`，它发现这并不是最新的也不是它所期待的，所以它就发送一个 RST 报文`<SEQ=91><CTL=RST>`来要求接收者关闭该连接。这样就避免了一个在网络中兜兜转转（过期的）的 SYN 数据报错误地开启一个连接了。

**其实，TCP 设计成三次握手而不是二次握手也正是为了避免如上描述的这种情况。**

再来看一个半开连接发现的情况：

所谓半开连接，就是有一方断开了连接但是并没有通知另外一方，比如说正在通信的一方的电源突然断掉了，就会出现半开连接的情况。

半开连接的问题在于没法发现对方已经断开连接了，还为对方保持着连接的状态。

```
      TCP A                                           TCP B

  1.  (CRASH)                               (send 300,receive 100)

  2.  CLOSED                                           ESTABLISHED

  3.  SYN-SENT --> <SEQ=400><CTL=SYN>              --> (??)

  4.  (!!)     <-- <SEQ=300><ACK=100><CTL=ACK>     <-- ESTABLISHED

  5.  SYN-SENT --> <SEQ=100><CTL=RST>              --> (Abort!!)

  6.  SYN-SENT                                         CLOSED

  7.  SYN-SENT --> <SEQ=400><CTL=SYN>              -->

                     Half-Open Connection Discovery
```

```
        TCP A                                              TCP B

  1.  (CRASH)                                   (send 300,receive 100)

  2.  (??)    <-- <SEQ=300><ACK=100><DATA=10><CTL=ACK> <-- ESTABLISHED

  3.          --> <SEQ=100><CTL=RST>                   --> (ABORT!!)

           Active Side Causes Half-Open Connection Discovery
```

```
      TCP A                                         TCP B

  1.  LISTEN                                        LISTEN

  2.       ... <SEQ=Z><CTL=SYN>                -->  SYN-RECEIVED

  3.  (??) <-- <SEQ=X><ACK=Z+1><CTL=SYN,ACK>   <--  SYN-RECEIVED

  4.       --> <SEQ=Z+1><CTL=RST>              -->  (return to LISTEN!)

  5.  LISTEN                                        LISTEN

       Old Duplicate SYN Initiates a Reset on two Passive Sockets
```

## 四次挥手

四次挥手机制是关闭的 TCP 连接的机制。

TCP 连接的关闭在 RFC793 的 3.5 小节有描述。

先看一个普通的关闭连接：

```
      TCP A                                                TCP B

  1.  ESTABLISHED                                          ESTABLISHED

  2.  (Close)
      FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  --> CLOSE-WAIT

  3.  FIN-WAIT-2  <-- <SEQ=300><ACK=101><CTL=ACK>      <-- CLOSE-WAIT

  4.                                                       (Close)
      TIME-WAIT   <-- <SEQ=300><ACK=101><CTL=FIN,ACK>  <-- LAST-ACK

  5.  TIME-WAIT   --> <SEQ=101><ACK=301><CTL=ACK>      --> CLOSED

  6.  (2 MSL)
      CLOSED

                         Normal Close Sequence

```

上面这个图就是经典的四次挥手过程。

在这个关闭连接的过程中，TCP 的双方会分别在不同时候进入不同的状态。

FIN_WAIT_1 状态就是等待自己发出去的 FIN 的 ACK，FIN_WAIT_2 状态就是等待对方发送的 FIN，TIME_WAIT 就是等定时器超时，CLOSE_WAIT 就是等待自己这边完成传输之后执行关闭操作，LAST_ACK 就是等待对方给自己的 FIN 回复 ACK。

再看一个双方同时发送关闭请求的情况：

```
      TCP A                                                TCP B

  1.  ESTABLISHED                                          ESTABLISHED

  2.  (Close)                                              (Close)
      FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  ... FIN-WAIT-1
                  <-- <SEQ=300><ACK=100><CTL=FIN,ACK>  <--
                  ... <SEQ=100><ACK=300><CTL=FIN,ACK>  -->

  3.  CLOSING     --> <SEQ=101><ACK=301><CTL=ACK>      ... CLOSING
                  <-- <SEQ=301><ACK=101><CTL=ACK>      <--
                  ... <SEQ=101><ACK=301><CTL=ACK>      -->

  4.  TIME-WAIT                                            TIME-WAIT
      (2 MSL)                                              (2 MSL)
      CLOSED                                               CLOSED

                      Simultaneous Close Sequence
```

当只有一方发出关闭连接的请求时：

主动方的状态变化过程是 ESTABLISH --> FIN_WAIT_1 --> FIN_WAIT_2 --> TIME_WAIT --> CLOSED
被动方的状态变化过程是 ESTABLISH --> CLOSE_WAIT --> LAST_ACK --> CLOSED

当双方同时发出关闭连接的请求时：

双方的状态变化过程是 ESTABLISH -- CLOSING -- TIME_WAIT -- CLOSED

发送 FIN 只是表明不想继续主动发送负载消息，不代表不继续发消息，更不是代表不可再接收信息。

处于 CLOSE_WAIT 状态的 TCP 还是可以不停地发送消息给对方，对方也可以继续 ACK，形象点来说就是本来 TCP 双向数据流现在变成了单向数据流了。

---

有一个经常会被问到的问题：在关闭连接时，为什么要有一个 TIME_WAIT 的状态，然后还要额外等待 2MSL 之后才关闭该连接？为什么不直接关闭该连接？

这是因为，当被动关闭的一方发送 FIN 之后，主动关闭的一方要回复一个 ACK，这样被动方才知道对方得到了 FIN 的消息，但是这个 ACK 如果在网络中迟迟到达不了对方的话怎么办？此时如何主动一方直接关闭了连接，然后又马上创建一个新的连接，发送一个 SYN 的报文，那么如果这个 SYN 报文可以顺利建立一个连接吗？不可以，因为被动的一方认为上一个连接还没有关闭，就不会再配合主动方建立一个新的连接。换句话说，TIME_WAIT 的设计是为了隔离一个连接的结束过程和下一个连接的同步过程，不让前者干扰后者的连接。

如果最后一个 ACK 丢失了，会怎么样，被动方如何进入 CLOSED 状态？在 TIME_WAIT 和 CLOSED 状态下，也是可以回复报文给对方的。TIME_WAIT 状态下如果再次收到 FIN 报文，那么它又会再次发送 ACK 给对方。如果在 CLOSED 状态下收到报文，那么它会发送 RST 报文给对方，让对方重置连接。如果整个网络都出问题呢？那么就很有可能进入半开状态，被动方还维持着 LAST_ACK 状态，然后不停地重发 FIN，直到重发一定的数量后自我重置连接。

---

连接队列

一般有两种类型的连接，一种是接收到了 SYN 但是没有建立连接，另外一种是已经处于 ESTABLISHED 状态，但是还没有被应用程序使用。

我们一般可以控制这两个队列来限制连接的数量。

这里面的逻辑稍微有点繁琐。

---

SYN 泛洪攻击

解决办法之一就是使用 SYN cookie：当一个连接请求到来时，服务端不需要为其分配内存，服务端会选择一个初始，然后直接把 SYN+ACK 返回给客户端，客户端

ICMP PTB 消息攻击

连接劫持/欺骗攻击
