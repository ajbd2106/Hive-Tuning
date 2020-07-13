# Hive-Tuning

set tez.am.resource.memory.mb=4096
set tez.task.resource.memory.mb=4096
set tez.am.java.opts=-Xmx6144m;
set tez.am.resource.memory.mb=4096;
set hive.tez.container.size=4096;
MR

set hive.execution.engine=mr;
set mapreduce.map.memory.mb=4096;
set mapreduce.reduce.memory.mb=4096;
set mapreduce.map.java.opts=-Xmx6144m;
set mapreduce.reduce.java.opts=-Xmx6144m;
and also these Yarn settings

set yarn.nodemanager.vmem-check-enabled=false;
set yarn.nodemanager.resource.memory-mb=98304;
set yarn.scheduler.minimum-allocation-mb=8192;
set yarn.scheduler.maximum-allocation-mb=98304;
set yarn.nodemanager.vmem-pmem-ratio=9;
--

set tez.am.resource.memory.mb=8192
set tez.task.resource.memory.mb=8192
set tez.am.java.opts=-Xmx6144m;
set tez.am.resource.memory.mb=8192;
set hive.tez.container.size=8192;
# First of all you need to understand heap is a subset of container. Your heap memory should be approximately 80% of container memory.

set hive.execution.engine=mr;  
set mapreduce.map.memory.mb=4096;  -- this is container memory
set mapreduce.reduce.memory.mb=4096;
Below values are wrong. They have to be less than 4096 or you will always get container running beyond memory limits issue.

set mapreduce.map.java.opts=-Xmx6144m;  -- this is heap memory
set mapreduce.reduce.java.opts=-Xmx6144m;
Instead set these to :

set mapreduce.map.java.opts=-Xmx3276m;   -- (80% of 4096)
set mapreduce.reduce.java.opts=-Xmx3276m;

------------

https://community.cloudera.com/t5/Community-Articles/Demystify-Apache-Tez-Memory-Tuning-Step-by-Step/ta-p/245279

Step 1 - Determine your YARN Node manager Resource Memory (yarn.nodemanager.resource.memory-mb) and your YARN minimum container size (yarn.scheduler.minimum-allocation-mb). Your yarn.scheduler.maximum-allocation-mb is the same as yarn.nodemanager.resource.memory-mb.

yarn.nodemanager.resource.memory-mb is the Total memory of RAM allocated for all the nodes of the cluster for YARN. Based on the number of containers, the minimum YARN memory allocation for a container is yarn.scheduler.minimum-allocation-mb. yarn.scheduler.minimum-allocation-mb will be a very important setting for our Tez Application Master and Container sizes.

So how do we determine this with just the number of cores, disks, and RAM on each node? The Hortonworks easy button approach. Follow the instructions at this link, Determine HDP Memory Config.

For example, if you are on HD Insight running a D12 node with 8 CPUs and 28GBs of memory, with no HBase, you run:

Run python yarn-utils.py -c 8 -m 28 -d 2 -k False
Your output would look like this.

1788-screen-shot-2016-02-02-at-54017-pm.png

In Ambari, configure the appropriate settings for YARN and MapReduce or in a non-Ambari managed cluster, manually add the first three settings in yarn-site.xml and the rest in mapred-site.xml on all nodes.

-----------------------------------------------------------------

Step 2 - Determine your Tez Application Master and Container Size, that is tez.am.resource.memory.mb and hive.tez.container.size.

Set tez.am.resource.memory.mb to be the same as yarn.scheduler.minimum-allocation-mb the YARN minimum container size.

Set hive.tez.container.size to be the same as or a small multiple (1 or 2 times that) of YARN container size yarn.scheduler.minimum-allocation-mb but NEVER more than yarn.scheduler.maximum-allocation-mb. You want to have headroom for multiple containers to be spun up.

A general guidance: Don't exceed Memory per processors as you want one processor per container. So if you have for example, 256GB and 16 cores, you don't want to have your container bigger than 16GB.

Bonus:

Container Reuse set to True: tez.am.container.reuse.enabled (Default is true)
Prewarm Containers when HiveSever2 Starts, under Hive Configurations in Ambari.
1792-screen-shot-2016-02-03-at-114441-pm.png

-----------------------------------------------------------------

Step 3 - Application Master and Container Java Heap sizes (tez.am.launch.cmd-opts and hive.tez.java.ops respectively)

By default these are BOTH 80% of the container sizes, tez.am.resource.memory.mb and hive.tez.container.sizerespectfully.

NOTE: tez.am.launch.cmd-opts is automatically set, so no need to change this.

In HDP 2.3 and above, no need to also set hive.tez.java.ops as it can be automatically set controlled by a new property tez.container.max.java.heap.fraction which is defaulted to 0.8 in tez-site.xml. This property is not by default in Ambari. If you wish you can add it to the Custom tez-site.sml.

1790-screen-shot-2016-02-03-at-110536-pm.png

As you can see from Ambari, in Hive -> Advance configurations, there are no manual memory configurations set for hive.tez.java.opts

1789-screen-shot-2016-02-03-at-105852-pm.png

if you wish to make the heap 75% of the container, then set the Tez Container Java Heap Fraction to 0.75


If you wish this set manually, you can add to hive.tez.java.ops for example -Xmx7500m -Xms 7500m, as longs as it is a fraction of hive.tez.container.size

-----------------------------------------------------------------

Step 4: Now to determine Hive Memory Map Join Settings parameters.

tez.runtime.io.sort.mb is the memory when the output needs to be sorted.

tez.runtime.unordered.output.buffer.size-mb is the memory when the output does not need to be sorted.

