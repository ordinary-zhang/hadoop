### 概述
该篇文章谈一谈Hadoop的性能调优，有哪些调优点，怎么去考虑。
### 1.存储格式压缩格式
对于存储格式压缩格式可以说是大数据场景下的很重要的一个调优点，压缩格式有哪些呢？最基本的TextFile(文本形式的行式存储)，SequenceFile（二进制形式的行式存储），RCfFile（先行后列）基于RCFile优化后的ORCFile（先行后列），Rarquet（先行后列）。选择合适的存储格式。

压缩的话，也是有好多种，并且他们具有不同的特点，Bzip2,gzip,lzo,lz4，snappy,default;了解他们的压缩比，以及能否分片的特点有助于选择合适的压缩方式。

对于存储格式和压缩格式的文章在本人博客中进行了详细的解答（<a href="https://blog.csdn.net/yu0_zhang0">博客地址</a>），这里就不说了，后面对于Hive的优化中也会说到。
### 2.设置Combiner
MapReduce计算过程我们知道Shuffe是非常影响性能的，对于一大批MapReduce程序，在不影响最终输出结果的情况下可以设置一个Combiner（比如累加，最大值等。Combiner的使用一定得慎重，如果用好，它对job执行效率有帮助，反之会影响reduce的最终结果。）,Combiner又称作map端的reduce,那么对于提高作业性能是十分有帮助的。Combiner可减少Map Task中间输出的结果，从而减少各个Reduce Task的远程拷贝数据量，最终表现为Map Task和Reduce Task执行时间缩短。

### 3.启动推测执行机制
推测执行是Hadoop对“拖后腿”的任务的一种优化机制，当一个作业的某些任务运行速度明显慢于同作业的其他任务时，Hadoop会在另一个节点上为“慢任务”启动一个备份任务，这样两个任务同时处理一份数据，而Hadoop最终会将优先完成的那个任务的结果作为最终结果，并将另一个任务杀掉。

### 4.设置失败容忍度
Hadoop运行设置任务级别和作业级别的失败容忍度。作业级别的失败容忍度是指Hadoop允许每个作业有一定比例的任务运行失败，这部分任务对应的输入数据将被忽略；
任务级别的失败容忍度是指Hadoop允许任务失败后再在另外节点上尝试运行，如果一个任务经过若干次尝试运行后仍然运行失败，那么Hadoop才会最终认为该任务运行失败。用户应该根据应用程序的特点设置合理的失败容忍度，以尽快让作业运行完成和避免没必要的资源浪费。
### 5.适当打开JVM重用功能
为了实现任务隔离，Hadoop将每个任务放到一个单独的JVM中执行，而对于执行时间较短的任务，JVM启动和关闭的时间将占用很大比例时间，为此，用户可以启用JVM重用功能，这样一个JVM可连续启动多个同类型的任务。

### 6.合理使用DistributedCache
一般情况下，得到外部文件有两种方法：一种是外部文件与应用程序jar包一起放到客户端，当提交作业时由客户端上传到HDFS的一个目录下，然后通过Distributed Cache分发到各个节点上；另一种方法是事先将外部文件直接放到HDFS上，从效率上讲，第二种方法更高效。第二种方法不仅节省了客户端上传文件的时间，还隐含着告诉DistributedCache:"请将文件下载到各个节点的pubic级别共享目录中”，这样，后续所有的作业可重用已经下载好的文件，不必重复下载。
### 7.提高作业优先级
所有Hadoop作业调度器进行任务调度时均会考虑作业优先级这一因素。作业的优先级越高，它能够获取的资源（slot数目)也越多。Hadoop提供了5种作业优先级，分别为VERY_HIGH、 HIGH、 NORMAL、 LOW、 VERY_LOW。</br>
注：在生产环境中，管理员已经按照作业重要程度对作业进行了分级，不同重要程度的作业允许配置的优先级不同，用户可以擅自进行调整。
### 8. 操作系统调优
1. 增大打开文件数据和网络连接上限，调整内核参数net.core.somaxconn，提高读写速度和网络带宽使用率
2. 适当调整epoll的文件描述符上限，提高Hadoop RPC并发
3. 关闭swap。如果进程内存不足，系统会将内存中的部分数据暂时写入磁盘，当需要时再将磁盘上的数据动态换置到内存中，这样会降低进程执行效率
4. 增加预读缓存区大小。预读可以减少磁盘寻道次数和I/O等待时间

