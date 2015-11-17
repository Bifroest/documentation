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
            - Select the metric `Bifroest.$application.EventBus.Handler-*.Utilization.Events.idle.TimeSpent`
            - Derive it, and scale it with 1e-9.
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
c.g.p.b.b.BasicClient | INFO | Connecting to initial nodes [NodeMetadata [funnyNameForDebugging=RidiculousQuokka, nodeIdentifier=255f8edb-bd69-41ea-af96-ef3338111ac7, address=caching04.bifroest.acme.org, ports=DeserializedPortMap{clusterPort=5300, includeMetricPort=5102, fastIncludeMetricPort=5200, getMetricPort=5100, getMetricSetPort=5100, getSubmetricPort=5101}],...] 
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


After this, the bootloader announces successful startup:

```
c.g.p.c.b.BootLoaderNG | INFO | Service startup successful.
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

### Important Configuration Parameters

The clustering system needs special configuration. It needs to contain the cluster-port for this node, the seeds for the bifroest cluster, and the ping-frequency between bifroests. The configuration looks about like this:

```
{
    "clustering" : {
        "cluster-port" : 5300,
        "seeds" : [
            { "host" : "caching01.bifroest.acme.org", "port" : 5000 },
            { "host" : "caching02.bifroest.acme.org", "port" : 5000 },
            { "host" : "caching03.bifroest.acme.org", "port" : 5000 }
        ],
        "ping-frequency" : "10s"
    }
}
```

First, note that there are two different ports involved here, in this case, 5000 and 5300. Port 5000 must be a multi-server command port which supports the command "get-cluster-state". For the configuration of this system,
check the commons project. These ports are different, because a starting bifroest is going to use the normal
multi-server "get-cluster-state" command in order to get the current nodes in the cluster. After this, the
bifroest nodes will open up a persistent connection on the cluster port for additional cluster maintenance.
Furthermore, the cluster ports don't need to be identical on differnet bifroest nodes - the cluster is
going to pass portmappings for each node around to figure this out.

Furthermore, these seed nodes behave like cassandra seed nodes, or -- if you use that -- elasticsearch seed
nodes. At least one of the seed nodes must be available if a new node is supposed to join an existing bifroest
cluster. Otherwise, nodes are going to build their own cluster, potentially leading to split-brain situations.

Finally, the bifroest-nodes ping each other periodically to figure out if a specific node is down. Having
too frequent pings risks that garbage collections might cause a node to be considered down, while having
too high pings might easily overlook downed nodes.

### Normal Startup
During startup, the following critical systems are booted:

 - The cassandra System
 - The Metric Caches
 - The bifroest clustering

Normal startup of the **cassandra system** is the same as the stream rewriter.

Normal startup of the **metric cache system** looks like this:

```
2015-11-17T13:14:15,194 | c.g.p.c.b.BootLoaderNG | INFO | Booting graphite.bifroest.metric-cache
2015-11-17T13:14:15,264 | c.g.p.g.m.MetricCache | INFO | Created cache for daily
2015-11-17T13:14:15,315 | c.g.p.g.m.MetricCache | INFO | Created cache for 1minute
2015-11-17T13:14:15,752 | c.g.p.g.m.MetricCache | INFO | Created cache for hourly
2015-11-17T13:14:16,191 | c.g.p.g.m.MetricCache | INFO | Created cache for 5minutes
2015-11-17T13:14:16,816 | c.g.p.g.m.MetricCache | INFO | Created cache for 10seconds
```

Normal start of the **bifroest clustering system** looks like this:

First, the node announces it's own metadata:
```
2015-11-17T13:14:16,949 | c.g.p.g.c.ClusterSystem | INFO | I am NodeMetadata [funnyNameForDebugging=ZappyFrisbee, nodeIdentifier=ec29bdf0-90df-4498-898b-afa56bf1a344, address=caching02.bifroest.acme.org, ports=BifroestPortmap{clusterPort=5300, includeMetricPort=5102, fastIncludeMetricPort=5200, getMetricPort=5100, getMetricSetPort=5100, getSubmetricPort=5101}]
```

After this, the node is going to check all bifroest seeds from the configuration for existing cluster
metadata. If the node has discovered an existing cluster, the node state will announce this and change it's
state to joining the cluster:

```
c.g.p.g.c.s.NodeIsLonelyAndSad | INFO | Found ClusterState Optional[com.goodgame.profiling.bifroest.bifroest_client.metadata.ClusterState@705fca1]
c.g.p.g.c.BifroestClustering | INFO | start clustering -> State change: NodeIsLonelyAndSad -> NodeIsJoining
```

In this state, the node is waiting for the updated cluster mapping. Once the node received the new
mapping, it will change it's state into a full member node:

```
c.g.p.g.c.BifroestClustering | INFO | reacting to new mapping -> State change: NodeIsJoining -> NodeIsMember
```

After this initial joining, the node is going to keep preloading metrics as the cluster tells the new node
what metrics have been requested:

```
c.g.p.g.c.BifroestClustering | INFO | reacting to transferred metrics -> State change: NodeIsMember -> NodeIsMember
```

After this, bifroest announces successful startup:

```
c.g.p.c.b.BootLoaderNG | INFO | Service startup successful.
```

#### Do's & Donts
- Don't (re)start bifroest-caches too quickly one after each other. If a starting bifroest is unable to find a running bifroest cluster, it will start to bootstrap it's own cluster. This can lead to a split-brain situation. In this case, restart all bifroest nodes except for one seed leader. If there is no seed leader, restart all nodes but one seed node.

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