hive.auto.convert.join.noconditionaltask.size is a very important parameter to size memory to perform Map Joins. You want to perform Map joins as much as possible.

In Ambari this is under the Hive Confguration

1791-screen-shot-2016-02-03-at-112931-pm.png

For more on this see http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.3.4/bk_performance_tuning/content/ch_setting_me...


SET tez.runtime.io.sort.mb to be 40% of hive.tez.container.size.  You should rarely have more than2GB set.  

By default hive.auto.convert.join.noconditionaltask = true

SET hive.auto.convert.join.noconditionaltask.size  to 1/3 of hive.tez.container.size

SET tez.runtime.unordered.output.buffer.size-mb to 10% of hive.tez.container.size
-----------------------------------------------------------------

FOR MORE ADVANCED SETTINGS CONCERNING QUERY OPTIMIZATION

Step 5 - For for Query optimization and Mapper Parallelism see http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.3.4/bk_performance_tuning/content/ch_query_optim...

Step 6 - Determining Number of Mappers

The following parameters control the number of mappers for splittable formats with Tez:

set tez.grouping.min-size=16777216; -- 16 MB min split

set tez.grouping.max-size=1073741824; -- 1 GB max split

Increase min and max split size to reduce the number of mappers.
See also

How Initial task parallelism works

https://community.hortonworks.com/questions/905/how-are-number-of-mappers-determined-for-a-query-w.h...

---------------------------

References For Microsoft Azure and HDInsight

https://azure.microsoft.com/en-us/documentation/articles/hdinsight-hadoop-hive-out-of-memory-error-o...
https://azure.microsoft.com/en-us/documentation/articles/hdinsight-hadoop-optimize-hive-query/

---


SET hive.exec.dynamic.partition = true; 
SET hive.exec.dynamic.partition.mode = nonstrict; 
SET hive.exec.max.dynamic.partitions=500000;
SET hive.exec.max.dynamic.partitions.pernode=200000; 
SET parquet.memory.min.chunk.size=1000000;
SET hive.tez.dynamic.partition.pruning = true;
SET hive.tex.auto.reducer.parallelism = true;
SET hive.vectorized.execution.enabled=true;
SET hive.vectorized.execution.reduce.enabled = true;
SET hive.vectorized.execution.reduce.groupby.enabled = true;
SET yarn.nodemanager.resource.memory-mb=8192;
SET yarn.scheduler.minimum-allocation-mb=2048;
SET yarn.scheduler.maximum-allocation-mb=8192;
SET hive.tez.container.size=7168;
SET hive.tez.java.opts=-Xmx8192m;
SET hive.optimize.sort.dynamic.partition = true;

set hive.auto.convert.join=false;
SET hive.vectorized.execution.enabled=false;
SET hive.vectorized.execution.reduce.enabled=false;
SET hive.exec.compress.intermediate=true;
SET hive.exec.compress.output=true;
SET mapred.map.output.compression.codec=org.apache.hadoop.io.compress.SnappyCodec;
SET mapred.output.compression.codec=org.apache.hadoop.io.compress.SnappyCodec;
SET mapred.output.compression.type=BLOCK;

analyze table store_sales partition (...) compute statistics;

-----------
First of all you need to understand heap is a subset of container. Your heap memory should be approximately 80% of container memory.

set hive.execution.engine=mr;  
set mapreduce.map.memory.mb=4096;  -- this is container memory
set mapreduce.reduce.memory.mb=4096;
Below values are wrong. They have to be less than 4096 or you will always get container running beyond memory limits issue.

set mapreduce.map.java.opts=-Xmx6144m;  -- this is heap memory
set mapreduce.reduce.java.opts=-Xmx6144m;
Instead set these to :

set mapreduce.map.java.opts=-Xmx3276m;   -- (80% of 4096)
set mapreduce.reduce.java.opts=-Xmx3276m;
-

set tez.am.resource.memory.mb=8192
set tez.task.resource.memory.mb=8192
set tez.am.java.opts=-Xmx6144m;
set tez.am.resource.memory.mb=8192;
set hive.tez.container.size=8192;

SET hive.exec.reducers.bytes.per.reducer=1048576;
SET mapred.map.tasks=300;
SET mapred.reduce.tasks=300;
SET mapreduce.map.java.opts=-Xmx8192m;
SET mapreduce.reduce.java.opts=-Xmx8192m;

 set mapred.map.tasks=100;
 set mapred.reduce.tasks=100;
 set mapreduce.map.java.opts=-Xmx4096m;
 set mapreduce.reduce.java.opts=-Xmx4096m;
 set hive.exec.max.dynamic.partitions.pernode=100000;
 set hive.exec.max.dynamic.partitions=100000;



SET tez.am.resource.memory.mb=5120;
SET hive.auto.convert.join.noconditionaltask.size=4096MB;
SET hive.heapsize=4096MB;
SET hive.metastore.heapsize=5120MB;

hive.tez.container.size = 682MB
hive.heapsize = 4096MB
hive.metastore.heapsize = 1024MB
hive.exec.reducer.bytes.per.reducer = 1GB
hive.auto.convert.join.noconditionaltask.size = 2184.5MB
hive.tex.auto.reducer.parallelism = True
SET hive.tez.dynamic.partition.pruning = true;

tez.am.resource.memory.mb = 5120MB
tez.grouping.max-size = 1073741824 Bytes
tez.grouping.min-size = 16777216 Bytes
tez.grouping.split-waves = 1.7
tez.runtime.compress = True
tez.runtime.compress.codec = org.apache.hadoop.io.compress.SnappyCodec
 

   hiveContext.setConf("hive.exec.max.dynamic.partitions", "100000") 


 set hive.exec.max.dynamic.partitions.pernode=500000;
