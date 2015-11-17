# Operational Overview

## Cluster-Wide operations

### Retention changes

## All applications

### Important Configuration Parameters

- multiserver-pools, multiserver utilization
- statistic system


### Shared possible problems

#### Overloaded Statistic System

All systems in the bifroest stack depend on the statistic system to 
provide the strong self-monitoring. However, the statistic system 
has potential to block the entire bifroest service if the event bus
fills up.

 - To identify this issue:
     - If graphite still outputs graphs:
         - Create yourself a graph:
            - Select all metrics under `Bifroest.$application.EventBus.Handler-*.Utilization.Events.*`
            - Derive all metrics, and scale them with 1e-9.
            - If there is no or very little time spent in idle, the statistic system is unable
              to handle the overall load of the system.
     - Otherwise, check the metric file with watch and grep for Utilization.Events.Idle. If this metric is
       increasing very little, the event system is always busy crunching events and overall, overloaded.

In order to mitigate this issue, increase the number of handlers in the statistic system of the service:

```
{
  "statistics" : {
-    "handler-count" : 1,
+    "handler-count" : 2,
  }
}
```

#### Not enough buffer space in the EventBus

If the application is peaking heavily with events, it is possible that the statistic system could
handle the load on average, but the service is currently trying to cram 60k events into a bus with
around 4k storage capacity. That's not going to work. 

 - To identify this issue:
     - Look at the number of events fired per second, either in Graphite (Bifroest.$application.EventBus.EventsFired),
       or in the metrics file and derive it. In the file, this requires some eye-balling, but we're mostly
       interested in the magnitude.
     - Look at the statistic system configuration, and the size-exponent of the eventbus. The overall buffer
       capacity of the eventbus is around 2^${size-exponent}. 

In order to mitigate this issue, ensure that the buffer size is around 100 times as large as the events per
second. (Yup, that means bifroest with something like 10k events should run with an event bus of 100k - 1m
events, which is a size exponent of around 20). If this is not sufficient, increase it:

```
{
  "statistics" : {
-    "size-exponent" : 10,
+    "size-exponent" : 20,
  }
}
```

## Stream Rewriter

### Expected Behaviour after a start:

#### Normal startup

During normal startup, the stream rewriter at least has to start up the following critical system in roughly
this order:

    - The bifroest-client system. This system is the connection to the bifroest-cluster
    - The cassandra connection system. This system provides a persistent cassandra connection.
    - The DBNator System. This system is reponsible for clearing up the queue of metrics filled by
      the netty-system.
    - The netty-system. This system is responsible for accepting metrics in the plaintext carbon format.

Normal startup of the **bifroest client system** looks roughly like this:

```
c.g.p.c.b.BootLoaderNG | INFO | Booting bifroest.bifroest-client
c.g.p.b.b.BasicClient | INFO | Connecting to initial nodes [NodeMetadata [funnyNameForDebugging=RidiculousQuokka, nodeIdentifier=255f8edb-bd69-41ea-af96-ef3338111ac7, address=caching04.bifroest-sglimm.meow.ggs-net.com, ports=DeserializedPortMap{clusterPort=5300, includeMetricPort=5102, fastIncludeMetricPort=5200, getMetricPort=5100, getMetricSetPort=5100, getSubmetricPort=5101}],...] 
c.g.p.b.b.BasicClient | INFO | New mapping: com.goodgame.profiling.bifroest.balancing.KeepNeighboursMapping@791fbb3d
```

The first line logged by the BasicClient should contain a list of all bifroest-bifroest instances in your bifroest cluster.
After this line, you'll see the TaskRunner log some stuff about pings to the various bifroests. Finally, new
"new mapping" logging indicates that bifroest-bifroest has sent this client a new cluster mapping. After this,
the bifroest-client is ready to send metrics to the bifroest cluster.

Normal startup of the **cassandra client system** looks roughly like this:

```
c.d.d.c.Cluster | INFO | New Cassandra host $fqdn/$ip:9042 added
c.d.d.c.Cluster | INFO | New Cassandra host $fqdn/$ip:9042 added
c.d.d.c.Cluster | INFO | New Cassandra host $fqdn/$ip:9042 added
c.d.d.c.Cluster | INFO | New Cassandra host $fqdn/$ip:9042 added
```

This should list all cassandra nodes on your cassandra cluster. After this, the cassandra system is ready to
fire query towards cassandra.

Normal startup of the DBNator system and the netty system are quite short:

```
c.g.p.c.b.BootLoaderNG | INFO | Booting systems.stream-rewriter.db
c.g.p.c.b.BootLoaderNG | INFO | Booting systems.stream-rewriter.netty
c.g.p.s.n.NettySystem | INFO | Opening port!
```

After this, port 9003 is open and the stream rewriter is ready for operation. 

#### Problems during startup.

Primarily the client-systems can fail. 

 - If all bifroest-seeds are down, the bifroest-client system will not start
 - If all cassandra nodes are down, the cassandra-client system will not start.

This is usually communicated with a number of exceptions.


 - TODO: note: If entire bifroest cluster was down, restart all SRs.

In order to start and stop the stream-rewriter 

### Important Configuration Parameters
- Should be located under /etc/bifroest/stream-rewriter
- 

### Troubleshooting
- Check in logs if either the cassandra nodes or bifroest-bifroest have problems. (Look out for cassandra exceptions otherwise check bifroest itself)
- What do to if port 9003 suddenly closes?
	- Check the general troubleshooting steps
- There are no more metrics coming to bifroest?!
	- For the bifroest nodes, check the metrics `Bifroest.Bifroest.metrics-on-wrong-node.dropped.TreeAndCacheDrain`. Currently, it is possible that the stream-rewriter misses a cluster update in bifroest-bifroest when adding a new node. After this, the stream-rewriter will send metrics to the wrong bifroest-instance, which will in turn drop the metric. In this case, restart all stream rewriters asap.

## Bifroest

 - TODO: expected behaviour after a start
 - TODO: talk about preloading stuff.

#### Do's & Donts
- Don't (re)start bifroest-caches too quickly one after each other. If a starting bifroest is unable to find a running bifroest cluster, it will start to bootstrap it's own cluster. This can lead to a split-brain situation. In this case, restart all leader bifroest nodes except for one.

### Important Configuration Parameters

 - TODO: Bifroest Seeds
 - TODO: Port

### Troubleshooting
- Bifroest slows down
	- Possible Cause: Cache is full.
	    - To identify this issue: Check the metrics Bifroest.Bifroest.Caches.\*.{Evictions, Size, MaxSize}. If the number of evictions is rising sharply, bifroest is unable to cache the requested metrics.
	    - To fix this issue: Add more bifroest-nodes, or increase the cache sizes on individual bifroest nodes. Take care of the JVM memory limits here.
	- Check the general troubleshooting section

- Bifroest is running out of file descriptors.
	- Possible Cause: Too much pressure on the fast include metric port.
		- To identify this issue, check if the process has tons and tons of CLOSED_WAIT sockets open.
		- To fix this problem, increase the number of bifroest nodes, or increase the number of threads for the netty system.

## Heimdall
- TODO: note: If entire bifroest cluster was down, restart all Heimdalls.

### Important Configuration Parameters

 - TODO: (Bifroest Seeds)

### Troubleshooting
- Check the general troubleshooting section

## Aggregator
### Important Configuration Parameters

 - TODO: (Cassandra Seeds)

### Troubleshooting
- Aggregations are slow!
	- As long as an aggregation of a table take less than a day, don't worry too much.
        - Once the aggregation of a single table takes more than a day, doom has arrived.

- Check the general troubleshooting section