### Hdfs参数调优
1. core-default.xml：
hadoop.tmp.dir：</br>
默认值： /tmp</br>
说明： 尽量手动配置这个选项，否则的话都默认存在了里系统的默认临时文件/tmp里。并且手动配置的时候，如果服务器是多磁盘的，每个磁盘都设置一个临时文件目录，这样便于mapreduce或者hdfs等使用的时候提高磁盘IO效率。
fs.trash.interval：</br>
默认值： 0</br>
说明： 这个是开启hdfs文件删除自动转移到垃圾箱的选项，值为垃圾箱文件清除时间。一般开启这个会比较好，以防错误删除重要文件。单位是分钟。</br>
io.file.buffer.size：</br>
默认值：4096</br>
说明：SequenceFiles在读写中可以使用的缓存大小，可减少 I/O 次数。在大型的 Hadoop cluster，建议可设定为 65536 到 131072。</br>
2. hdfs-default.xml：</br>
dfs.blocksize：</br>
默认值：134217728</br>
说明： 这个就是hdfs里一个文件块的大小了，CDH5中默认128M。太大的话会有较少map同时计算，太小的话也浪费可用map个数资源，而且文件太小namenode就浪费内存多。根据需要进行设置。</br>
dfs.namenode.handler.count：</br>
默认值：10</br>
说明：设定 namenode server threads 的数量，这些 threads 會用 RPC 跟其他的 datanodes 沟通。当 datanodes 数量太多时会发現很容易出現 RPC timeout，解決方法是提升网络速度或提高这个值，但要注意的是 thread 数量多也表示 namenode 消耗的内存也随着增加

### 任务级别参数调优
hadoop任务级别参数调优分两个方面: Map Task和Reduce Task。这方面的调优也很重要，在后面的Hive中还会涉及到。
1. Map Task调优
map 任务执行会产生中间数据,但这些中间结果并没有直接IO到磁盘上,而是先存储在缓存(buffer)中,并在缓存中进行一些预排序来优化整个map的性能,存储map中间数据的缓存默认大小为100M，由io.sort.mb 参数指定。这个大小可以根据需要调整。当map任务产生了非常大的中间数据时可以适当调大该参数,使缓存能容纳更多的map中间数据，而不至于大频率的IO磁盘,当系统性能的瓶颈在磁盘IO的速度上,可以适当的调大此参数来减少频繁的IO带来的性能障碍。

由于map任务运行时中间结果首先存储在缓存中,默认当缓存的使用量达到80%(或0.8)的时候就开始写入磁盘,这个过程叫做spill(也叫溢出),进行spill的缓存大小可以通过io.sort.spill.percent 参数调整，这个参数可以影响spill的频率。进而可以影响IO的频率。

当map任务计算成功完成之后，如果map任务有输出，则会产生多个spill。接下来map必须将些spill进行合并,这个过程叫做merge, merge过程是并行处理spill的,每次并行多少个spill是由参数io.sort.factor指定的默认为10个。但是当spill的数量非常大的时候，merge一次并行运行的spill仍然为10个,这样仍然会频繁的IO处理,因此适当的调大每次并行处理的spill数有利于减少merge数因此可以影响map的性能。

当map输出中间结果的时候也可以配置压缩。

2. Reduce Task调优
reduce 运行阶段分为shuflle(copy) merge sort   reduce write五个阶段。

shuffle 阶段为reduce 全面拷贝map任务成功结束之后产生的中间结果,如果上面map任务采用了压缩的方式,那么reduce 将map任务中间结果拷贝过来后首先进行解压缩,这一切是在reduce的缓存中做的,当然也会占用一部分cpu。为了优化reduce的执行时间,reduce也不是等到所有的map数据都拷贝过来的时候才开始运行reduce任务，而是当job执行完第一个map任务时开始运行的。reduce 在shuffle阶段 实际上是从不同的并且已经完成的map上去下载属于自己的数据,由于map任务数很多,所有这个copy过程是并行的,既同时有许多个reduce取拷贝map，这个并行的线程是通过mapred.reduce.parallel.copies 参数指定，默认为5个,也就是说无论map的任务数是多少个，默认情况下一次只能有5个reduce的线程去拷贝map任务的执行结果。所以当map任务数很多的情况下可以适当的调整该参数，这样可以让reduce快速的获得运行数据来完成任务。

