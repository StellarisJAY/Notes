# Raft算法

高可用、最终一致性的分布式协议。

解决问题：

复制集中节点一致性

## leader选举

所有节点都有三种可能状态：leader、follower、candidate。

类似民主社会的领导人选举。哪一个节点作为leader是由所有节点选举出来的，而每一个leader的任职期叫做term，当一个term结束或者leader故障会触发新的选举。

term包含了选举(election)和任期工作(normal operation)。term的编号是递增的。

![img](https://img2018.cnblogs.com/blog/1089769/201812/1089769-20181216202049306-1194425087.png)

![img](https://img2018.cnblogs.com/blog/1089769/201812/1089769-20181216202155162-452543292.png)

## 选举过程

follower在超时时间内没有收到leader的心跳，则会主动发起选举。

1. 自增本地的term。切换状态到candidate
2. 投自己一票
3. 发送给其他所有节点，RequestVote RPCs
4. 等待其他节点回复
   - 自己收到多数票，赢得选举成为leader
   - 被告知其他节点当选，切换为follower
   - 没有收到投票结果，保持candidate

### 平局处理

每个参与方随机休息一阵（Election Timeout）重新发起投票直到一方获得多数票。这里的关键就是随机 timeout，最先从 timeout 中恢复发起投票的一方向还在 timeout 中的另外两方请求投票，这时它们就只能投给对方了，很快达成一致。

## log replication

**初始一致性**：选举的产生是因为follower没有收到前一个leader的心跳，当新的leader选出后，此时follower的log很可能是不一致的，所以会立即发起log replication来同步。

leader将客户端的command封装到多个log entry中，将log entry复制到follower节点，然后所有follower节点按相同的顺序执行log entry的命令。

当leader收到大多数节点的回复，就可以视为一致了，并提交执行结果了。该过程类似2PC，不过Raft只要求有半数以上的节点确认即可提交执行。

过程：

- leader收到客户端command

- leader 封装 log entry

- leader发起AppendEntry RPCs，将log entry的副本发送到所有节点

- 所有节点按顺序执行log entry 的 command，leader等待确认

- leader收到半数以上的确认

- leader提交log entry的执行

- leader通知客户端

- leader通知follower提交log

  

### 两阶段提交（2PC）

leader先发起请求，当所有节点都确认后才最终提交执行。

### 一阶段提交（1PC）

master节点将数据发送给follower节点后直接提交，只要基础设施没有出错就能够保证一致性。

## 高可用和最终一致性

![img](https://img2018.cnblogs.com/blog/1089769/201812/1089769-20181216202309906-1698663454.png)

半数以上就提交，这使得Raft算法没有保证高一致性，而是保证了高可用性。没有正确提交确认的follower，leader会重发log entry直到follower的log entry正确为止。

- **高可用性**：leader只收到半数以上正确确认后就能给客户端发送回复
- **最终一致性**：leader会重发log entry直到正确为止。

## 安全性

### 选举安全（election safety）

保证任何时刻只有一个leader。出现多个leader成为脑裂（brain split）

- 每个follower在任期内只能选举一次
- 只有获得多数票的节点才能成为leader
- 奇数节点数保证了多数票的存在，避免了平票

### 日志匹配（log matching）

如果两个log entry的log index和term相同，那么该index之前的所有log entry都是相同的。

leader在发送AppendEntries RPCs时，会包含最新log entry之前一个log entry的index和term。follower收到后如果找不到对应的term index就告诉leader不一致。

不一致时，follower可能有6种情况：

![img](https://img2018.cnblogs.com/blog/1089769/201812/1089769-20181216202408734-1760694063.png)

比leader日志多（多term或多index）、比leader少、有的多有的少

#### 解决不一致

leader会强制follower复制自己的log。

对每个follower，leader会定位到相同的最后一个index，然后将后面的所有log entries发送。

leader会维护一个nextIndex数组，记录leader可以发送给follower的log index。初始值是每次广播的最后一个log index+1

1. leader初始化nextIndex[x] 为 leader的最后一个log index + 1
2. AppendEntries RPCs时设置preTerm和preLogIndex为logs[nextIndex[x]-1]（即前一个log entry的term和index）
3. 如果follower判断自己log中preLogIndex位置的term和preLogTerm不同，则向leader发送不一致信息
4. leader收到后，将nextIndex[x] -= 1，从第2步重新开始过程

### leader完整性和选举限制

**leader完整性**：如果一个log etnry在某个任期被提交，那么这条日志会出现在所有更高term的leader的日志里面。

- commit majority：一个log只有复制到了majority节点才能提交
- vote majority：一个节点只有得到了majority投票才能当选leader

## 脑裂（brain split）

网络分区会导致原先的leader和follower被分开，使follower无法收到leader的心跳而触发选举。又因为网络分区，只有部分节点参与选举，就会导致出现两个leader。

![img](https://images2015.cnblogs.com/blog/815275/201603/815275-20160301175637220-1693295968.png)