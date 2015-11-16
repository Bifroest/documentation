# Operational Overview

## Cluster-Wide operations

### Retention changes

## All applications

### Important Configuration Parameters

- multiserver-pools, multiserver utilization
- statistic system


### Troubleshooting
 - If everything else fails, check these / document likeliness of these issues.
 - Statistic system.
 - Check in logs if either the cassandra nodes or bifroest-bifroest have problems. (Look out for cassandra exceptions otherwise check bifroest itself)
        - If you cannot see anything in the logs just restart the service and look if the problem still exists
        - Possible Cause: Statistic System
            - To identify this issue: Go to graphite and create yourself a graph, or check the metric-files bifroest outputs:
              - Select all metrics under `Bifroest.$application.EventBus.Handler-\*.Utilization.Events.\*`
              - Derive all metrics, and scale them with 1e-9.
              - This is going to look like a CPU graph.
	- If there is no more time spent in Idle and all time is spent in other events, increase the number of threads for the statistic system in the config.

## Stream Rewriter

 - TODO: expected behaviour after a start
 - TODO: note: If entire bifroest cluster was down, restart all SRs.

In order to start and stop the stream-rewriter 

### Important Configuration Parameters
- Should be located under /etc/bifroest/stream-rewriter
- 

### Troubleshooting
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

