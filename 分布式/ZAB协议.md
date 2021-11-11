# ZAB协议

ZAB（Zookeeper Atomic Broadcast），Zookeeper原子广播协议。Zookeeper中用于数据同步的协议，ZAB协议与Raft算法的广播方式类似。

## 超半数确认-两阶段提交

和Raft算法的超半数两阶段提交不同，Zookeeper中的请求可以由Follower接收，然后再将请求转发给leader。之后的过程和Raft的2PC相同。

![img](https://www.runoob.com/wp-content/uploads/2020/09/zk-data-stream-async.png)

1. follower收到的所有写请求都转发给leader
2. leader发起事务提议，将提议转发给follower
3. leader等待半数以上的follower反馈
4. follower对提议给出反馈
5. leader收到半数以上的反馈，提交事务
6. leader指示follower提交事务
7. leader将数据同步到Observer
8. 向客户端发送response

## 崩溃恢复

follower崩溃时，leader强制要求follower复制leader的数据。

leader崩溃时，因为follower无法收到心跳而触发选举，选举出zxid和myid最大的节点作为leader。

leader选举完成后会立即进行一次ZAB数据同步，来避免崩溃的数据不一致。