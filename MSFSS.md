## MSFSS: A Storage System for Mass Small Files

原文链接：[A Storage System for Mass Small Files](http://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=4281592)

摘要：我们设计并实现了一个叫做MSFSS的用于存储和访问大量小文件的可扩展的灵活的分布式文件系统。MSFSS是构建在现有的常用文件系统之上的一个平台。它能够通过它们的访问模式将文件存放到最合适的文件系统上。为了防止中央瓶颈，它优化了元数据大小，将元数据操作从文件数据传输中分离，并实现了批量元数据操作。该系统提供数据迁移、热点文件缓存和副本机制，这些都是构建大规模可靠的存储系统的必要的特性。它已经成功地用于我们的web应用程序的存储系统，我们的web应用有大约50TB的小文件。实验结果表明，MSFSS在文件操作方面，可以提供高可扩展性和高吞吐量。

# 1 介绍

因特网服务常常需要存储和访问大量的小文件，比如html文件，电子邮件，论文，图片等等。因特网服务的存储特点对现有的存储系统带来了挑战。第一，拥有10亿多个文件是很正常的。大量的文件在传统的分布式文件系统上会产生大量的元数据。这对元数据操作的高效性和可扩展性带来了压力。第二，文件不大。因此，没有必要将文件进行分条带地存储到多个服务器上。相反，我们可以将整个文件存储到一个服务器上，这样能够简化实现。第三，不同类型的数据通常有不同的性能、可靠性和可用性要求，这就在选择合适的存储系统方面增加了复杂性。比如，属于VIP用户的图片比免费用户的额图片提供更高的可靠性。

我们设计并实现了MSFSS，以此来满足因特网服务的存储需求。它是一个可扩展的和灵活的分布式文件系统，特别适合存储和访问小文件或者大小适度的文件。MSFSS基于分布单元来分布数据，而且它的元数据大小可以控制。另外，它还能通过在服务器集群中对元数据分区和对元数据操作进行批处理来提供元数据的性能。MSFSS构建于成熟的常用的文件系统之上，将块分配的细节留给底层来做。这样可以简化我们的设计，使得系统更加可靠。它还是高度可配置的，它可以自动为各种文件选择合适的底层文件系统。
现在，MSFSS已经部署为我们的web应用的存储系统，该web应用有大量用户。系统当前有50TB的数据，聚合带宽大约是100MB/s。

# 2 系统概要

## 2.1 体系结构

如图1，一个MSFSS集群由一个master，多个MDS服务器和若干个存储节SN点组成，它可以通过一个叫做FSI的客户端库进行访问。它们都是用户级进程，因此，只要机器资源允许，很容易将SN和MDS搭建在同一个机器上。

![](https://github.com/luofengmacheng/translation/raw/master/pic/pic1.png)

MSFSS通过一个全局唯一的128位的文件ID（FID）来标识一个文件，这个ID是在文件创建时生成的。SN将文件存储在本地文件系统，然后通过FID和范围来读写文件数据。MSFSS支持文件副本机制，通过分布式业务来保证副本一致性，使用快速同步来恢复不一致的副本。而且，它还支持并发控制和块操作，比如数据迁移和数据拷贝。

MDS将系统元数据保存在内存中，主要是DAT（Distribution unit Allocation Table分布单元分配表，它是分布单元和文件到存储节点的映射关系表）。它主要提供元数据操作，比如FID分配和文件位置查询。

Master持续监视存储节点和MDS的状态，并且平衡负载和控制整个系统的活动，比如，故障恢复、数据同步、数据迁移、终止未决的业务。Master周期性地与存储节点和元数据服务器通过心跳消息来通信，来收集它们的状态，并给他们指令。

FSI（链接到应用程序的客户端库）实现了MSFSS访问接口，代表应用程序访问文件。FSI从MDS处获取元数据，直接从存储节点处获取数据。它还会对元数据进行缓存，这样能够提高性能。

## 2.2 常用文件系统的协作

存储节点将存储管理中的繁杂任务留给常用文件系统，比如块分配，实践证明，这种策略使得系统更可靠。这种方法带来的好处：（1）减少实现成本；（2）能够有效利用多种有用的文件系统工具。

然而，没有常用文件系统对所有类型的文件是最优的。比如，ReiserFS可以高效地存储和访问小文件，但是它的挂载时间较长，而且可靠性比EXT2/3低。通过我们的测试，XFS和JFS的读吞吐量很高，但是文件创建效率很低。虽然EXT2/3使用较多，它的缺点是有严重的文件碎片，当文件系统运行时间过长会使性能显著下降。而且，不同的文件系统挂载选项对文件系统的性能和可靠性的影响很大。比如，最常用的选项之一是"sync"，它能够减少数据丢失的可能性，但是是以性能为代价的。

MSFSS对应用程序隐藏了常用的文件系统，能够基于文件所要求的性能、可靠性和开销自动选择文件系统。

## 2.3 分布单元

在一个分布式文件系统中，常常会有几亿个文件。上面已经提到过，MSFSS基于分布单元来进行分布，从而减少元数据大小。

分布单元的概念跟AFS中的卷的概念类似，代表文件的集合。另外，它还有所有文件的通常的存储属性，比如可靠性，性能，可用性需求，合适的大小，最大容量。合适的大小和最大容量限定了文件的大小和分布单元的容量。合适的大小提供了选择常用文件系统的一种基本原则。我们喜欢用ReiserFS存储小文件，并且会将大小相近的文件存放在同一个服务器上来减少文件系统碎片。分布单元的容量对元数据的大小有很大的影响。大的分布单元使得元数据较大，但是，小的分布单元容量能够获得更好的负载均衡能力和更快的副本同步速度。MSFSS能够通过分布单元的属性自动决定它的容量。比如，归档数据的分布单元容量较大。相反，副本数据的分布单元容量较小。

与文件类似，分布单元是通过一个全局的64位的分布单元ID来标识的。当新文件存储在系统中，分布单元自动被创建，不需要管理员的干涉。

## 2.4 FID 和 FID的分配

前面已经提到，MSFSS中的每个文件都有一个全局唯一的FID。如下所示，它是一个128位的整数，每个域都可以支持MSFSS的扩展。

![](https://github.com/luofengmacheng/translation/raw/master/pic/pic2.png)

UDID表示文件所在的分布单元ID，SEQ用于标识分布单元中的某个文件。为了简单起见，DUID和SEQ持续增长，MSFSS不会再使用之前已经分配的值。我们可以期望它们都足够大，MSFSS不会用完。比如，假如系统每秒产生一百万个文件，那么64位的SEQ可以用50万年。

MSFSS创建文件需要两步，因此，它需要两种网络消息。首先，FSI在一个系统选定的分布单元中分配一个FID；然后，FSI发送新分配的FID和文件数据给对应的存储节点。为了减少网络开销和MDS的负载，FID以批处理方式分配FID。也就是说，FSI在一个分配请求中分配多个FID，然后把FID缓存在本地内存，之后就可以快速分配。这种方法能够有效减少通信开销，使得每个创建请求只产生一个消息。即使在FSI故障时缓存的FID丢失了，由于FID的空间很大，可以忽略这部分FID。这种方法需要考虑的另一个问题是，它对负载均衡有负面影响。事实上，这并不是一个严重的问题，因为FSI的可用数量很大。另外，负载均衡可以通过限定一个批处理中FID分配的数量进行改善，防止只在一个分布单元中分配FID。

## 2.5 分布式元数据管理

传统文件系统中，元数据操作的工作量占到了典型文件系统工作量的一半，因此，提高元数据操作的效率对提高整个文件系统的性能至关重要。MSFSS通过以下集中方式提高元数据操作效率。第一，MSFSS最大限度地从文件数据中分离出文件元数据。元数据的操作被一个MDS集群集中管理。第二，渐渐少元数据大小，MSFSS采用了扁平的文件命名空间，将文件组合成分布单元，再将分布单元存储在存储节点上。因此，元数据很小，可以加载到MDS的内存中。第三，客户端库FSI可以缓存元数据。第四，MSFSS采用批量FID分配和预分配策略，就能够最小化MDS和FSI之间的通信负载。

MSFSS有两种主要的元数据：分布单元信息和每个分布单元的位置信息。首先，MSFSS用DUID将元数据哈希到bucket中；然后，它将元数据bucket分布到所有的MDS中。每个MDS会在内存中存储元数据，因此，可以提高元数据操作的速度。

元数据的前一种类型记录了每个分布单元的最大分配序列号，将它作为下一个FID分配的起始点。后者将分布单元映射到实际存储文件的一些存储节点。MDF不保存这种元数据的持久拷贝。在MDS启动时，检查所有的存储节点就能够获得所有的元数据。

## 2.6 并发控制

现在，MSFSS支持全文件锁，也就是锁整个文件，这对于小文件是足够的。为了最小化master在系统活动中的参与，锁服务是在存储节点实现的。

与DBMS类似，MSFSS采用多种粒度的锁和意向锁。特别地，有两种级别的锁：文件锁和分布单元锁。为了读一个文件，存储节点首先向对应的分布单元申请一个读意向锁，然后向申请对文件的读锁；类似的，在写操作之前，必须申请对应分布单元的写意向锁，然后申请文件的写锁；同样的，为了迁移数据，需要申请分布单元的读锁。如果是副本文件，读操作只需要获得主副本的锁，而写操作则需要获得所有副本的锁。

在拷贝分布单元时，可能会发生死锁，因为需要获得所有副本的锁。我们给一个存储节点一个序列号，然后按照序列号来申请锁，以此来避免死锁。

# 3 缓存和一致性

MSFSS主要缓存了两类数据：分布单元的位置信息和读热点文件的内容。这部分，我们将描述这两类数据。

## 3.1 元数据缓存

FSI缓存分布单元的位置信息以加速确定文件位置。缓存的开销相对于重新操作的性能开销是很小的，因为元数据很小，而且很少改变。事实上，在一个有数十TB数据的一般的系统(modest system)中，元数据的大小少于1MB。另外，只有当数据迁移和故障恢复时，缓存的数据才会改变，但是这两种操作都是很少出现的。频繁的元数据操作不会改变这些数据，比如FID分配和分布单元创建。结果，FSI访问文件只需要一个请求和一个响应。

MSFSS采用(investigates,调查，研究)时间戳来获得元数据一致性。MSFSS将元数据哈希到桶(buckets)中，每个桶关联一个时间戳。时间戳只在元数据改变时才改变。MSFSS采用延迟策略将元数据的更新操作反馈到FSI，但是它会及时更新受到影响的存储节点的元数据版本。我们能够保证有状态的元数据不会误导FSI。在每个请求消息中，FSI加上对应的元数据时间戳。在收到请求后，存储节点用它自己的时间戳验证消息的时间戳，当时间戳不匹配时，它就会返回一个特殊的错误消息。然后，FSI意识到它已经丢失了元数据的部分更新，之后，它就会向MDS请求一个新的版本。最后，当元数据刷新后，FSI重新尝试操作。

这些基于时间戳的方法有几个好处。第一，它可以加快元数据操作，因为受影响的存储节点通常很少。比如，在一次数据迁移过程中，只有两个存储节点受到影响。相反，FSI更新元数据时间戳的代价很高，因为系统中有几百个FSI。第二，FSI不需要更新那些它不会访问的文件的元数据。在一个大型系统中，多个应用程序常常并发执行，每个应用访问不同的数据集。

## 3.2 热点文件缓存

相对与传统分布式文件系统，MSFSS只在FSI的内存中缓存一小部分经常读取的文件。我们称这些文件为热点文件(hot files)。热点文件的访问模式是经常读，因此，我们能够研究代价相对较高的方法提升缓存一致性。

MSFSS用热度(heat)表示文件的访问频率。系统通过如下方法计算热度值：首先，将系统活动分为2类，一种是时间间隔，一种是操作次数。然后，通过历史热度和当前访问频率来计算当前热度值。热度值的计算如下公式所示，这里的w(e)表示写的次数，r(e)表示读的次数，d是介于0到1之间的衰退因子。

存储节点周期性地收集一部分热点值最大的热点文件，并将它们发送给master。因此，master能够计算全局的热点文件。之后，当FSI访问热点文件时，master会通知FSI缓存它们。与此同时，系统会在内存中注册一个回调函数，当文件改变或者变成冷数据时，该回调函数会被调用使得FSI的缓存数据失效。这种回调策略借用自AFS，在我们的场景中开销是相对较低的，因为我们只缓存读最多的热点文件。

# 4 副本策略

MSFSS提供分布单元的副本来提升读性能、可用性和可靠性。默认情况下，它存储两个副本；但是，应用程序可以指定副本级别。

任何副本都可以用来进行读操作，但是当进行写操作时，所有的副本都必须更新。为了在数据改变时保持一致性，所有的副本都必须原子的更新。为了获得一致性，我们使用两阶段提交协议的修改版，其中FSI作为提交的协调者，副本作为协调的参与者。协议运行方式如下：第一个阶段，FSI将prepare请求和文件的修改内容发送到对应的存储节点，然后等待响应。如果没有发生错误，存储节点会把文件内容存储到本地LRU缓冲区，并发送一个ready消息，然后等待FSI的全局确认；否则，存储节点会响应abort消息。第二个阶段，FSI基于响应消息作出决定，然后向存储节点发送commit或者abort消息。

如果某个存储节点在等待FSI时超时了，它就会将这次的事物作为垂悬事务(dangling transaction)，然后立即向master报告。如果master收到垂悬事务消息，master就修改相关的存储节点的事务状态。如果任何参与事务的存储节点收到commit消息，master就提交事务。否则，master就告诉参与的存储节点终止。由于网络或者master故障，垂悬事务可能不会立即解决。这种情况下，存储节点就会周期地发送垂悬消息，知道所有的垂悬事务被处理了。

如果某个存储节点崩溃了，FSI只是简单地在剩余存储节点上提前两阶段提交。然而，事实上，不太可能将网络分区和存储节点故障区分开来。因此，我们依赖于master对于所有正常的存储节点的视图。理论上，master可能在网络故障后停留在一个小的网络分区中，因此，只有一小部分系统可以正常工作。但是，网络分区在实际的系统中很少发生，因此，这是可以接受的。

# 5 故障恢复

MSFSS的任何组件在任何时刻都有可能发生故障。在前面，我们已经讨论过FSI故障的处理。我们不需要处理master和MDS的故障，因为，它们通常没有持久状态。因此，这部分，我们只讨论存储节点的故障处理。

如果某个存储节点发生故障，那么它上面的没有副本的分布节点变得不可用，如果有副本的话，将丢失一个副本。我们不会立即为有副本的分布节点生成一个新的副本。而是让MSFSS等待预定义的一段时间。如果发生故障的存储节点在这段时间内重启了，它就会从别的副本上面对该故障机器的副本进行重建。否则，master选择一个新的存储节点，然后申请分布单元的读锁，从其它副本拷贝文件。为了最大化重建的并行度，master试图对每个需要重建的分布单元选择不同的存储节点。

# 6 实验

我们测试了创建和访问的性能，这两种操作在网络服务中是最常用的。我们测试了一个MSFSS集群的吞吐量：一个MDS，一个master，4个存储节点，数十个测试客户端。MDS和master运行在同一台机器，所有的测试客户端运行在6台机器上。这些机器用一个交换机互联。所有的存储节点使用新创建的XFS作为底层文件系统。机器的硬件配置如下：

```
CPU AMD64 2.0GHz
DISK 80G SATA with RAID
MEMORY 2G
LINK 1Gbps
```

在所有的测试中，我们都使用随机生成的负载，因此，在每两个访问之间没有关联。所有的测试客户端都是多线程的。在文件创建测试中，每个客户端线程根据预定义的值随机选择文件大小，生成随机数据，将数据上传到存储节点，然后在本地磁盘记录FID。在读测试中，每个客户端线程随机选择一个之前保存的FID，读取这个FID对应的文件。

我们生成了两类负载，包括大小缓和(modest)的文件和小文件。没有副本时的吞吐量如图3和图4。图5和图6是不同的副本数时的吞吐量。XFS性能如图2。

![](https://github.com/luofengmacheng/translation/raw/master/pic/pic3.png)

![](https://github.com/luofengmacheng/translation/raw/master/pic/pic4.png)

![](https://github.com/luofengmacheng/translation/raw/master/pic/pic5.png)

# 7 相关工作

AFS将整个文件存储在单个存储服务器上，用卷定位文件。与AFS类似，MSFSS将分布单元作为负载均衡的基本单元。与卷不同，分布单元是精细(fine-grained)的和动态的，并且有存储属性，比如可靠性和可用性。当存储大小增加时，MSFSS可以动态地创建分布单元，并且在存储节点之间迁移分布单元来均衡负载。

Vesta，Galley，PVFS，Swift将数据分条带地存储到多个存储服务器上，以此获得对于大文件的高传输速率。MSFSS没有实现数据条带化，因为数据条带化对于小文件没有意义。

xFS，GFS，GPFS全局地管理磁盘块并且使用块链表来分配文件。这种方法没有足够的可扩展性，因为块的数量比文件数量大很多。相反，我们把块分配这样的任务留给存储节点上的常用文件系统。Google FS也属于这类，它使用非常大的块获得可扩展性。然而，它只对大文件的读和追加进行了优化。

Ceph是最近才出来的一个可扩展的文件系统。它使用CRUSH来放置和确定文件数据，CRUSH是一个数据分布函数。即使这种方法是快速的和可扩展的，很难将文件数据放置到专用的存储节点，也很难从一个服务器将文件迁移到另一个服务器上。相反，MSFSS能够将小文件的分布单元放置在ReiserFS。

FAB，Lustre，zFS是基于对象存储范式的，它们最接近MSFSS。然而，它们缺少可扩展性和可适配的元数据管理。MSFSS优化了元数据大小，对元数据服务器集群上的元数据进行分区。

Ursa Minor是一个基于集群的存储系统，它允许特殊数据的选择，在线改变，编码和错误模型。MSFSS与Ursa Mino很类似，但是它着重于不同类型的文件选择合适的常用文件系统。

# 8 结论

本文展示了优化大量小文件存储的分布式文件系统，它是构建网络服务的一种重要工具。大量小于4MB的小文件使得我们必须作出与传统系统不同的设计选择。因此，MSFSS使用分布单元作为数据分布的基本单元，在并发控制方面，采用多种粒度的全文件锁，使用两阶段提交的方法保证副本一致性。

MSFSS将元数据操作从文件操作中分离出来，降低了存储节点的负载，减少了中央瓶颈。MSFSS减小了元数据大小，对MSFSS集群中的元数据分区，在FSI部分缓存元数据。

MSFSS使常用文件系统管理块分配。此外，它试图为不同类型的数据选择最合适的常用文件系统。
