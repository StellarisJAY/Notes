# synchronized实现原理

synchronized关键字在java并发编程中极其重要，我们常常误认为synchronized就是重量级锁，其实在jdk6以后就对synchronized做了很多优化，包括偏向锁、轻量锁、自旋优化、锁升级等。

## 对象头

关于synchronized的锁信息都是记录在java的对象头中的。java中的对象由对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）组成。对象头主要记录了类信息、hashcode、分代年龄、锁信息等。

### 对象头结构

| 长度     | 字段                   | 说明                           |
| -------- | ---------------------- | ------------------------------ |
| 32/64bit | Mark Word              | 存储对象的hashcode、年龄等信息 |
| 32/64bit | Class Metadata Address | 指向类对象的指针               |
| 32/64bit | ArrayLength            | 数组长度（该对象是数组）       |



### Markword

Markword记录了对象的基本信息和锁相关信息

| 锁状态   | 25bit        | 4bit     | 是否偏向 | 标记位 |
| -------- | ------------ | -------- | -------- | ------ |
| 无锁     | hashcode     | 分代年龄 | 0        | 01     |
| 偏向锁   | 线程ID+Epoch | 分代年龄 | 1        | 01     |
| 轻量级锁 | 指向栈中     | 锁记录   | 指针     | 00     |
| 重量级锁 | 指向         | 互斥量   | 指针     | 10     |
| GC标记   |              |          |          | 11     |

### 如何查看对象头

使用jol的ClassLayout查看对象头内容。

**Maven依赖**：

```xml
<dependency>
	<groupId>org.openjdk.jol</groupId>
	<artifactId>jol-core</artifactId>
	<version>0.8</version>
</dependency>
```

**打印对象头**：

```java
Object lock = new Object();
System.out.println(ClassLayout.parseInstance(lock).toPrintable());
```

**显示结果**：

```
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
      
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```



## 偏向锁

经过研究发现，大多数情况下锁不仅不出现竞争，而且总是一个线程多次获得。为了减少获得锁的成本，jvm提供了偏向锁。

顾名思义，偏向锁是偏向一个线程的锁。当想要获取锁时，会使用CAS操作在MarkWord中记录偏向的线程ID。以后的加锁和释放就不再需要CAS操作，只需要判断一下MarkWord的线程ID即可加锁。

简单地说，偏向锁的思路就像是在门上贴上名字，以后看到是自己的名字就不用再上锁或开锁直接进门。

### 偏向加锁过程

1. 判断MarkWord的ThreadID是否是自己的ID
2. 不是则用CAS替换ThreadID
3. 是则获得锁，进入同步代码块执行


![偏向锁.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f393f7648ce24ab580ce7419b6013407~tplv-k3u1fbpfcp-watermark.image)

### 撤销偏向

撤销偏向发生在出现锁竞争的时候。当一个线程使用CAS替换线程ID失败时触发撤销偏向。

- 首先会等待原持有锁线程进入**全局安全点**（该时间点没有字节码执行）
- 然后判断该线程是否已经退出同步块
- 如果已退出，则释放偏向锁，然后让当前竞争锁的线程再次尝试获取偏向锁
- 如果没有退出，则生成锁记录，然后升级为轻量级锁

### 与偏向锁相关jvm参数 

偏向锁是默认开启的，不过它会在程序启动后的几秒钟后才启动。

- -XX:UseBiasedLocking=false：关闭偏向锁
- -XX:BiasedLockingStartupDelay=0：调整偏向锁延迟启动

### 总结

- 偏向锁主要出现在没有锁竞争的场景
- 偏向锁的实现是在MarkWord中用CAS操作记录线程ID
- 发生竞争后会触发撤销偏向

## 轻量级锁

轻量级锁出现在存在竞争，但是竞争不激烈，即线程之间大致是交替执行的场景。

轻量级锁通过在锁记录、CAS、MarkWord实现了锁机制，还通过自旋进行了优化。

### 锁记录（Lock Record）

使用轻量级锁时，线程会在栈中生成锁记录，并将锁对象头中的原MarkWord拷贝到锁记录中。

所以锁记录除了基本的线程信息还包括了锁对象的MarkWord。

### 加锁过程


![轻量级锁加锁.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca063dab21f54149b2a5901672eb7695~tplv-k3u1fbpfcp-watermark.image)

1. 线程会先在栈中生成LockRecord，并将对象的原MarkWord拷贝到LockRecord中。比如图上的线程将无锁状态的带有hashcode、age的Markword拷贝到了LockRecord中
2. 线程会使用CAS操作尝试将MarkWord的字段替换为指向自己锁记录的指针，然后改变状态为为00
3. 如果步骤2 成功，线程加轻量级锁成功。如果失败，表示有竞争发生，这时线程不会立即阻塞，而是以自旋的方式继续尝试

### 自旋优化


![自旋优化.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80d7c741d75e4b49bfad83ac7e156555~tplv-k3u1fbpfcp-watermark.image)

之前提过，轻量级锁存在的场景是线程很少发生竞争，即线程近似交替执行。所以在轻量级锁的情况下，线程不会因为竞争锁失败就立即阻塞。它会进入自旋，即再次尝试CAS替换锁记录。只有当自旋超过一定次数后，jvm才会认为竞争已经超过了轻量级锁的能力而升级为重量级锁。

自旋优化更适合多核CPU下进行，通过让线程空转（自旋）来避免阻塞会占用CPU，所以在单核CPU中，自旋会带来频繁的上下文切换，效率不一定高。

### 总结

- 轻量级锁解决的是线程之间竞争很少，即线程可以近似看作交替执行的场景。
- 轻量级锁通过锁记录实现，加锁先拷贝MarkWord再用CAS替换锁记录指针。
- 自旋优化是在CAS替换失败后的再次尝试，在多核CPU下性能更佳。

## 锁膨胀

当超过自旋次数上限还没有竞争到锁时，轻量级锁会升级为重量级锁。升级的过程称为锁膨胀。


![锁膨胀.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4025920a61324146a59f87ff49afe22b~tplv-k3u1fbpfcp-watermark.image)

锁膨胀的目的是要生成重量级锁的Monitor，所以过程如下：

1. 生成Monitor，将Owner设置为当前持有轻量级锁的线程，线程通过锁记录指针找到
2. 将锁对象MarkWord改为指向Monitor的指针，状态改为10
3. 将竞争失败的线程放入Monitor的EntryList阻塞

## 重量级锁

重量级锁是通过操作系统的互斥量和管程机制实现的，所以要创建和管理一个重量级锁有很大的成本，这也是为什么jvm要用偏向和轻量级锁来优化synchronized。


![重量级锁.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/abd2b883499a4e0d8d99d96527dad4f2~tplv-k3u1fbpfcp-watermark.image)

重量级锁的Monitor如图，锁对象通过MarkWord中的指针与Monitor关联。Monitor中的主要几个结构有Owner、EntryList、WaitSet

- **Owner**：当前持有锁的线程
- **EntryList**：阻塞队列，所有竞争锁失败的线程会记录在这个队列中，虽然是队列结构，但是锁的竞争不是公平的。当持有锁线程释放锁后，entryList中的所有线程会再次竞争，不是按照先来先得的顺序获得锁。
- **WaitSet**：调用wait()方法释放锁并进入等待的线程会在该集合中。只有当持有锁线程调用notify()或notifyAll()才会离开WaitSet然后进行竞争。注意，唤醒后不会直接执行，而是会经历又一次竞争。
