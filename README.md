# Whale

# 1. Introduction
With the ability to cope with the emerging big streams, Distributed Stream Processing Systems (DSPSs) have attracted much research efforts. 
To achieve high throughput and low latency, DSPSs exploit different parallel processing techniques. 
The one-to-many data partitioning strategy plays an important role in various applications. 
With one-to-many data partitioning, the upstream processing instance sends a generated tuple to a potentially large number of downstream processing instances. 
Existing DSPSs leverages the instance-oriented communication, where an upstream instance transmits a tuple to different downstream instances separately. 
However, in one-to-many data partitioning, the multiple downstream instances typically run on the same machine to exploit multi-core resources. 
Therefore,  a DSPS actually sends a same data item to a machine multiple times, raising significant unnecessary costs for serialization and communications. 
We show that such a mechanism can lead to serious performance bottleneck due to CPU overload. 

To address the problem, we design and implement Whale, an efficient RDMA-assisted distributed stream processing system. 
Two factors contribute to the efficiency of this design. 
First, we propose a novel RDMA-assisted stream multicast scheme with a self-adjusting non-blocking tree structure to alleviate the CPU workloads of an upstream instance during data partitioning. 
Second, we re-design the communication mechanism in existing DSPSs by replacing the instance-oriented communication with a new worker-oriented communication scheme, which saves significant costs for redundant serialization and communications. 
We implement Whale on top of Apache Storm and evaluate it using experiments with large-scale datasets. The results show that Whale achieves 56.6x improvement of system throughput and 97% reduction of processing latency compared to existing designs.

# 2. Architecture of Whale
![image](https://github.com/Whale2021/Whale/blob/master/images/Whale_architecture.png)

Whale includes two main components: stream multicast and worker-oriented communication. The former fully exploits the zero-copy RDMA network to efficiently process highly dynamic streams. The stream multicast scheduler constructs a non-blocking multicast tree which elaborately chooses the maximum out-degree and minimizes the average multicast latency for an incoming stream. To cope with stream dynamics, the system workload monitor periodically monitors the current workloads of the source instance while the multicast controller adjusts the dynamic multicast structure accordingly. The work-oriented communication aims at reducing the redundant serialization and communication. The batch component packages the IDs with the data item into a BatchTuple and serializes BatchTuple for multicasting. The dispatcher deserializes BatchTuple and dispatches the receiving tuple to the locally hosted destination instances according to the obtained instance IDs.

# 3. How to use?
## 3.1 Environment
We deploy the Whale system on a cluster consisting of 30 machines. Each machine is equipped with a 16-core 2.6GHz Intel(R) Xeon(R) CPU , 64GB RAM, 256GB HDD, Red Hat 6.2 system, a Mellanox InfiniBand FDR 56Gbps NIC and a 1Gbps Ethernet NIC. One machine is configured to be the master node running as nimbus, and the others machines serve as worker nodes running as supervisors.

## 3.2 Building Whale
Before building Whale, developers should first build Apache Storm version 2.0.0-SNAPSHOT source code and install the library in local warehouse.

If you already deploy the Apache Storm (version 2.0.0-SNAPSHOT) cluster environment, you only need to replace these jars to `$STORM_HOME/lib` and `$STORM_HOME/lib-worker`
> * storm-client-2.0.0-SNAPSHOT.jar
> * whale-rdma-2.0.0-SNAPSHOT.jar

Dependent on RDMA jars
> * disni-1.6.jar
> * rdmachannel-core-1.0-SNAPSHOT.jar

### storm-client.jar
Storm-client module source code is maintained using [Maven](http://maven.apache.org/) (version 3.5.4). Generate the executable jar by running
```
$ cd storm-client
$ mvn clean install -Dmaven.test.skip=true -Dcheckstyle.skip=true
```

### whale-rdma.jar
Whale-rdma module source code is maintained using [Maven](http://maven.apache.org/). Before build the whale-rdma module, you needd to generate whale-multicast and whale-common. Generate the executable jar by running
```
$ cd whale-multicast
$ mvn clean install -Dmaven.test.skip=true -Dcheckstyle.skip=true
$ cd ../whale-common
$ mvn clean install -Dmaven.test.skip=true -Dcheckstyle.skip=true
```
After you install the whale-multicast.jar and whale-common.jar, you can generate the executable whale-rdma.jar by running
```
$ cd whale-rdma
$ mvn clean install -Dmaven.test.skip=true -Dcheckstyle.skip=true
```

### disni.jar and rdmachannel-core.jar
Whale use RDMA Verbs by [Disni](https://github.com/zrlio/disni) , then we further encapsulate the RDMA Verbs primitive [RDAM-Channel](https://github.com/Whale2021/RDMAChannel) to make it more practical and efficient.

Building disni.jar and rdmachannel-core.jar, you just only see [RDMA-Channel](https://github.com/Whale2021/RDMAChannel) project. It includes all environments running RDMA-channel.

# 4. Whale Benchmark
## 4.1 Buidling Benchmark
Whale benchmark code is maintained using [Maven](http://maven.apache.org/). Before generate the executable jar, you should build and install the benchmark-common model. Do all these steps by running
```
$ cd benchmark/benchmark-common
$ mvn clean install -Dmaven.test.skip=true -Dcheckstyle.skip=true
$ cd ../benchmark-didiOrderMatch
$ mvn clean package -Dmaven.test.skip=true -Dcheckstyle.skip=true
```
Then you get the didi benchmark benchmark-didiOrderMatch.jar.

Generate the executable jar of benchmark-stockDeal by running 
```
$ cd benchmark/benchmark-stockDeal
$ mvn clean package -Dmaven.test.skip=true -Dcheckstyle.skip=true
```

Then you get the stock benchmark benchmark-stockDeal.jar.

## 4.2 Running Benchmark
After deploying a Whale cluster, you can launch Whale by submitting its jar to the cluster. Please refer to Storm documents for how to
[set up a Storm cluster](https://storm.apache.org/documentation/Setting-up-a-Storm-cluster.html) and [run topologies on a Storm cluster](https://storm.apache.org/documentation/Running-topologies-on-a-production-cluster.ht)

```
$ storm jar benchmark-didiOrderMatch-2.0.0-SNAPSHOT.jar org.apache.storm.benchmark.multicast.MulticastModelBalancedParitalBenchTopology BenchTopology <ordersTopic> <numMachines> 1 <parallelism> 3 rdma 1  
$ storm jar benchmark-stockDeal-2.0.0-SNAPSHOT.jar org.apache.storm.benchmark.stockdeal.StockDealBalancedPartialBenchTopology StockDeal <stockTopic> <numMachines> 1 <parallelism> 3 rdma 1
```

(Didi data and NASDAQ data have to be import into Kafka before running)
## License
Whale is released under the [Apache 2 license](http://www.apache.org/licenses/LICENSE-2.0.html).
