# raft extented论文笔记



# raft基础

一个raft集群一般来说有5个服务器结点，能让系统承受其中任意两台的down掉。每个服务器一共有三个状态，leader,follower,candidate,follower被动的接受来自candidate与leader的消息并作出回应。leader handle了所有了所有的客户端的请求，如果follower接收到了请求，会重定向到leader那去。


raft把时间分成任意长的阶段，从选举开始，每个阶段都是一个连续增长的整数，至少有一个或者多个candidate会尝试成为leader。如果有一个candidate成为了leader，则接下来的term，它将是leader。在一些情况下，选举会出现，平分票数的情况，这种情况，term将会维持没有leader的状态到结束，到下一次的新的选举。


不同的服务器 可能感知不到不同的term之间的转换，或者根本不知道有选举这回事，在这其中，term充当了一个逻辑时钟的作用，能让服务器能检测过时的信息，比如落后的leader。每一个服务器有一个current term number，随着时间增长，当服务器之间通信时，current term 会被 交换，如果有一个服务器的term小于别的，就会更新自己的，如果一个canditate或者leader发现它自己的term过时了，就会马上转变成follower，如果服务器收到了一个过时的请求，则会直接拒绝这个请求。

raft的服务器之间用rpc请行通信，其中RequestVote用于canditate 来进行选举，AppendEntries 用于分发log与当心跳包（事实上是无论收到一个请求，都应该重置time out chekcer）。另外 还有一个转移快照的rpc。之后 再说。

<!-- more -->

# 选举

raft使用心跳机制来触发选举。当服务器启动的时候，它们都是follower，follower在收到来自leader或者 candidate合法的rpc请求之前，一直是follower，leader会周期性的发送心跳包来保持leader的权威性，心跳包是一个空的AppendEntries，不带有Log entries。当一个follower一个election timeout后没有收到心跳包，则认为leader没了，将开始一轮选举，选出一个新的leader。

为了开始一次选举，follower会给current term+1然后转变成candidate，然后投自己一票，然后同时发起RequestVote RPC给别的服务器。一个candidate会持续到以下三件情况出现，(a)它成了leader，(b)其它服务器成了leader(c)一段时间过去后，没有server成为leader。这将在之后讨论（应该是过一段时间重新发起）

一个candidate如果收到了整个集群中大多数的投票 (with same term)，就会成为leader。每一个服务器最多只能投出一票。这个`大多数` 的规则决定了，在一个term里，只能一个candidate成为leader。一旦candidate成为了leader，则会开始发送心跳包给其它所有的服务器，来保证自己的权威与阻止其它服务器发起选举。

在投票的过程中，可能 会收到来其它服务器发来的AppendEntries RPC来声称自己是leader，这种情况，看leader的term与自己的current term的大小，如果比自己大，那别人就是leader，自己将退回到follower的状态，如果比自己小，就是拒绝别人的RPC call，并继续进行选举。

一个选举如果选不出一个leader那就是分裂的选举，即没人得到大多数票，这种情况下，会等待超时直到下一次的选举，但是如果 没有做额外的操作，分裂的选举会不停的进行下去。

raft采用了随机的选举超时来保证split vote是很少见的并且能快速的被解决。即每一个服务器的election timeout不一样，在重新发起选举的之前会等待这个timeout，timeout少的会先发起选举。一般的timeout会选择一个区间比如150-300ms之间。这样会有一个服务器超时完成并在别的服务器节点超时完成前完成选举。

# Log replication

一旦一个leader被选举，那它就开始接受client的请求，每一个client的请求都包含一个命令，这个命令会被 replicated state machines执行，leader将会把这个command append到日志中去，作为一个新的entry，然后用AppendEntries 分发到其它的服务器。 leader会永远的尝试去写入follower log entry，直到所有的log entry都被 写入。

leader决定什么时候日志被 commit，当大多数的服务器都复制了一份entry后，就会提交，也会提前之前的entries，论文中设计了两个重要的属性

- 如果不同日志中的两个Entry有一样的index跟term，那他们存的一个东西
- 如果不同日志中的两个entry有一样的index跟term，那他们之前的所有entry是相同的。

属性1保证了，在给定的term与index中只会创建一个entry，属性2保证了一致性的简单性。

如果follower发现leader发过来的index跟term在自己的log里没有，那么会拒绝这个新的entry。（因为在消息中包含了这条新的之前的index跟term，所以可以检测之前的一致性，如果之前的都不一致就根本不会append这个新的）

在Raft中，处理leader与follower的不一致性是通过强制把leader的log记录复制给follower来完成的，这意味着follower的log的冲突部分，会被完全覆盖。

为了使follwer的log保持一致，leader必须找到leader与follwer最近的一个一样的entry，然后把这个之后的log都发给follower。

leader对于每一个follower都维护了一个index，这个index是leader将要发给follower的下一个index，当一个leader上台后， 会把这个next index初始化为自己log中的最后一个index，如果有follower的记录与leader不一致，那么在下一次Append Entries RPC的一致性检查中失败。当这个调用失败后，leader将决定下一个next index是多少，最终，next index会在重试中到达leader与follower一致的地方。当RPC调用成功之后，leader会把之后的记录全发给follower，这样就follower就完成了与leader的一致性同步。

leader永远不会改自己的log。这是强保证。

# 强约束

为了达成算法的强一致性，必须加上一些强约束。

## 选举

RPC必须包含candidate的log，然后投票者看canditate的log是否比自己的旧，如果旧，则会拒绝投给它，这样会保证选举出来的leader跟大多数的投票者中的log是一致的。

## 非提交的entry

raft不会提交之前阶段的非提交的entry。对于raft来说，如果leader当前的entry提交了，其中的潜在的语义包含了之前的entry都是提交了的。

## 安全性争论

这部分主要是证明future term的log是 一定包含了之前term的提交的，因为如果没提交的话，会有两个矛盾，第一，它成为不了leader，第二，它的entry log一定比之前的要大。原文中用的反证法，说term U没有包含term T的 log。实际上，在投票的时候 它成为不了leader，另外一个就是假如它是leader，它的log也得包含之前的，不然提交不了到follower。

## 崩溃

如果follower或者candidate崩溃了，leader会一直发请求，直到成功，如果follower在commit log之后但是没有回复，之后重启了，发现收到了同样index的entry写入请求，会忽视这个请求。

## 时间与可用性

raft的安全性不应该依赖时间：系统不应该因为一些事件发生的比预期的慢或者快就产生错误的结果。然而可用性却是不可避免的依赖时间。例如，因为如果回复慢了，follower就要变成candidate了。

其中leader election是raft中对时间最敏感的。raft将会选出一个稳定的leader，且这个系统满足这样的延迟要求：broadcastTime < electionTimout < MTBF。

广播时间应该要比选举超时少一个数量级，而选举时间应该要比恢复时间要少几个数量级。这样当leader崩溃的时候，也只有选举时间内不可用而已。

通常来说broadcasttim从0.5ms到20ms不等，而选举超时从10 ms 到500ms不等。而mtbf几个月。


