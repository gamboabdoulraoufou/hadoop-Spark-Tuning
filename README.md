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
### 3 - Serialized RDD Storage
### 4 - Garbage Collection Tuning
### 5 - Memory Usage of Reduce Tasks
### 6 - Broadcasting Large Variables
### 7 - Level of Parallelism
