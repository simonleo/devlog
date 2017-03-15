Spark RDD 的元素顺序问题
---  
---  
### 一个工作场景  

最近工作当中有遇到这样一个需求，我和一位同事需要对同一批数据进行离散化处理，他用于模型训练，我用于使用他训练出来的模型算分做排序。由于离散化处理细节太多，比较复杂，为了保证两边处理完全一致，我们希望把两边离散化的结果做一个比对。大致过程是：在 hdfs 上找到源文件读取这一批待处理的数据，在 Spark 上做一系列计算，然后输出为文件，将文件内容与同事的结果做比对。  

不过这就涉及到一个问题：读取源文件，分成 n 片后处理，再收集回来，数据还是原来的顺序吗？之前没有想过这个问题，于是好好研究了一下。  

### 读取数据的顺序
我们知道，HDFS 中的文件数据在物理上是以分块(block)的形式存储的，默认大小是 128M ，包括副本分散在 HDFS 集群的各个机器中。namenode 维护着这些文件的元数据，给客户端提供了一个统一的抽象访问接口，包括在读取文件时能按照写入时数据的顺序(假如是串行读取)。  

另外还有另外一种直接创建 RDD 的方式，即 sc.parallelize(seq, numSlices)，将一个序列并行化，分成 numSlices 个区间。这里面是怎么划分数据的呢，看下源码，实际上该方法返回的是 ParallelCollectionRDD ，查看 ParallelCollectionRDD 的实现  

    override def getPartitions: Array[Partition] = {
        val slices = ParallelCollectionRDD.slice(data, numSlices).toArray
        slices.indices.map(i => new ParallelCollectionPartition(id, i, slices(i))).toArray
    }  

slice 方法是定义在 object ParallelCollectionRDD 里的方法  

    def slice[T: ClassTag](seq: Seq[T], numSlices: Int): Seq[Seq[T]] = {
        if (numSlices < 1) {
            throw new IllegalArgumentException("Positive number of slices required")
        }
        // Sequences need to be sliced at the same set of index positions for operations
        // like RDD.zip() to behave as expected
        def positions(length: Long, numSlices: Int): Iterator[(Int, Int)] = {
            (0 until numSlices).iterator.map { i =>
            val start = ((i * length) / numSlices).toInt
            val end = (((i + 1) * length) / numSlices).toInt
            (start, end)
            }
        }
        seq match {
            case r: Range =>
            positions(r.length, numSlices).zipWithIndex.map {   
                case ((start, end), index) =>
                // If the range is inclusive, use inclusive range for the last slice
                if (r.isInclusive && index == numSlices - 1) {
                    new Range.Inclusive(r.start + start * r.step, r.end, r.step)
                }
                else {
                    new Range(r.start + start * r.step, r.start + end * r.step, r.step)
                }
            }.toSeq.asInstanceOf[Seq[Seq[T]]]
            case nr: NumericRange[ _ ] =>
            // For ranges of Long, Double, BigInteger, etc
            val slices = new ArrayBuffer[Seq[T]](numSlices)
            var r = nr
            for ((start, end) <- positions(nr.length, numSlices)) {
                val sliceSize = end - start
                slices += r.take(sliceSize).asInstanceOf[Seq[T]]
                r = r.drop(sliceSize)
            }
            slices
            case _ =>
            val array = seq.toArray // To prevent O(n^2) operations for List etc
            positions(array.length, numSlices).map { case (start, end) =>
            array.slice(start, end).toSeq
            }.toSeq
        }
    }
可见仍然是按照 seq 的顺序，从前往后将每 (length / numSlices) 个元素放入一个 Partition 中的(最后一个分区可能没有 length/numSlices 个，放到最后一个)。

### RDD操作时的顺序
我们再来看 RDD 操作时给数据顺序带来的影响，先考虑不带 Shuffle 的转换操作以及执行操作。  

转换操作比较好理解，只要不发生数据 Shuffle 的过程，在每个分区上数据总是一个个顺序处理的。比如 map 操作和 mapPartitions 操作：(f) => Iterator(f), mapPartitions(f) => f(Iterator), 无非是迭代器放在处理函数外面还是里边的区别。其他的算子也都大同小异。  

coalesce(numPartitions, shuffle) 比较复杂，其第二个参数决定会不会发生 Shuffle ，有 Shuffle 的情况下面会说，而 shuffle == flase 的情况呢，会将 parentRDD 的 partition 个数进行调整(减少)。


### Shuffle带来的影响
发生 Shuffle 后，数据可就打乱了，自然也无顺序可言了，写了个例子验证一下：  

    val num = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12)
    val reduce = sc.parallelize(num, 4).map(kv => (kv, 1)).groupByKey().map(kv => kv._ 1).collect
打印出 reduce 的结果是： 4, 8, 12, 1, 5, 9, 2, 6, 10, 3, 7, 11  
很显然由于给每个分区的数据进行了重分区(HashPartitioner)，每个分区计算完收集起来的结果也是跟之前顺序不一样的了。  

由此可见，只要 RDD 计算没有 Shuffle 的过程，我们就可以认为源数据的顺序没有打乱。我做比对的结果也证明了这一点，在读取源文件数据并分区后，做了一系列处理(没有Shuffle)，最后 coalesce(1, false) ，输出为文件，顺序完全跟之前的一致！  

注意：本文探讨的都是相对常见、直观一些的处理流程，文字也不够严谨，一些复杂的场景比如 RDDa.join(RDDb) ，可能存在没有 Shuffle 但有 map-side Combine 的情况，其实也会打乱之前的顺序。其实研究数据顺序并不是目的，工程场景中也很少遇到要明确顺序的情况，主要是为了从另一个角度思考加深对 Spark 计算原理的理解。

#### 参考文档
