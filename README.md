# hadoop-Spark-Tuning

### 1 - Data Serialization
Serialization plays an important role in the performance of any distributed application. Formats that
are slow to serialize objects into, or consume a large number of bytes, will greatly slow down the
computation.  
Spark provides two serialization libraries:
- Java serialization
By default, Spark serializes objects using Java’s ObjectOutputStream
framework, and can work with any class you create that implements java.io.Serializable.  
- Kryo serialization
Spark can also use the Kryo library (version 2) to serialize objects more
quickly. Kryo is significantly faster and more compact than Java serialization (often as much as
10x)
```sh
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer"). 

```
This setting configures the serializer used for not only shuffling data between worker nodes but also when serializing RDDs to disk.


### 2 - Memory Tuning
There are three considerations in tuning memory usage: the amount of memory used by your
objects (you may want your entire dataset to fit in memory), the cost of accessing those objects, and
the overhead of garbage collection (if you have high turnover in terms of objects).

The best way to size the amount of memory consumption a dataset will require is to create an RDD,
put it into cache, and look at the “Storage” page in the web UI. The page will tell you how much
memory the RDD is occupying.
To estimate the memory consumption of a particular object, use SizeEstimator’s estimate method
This is useful for experimenting with different data layouts to trim memory usage, as well as
determining the amount of space a broadcast variable will occupy on each executor heap.

### 3 - Serialized RDD Storage
When your objects are still too large to efficiently store despite this tuning, a much simpler way to
reduce memory usage is to store them in serialized form, using the serialized StorageLevels in the
RDD persistence API, such as MEMORY_ONLY_SER. Spark will then store each RDD partition as one
large byte array. The only downside of storing data in serialized form is slower access times, due to
having to deserialize each object on the fly. We highly recommend using Kryo if you want to cache
data in serialized form, as it leads to much smaller sizes than Java serialization (and certainly than
raw Java objects).

parsedData3.persist(StorageLevel.MEMORY_ONLY_SER)
https://spark.apache.org/docs/1.5.0/programming-guide.html#rdd-persistence


### 4 - Garbage Collection Tuning
One important configuration parameter for GC is the amount of memory that should be used for
caching RDDs. By default, Spark uses 60% of the configured executor memory
(spark.executor.memory) to cache RDDs. This means that 40% of memory is available for any
objects created during task execution.
In case your tasks slow down and you find that your JVM is garbagecollecting
frequently or running
out of memory, lowering this value will help reduce the memory consumption. To change this to, say,
50%, you can call conf.set("spark.storage.memoryFraction", "0.5") on your SparkConf.
Combined with the use of serialized caching, using a smaller cache should be sufficient to mitigate
most of the garbage collection problems. In case you are interested in further tuning the Java GC,
continue reading below.

### 5 - Memory Usage of Reduce Tasks
Sometimes, you will get an OutOfMemoryError not because your RDDs don’t fit in memory, but
because the working set of one of your tasks, such as one of the reduce tasks in groupByKey, was
too large. Spark’s shuffle operations (sortByKey, groupByKey, reduceByKey, join, etc) build a hash
table within each task to perform the grouping, which can often be large. The simplest fix here is to
increase the level of parallelism, so that each task’s input set is smaller. Spark can efficiently support
tasks as short as 200 ms, because it reuses one executor JVM across many tasks and it has a low
task launching cost, so you can safely increase the level of parallelism to more than the number of
cores in your clusters.

### 6 - Broadcasting Large Variables
Using the broadcast functionality available in SparkContext can greatly reduce the size of each
serialized task, and the cost of launching a job over a cluster. If your tasks use any large object from
the driver program inside of them (e.g. a static lookup table), consider turning it into a broadcast
variable. Spark prints the serialized size of each task on the master, so you can look at that to decide whether your tasks are too large; in general tasks larger than about 20 KB are probably
worth optimizing.
### 7 - Level of Parallelism
Clusters will not be fully utilized unless you set the level of parallelism for each operation high
enough. Spark automatically sets the number of “map” tasks to run on each file according to its size
(though you can control it through optional parameters to SparkContext.textFile, etc), and for
distributed “reduce” operations, such as groupByKey and reduceByKey, it uses the largest parent
RDD’s number of partitions. You can pass the level of parallelism as a second argument (see the
spark.PairRDDFunctions documentation), or set the config property spark.default.parallelism to
change the default. In general, we recommend 23
tasks per CPU core in your cluster