reduce线程在下载map数据的时候也可能因为各种各样的原因(网络原因、系统原因等），存储该map数据所在的datannode 发生了故障，这种情况下reduce任务将得不到该datanode上的数据了,同时该 download thread 会尝试从别的datanode下载,可以通过mapred.reduce.copy.backoff (默认为30秒)来调整下载线程的下载时间，如果网络不好的集群可以通过增加该参数的值来增加下载时间,以免因为下载时间过长reduce将该线程判断为下载失败。

reduce 下载线程在map结果下载到本地时,由于是多线程并行下载，所以也需要对下载回来的数据进行merge,所以map阶段设置的io.sort.factor 也同样会影响这个reduce的。

同map一样 该缓冲区大小也不是等到完全被占满的时候才写入磁盘而是默认当完成0.66的时候就开始写磁盘操作,该参数是通过mapred.job.shuffle.merge.percent 指定的。

当reduce 开始进行计算的时候通过mapred.job.reduce.input.buffer.percent 来指定需要多少的内存百分比来作为reduce读已经sort好的数据的buffer百分比,该值默认为0。Hadoop假设用户的reduce()函数需要所有的JVM内存，因此执行reduce()函数前要释放所有内存。如果设置了该值，可将部分文件保存在内存中(不必写到磁盘上)。
3. 参数
mapred.reduce.tasks（mapreduce.job.reduces）：</br>
默认值：1</br>
说明：默认启动的reduce数。通过该参数可以手动修改reduce的个数。</br>

mapreduce.task.io.sort.factor：</br>
默认值：10</br>
说明：Reduce Task中合并小文件时，一次合并的文件数据，每次合并的时候选择最小的前10进行合并。</br>

mapreduce.task.io.sort.mb：</br>
默认值：100</br>
说明： Map Task缓冲区所占内存大小。</br>

mapred.child.java.opts：</br>
默认值：-Xmx200m</br>
说明：jvm启动的子线程可以使用的最大内存。建议值-XX:-UseGCOverheadLimit -Xms512m -Xmx2048m -verbose:gc -Xloggc:/tmp/@taskid@.gc</br>

mapreduce.jobtracker.handler.count：</br>
默认值：10</br>
说明：JobTracker可以启动的线程数，一般为tasktracker节点的4%。</br>

mapreduce.reduce.shuffle.parallelcopies：</br>
默认值：5</br>
说明：reuduce shuffle阶段并行传输数据的数量。这里改为10。集群大可以增大。</br>

mapreduce.tasktracker.http.threads：</br>
默认值：40</br>
说明：map和reduce是通过http进行数据传输的，这个是设置传输的并行线程数。</br>

mapreduce.map.output.compress：</br>
默认值：false</br>
说明： map输出是否进行压缩，如果压缩就会多耗cpu，但是减少传输时间，如果不压缩，就需要较多的传输带宽。配合 mapreduce.map.output.compress.codec使用，默认是 org.apache.hadoop.io.compress.DefaultCodec，可以根据需要设定数据压缩方式。

mapreduce.reduce.shuffle.merge.percent：</br>
默认值： 0.66</br>
说明：reduce归并接收map的输出数据可占用的内存配置百分比。类似mapreduce.reduce.shuffle.input.buffer.percen属性。</br>

mapreduce.reduce.shuffle.memory.limit.percent：</br>
默认值： 0.25</br>
说明：一个单一的shuffle的最大内存使用限制。</br>

mapreduce.jobtracker.handler.count：</br>
默认值： 10</br>
说明：可并发处理来自tasktracker的RPC请求数，默认值10。</br>

mapred.job.reuse.jvm.num.tasks（mapreduce.job.jvm.numtasks）：</br>
默认值： 1</br>
说明：一个jvm可连续启动多个同类型任务，默认值1，若为-1表示不受限制。</br>

mapreduce.tasktracker.tasks.reduce.maximum：</br>
默认值： 2</br>
说明：一个tasktracker并发执行的reduce数，建议为cpu核数</br>





