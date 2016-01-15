---
layout: post
title: "ElasticSearch Practice"
description: "ElasticSearch Understanding & Tuning"
category: 
tags: ["es","tuning"]
---
{% include JB/setup %}
# **Introduction**
We have tried to setup ELK(elasticsearch-logstash-kibana) stack as parrt of our cloud monitor eco-system.
## Use Case
In our cloud eco-system, we will setup ELK to collect openstack control plane log and search for troubleshooting.
Also, we will preiodically send out VM availability metrics (with multiple fields [E.g. vpc, image name, VM_uuid, cos]) into ES and work as time-series major portal for KPI evaluating.

We'd like to share some performance tuning experience for elasticsearch, also leads to under elasticsearch from code perspective.

Below is our ELK version.

    Elasticsearch version: 1.5.3 
    Logstash version: 1.0.9
    Kibana version: 4.0.3
 Target OS Version: Ubuntu 14.04

# **Concept Highlights**
## *Understand ES Toplogy*
ES node has 3 role responsibilities as below. They're not exclusive roles. One instance node can take one or more roles (E.g. both matser & data node).

| Role        | settings           | Responsibility  |
| ------------- |:-------------:| -----:|
| Dedicated Master Node | node.master = true; node.data=false | Only participant to work as coordinator node & won't store index repo locally|
| Dedicated Data Node      | node.master = false; node.data=true | store index repo locally, won't work as coordinator node|
| clinet node | node.master = true; node.data=false; http.enabled=false     |    load balance node |

* Only when the nodes is enabled with master attribute, then it's eligible to do leader election

* For Detials: Please check [Elasticsearch Node](https://www.elastic.co/guide/en/elasticsearch/reference/master/modules-node.html)

Here's our ES Topo with 3 matser nodes & 3 data nodes. This architect is more reliable & isolate data & cluster control impact when each node is down. 
Also, master & data node has different hardware SKU requirement. Master node does not need large volume disk, but more powerful CPU Cores & enough memory.
Data node need large volume disk for index repository storage and enough memory for caching purpose.

![es-topo]({{ site.JB.IMAGE_PATH }}/es_topo.png "ES Topology Diagram")


# **Perf Tuning Tips**
## *Avoid Swap Out Memory*
When setting `bootstrap.mlockall=true`, elasticsearch try to lock the process address space into RAM, preventing any Elasticsearch memory frombeing swapped out.
> About bootstrap.mlockall option, Please check code for org.elasticsearch.bootstrap.Bootstrap

```java

private void setup(boolean addShutdownHook, Tuple<Settings, Environment>  tuple) throws Exception {
        if (tuple.v1().getAsBoolean("bootstrap.mlockall", false)) {
                    Natives.tryMlockall();
}
```

> org.elasticsearch.common.jna.CLibrary


```java
￼￼public static native int mlockall(int flags);
```
* **Linux OS Support for mlockall**
However, only enable the above option is not enough, it needs support from OS layer. Otherwise, we will encounter the below issue.

```
2015-05-24 17:44:49,247][WARN ][monitor.jvm              ][slc5b01c-6gjg-elasticsearch] [gc][old][75543][4255] duration [39.9s], collections[1]/[40.3s], total [39.9s]/[1.7d], memory [62.8gb]->[62.8gb]/[63.8gb], all_pools{[young] [768.5mb]->[768.7mb]/[1.4gb]}{[survivor] [0b]->[0b]/[191.3mb]}{[old][62.1gb]->[62.1gb]/[62.1gb]}[2015-05-24 17:45:24,257][WARN ][monitor.jvm] [slc5b01c-6gjg-elasticsearch] [gc][old][75545][4256] duration [32.5s], collections[1]/[32.9s], total [32.5s]/[1.7d], memory [62.8gb]->[62.8gb]/[63.8gb], all_pools{[young] [770.1mb]->[768.8mb]/[1.4gb]}{[survivor] [0b]->[0b]/[191.3mb]}{[old][62.1gb]->[62.1gb]/[62.1gb]}[2015-05-24 17:46:06,697][WARN ][monitor.jvm              ][slc5b01c-6gjg-elasticsearch] [gc][old][75547][4257] duration [39.9s], collections[1]/[40.3s], total [39.9s]/[1.7d], memory [62.8gb]->[62.8gb]/[63.8gb], all_pools{[young] [770.4mb]->[768.9mb]/[1.4gb]}{[survivor] [0b]->[0b]/[191.3mb]}{[old][62.1gb]->[62.1gb]/[62.1gb]}[2015-05-24 17:46:49,276][WARN ][monitor.jvm              ][slc5b01c-6gjg-elasticsearch] [gc][old][75549][4258] duration [40s], collections[1]/[40.4s], total [40s]/[1.7d], memory [62.8gb]->[62.8gb]/[63.8gb], all_pools{[young] [784.6mb]->[769mb]/[1.4gb]}{[survivor] [0b]->[0b]/[191.3mb]}{[old][62.1gb]->[62.1gb]/[62.1gb]}[2015-05-24 17:47:28,451][WARN ][monitor.jvm              ][slc5b01c-6gjg-elasticsearch] [gc][old][75550][4259] duration [38.8s], collections[1]/[39.1s], total [38.8s]/[1.7d], memory [62.8gb]->[62.8gb]/[63.8gb], all_pools{[young] [769mb]->[769.1mb]/[1.4gb]}{[survivor] [0b]->[0b]/[191.3mb]}{[old][62.1gb]->[62.1gb]/[62.1gb]}[2015-05-24 17:48:08,778][WARN ][monitor.jvm              ][slc5b01c-6gjg-elasticsearch] [gc][old][75551][4260] duration [39.9s], collections[1]/[40.3s], total [39.9s]/[1.7d], memory [62.8gb]->[62.8gb]/[63.8gb], all_pools{[young] [769.1mb]->[769.2mb]/[1.4gb]}{[survivor] [0b]->[0b]/[191.3mb]}{[old][62.1gb]->[62.1gb]/[62.1gb]}[2015-05-24 17:48:19,869][WARN ][common.jna               ] Unable to lock JVM memory(ENOMEM). This can result in part of the JVM being swapped out. IncreaseRLIMIT_MEMLOCK (ulimit).[2015-05-24 17:48:41,028][WARN ][monitor.jvm              ][slc5b01c-6gjg-elasticsearch] [gc][young][4][3] duration [1.5s], collections[1]/[2.5s], total [1.5s]/[12.6s], memory [2.4gb]->[2.4gb]/[63.8gb], all_pools {[young][735.6mb]->[22.3mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][1.5gb]->[2.2gb]/[62.1gb]}[2015-05-24 17:49:20,911][WARN ][monitor.jvm              ][slc5b01c-6gjg-elasticsearch] [gc][young][42][4] duration [2.7s], collections[1]/[2.8s], total [2.7s]/[15.3s], memory [3.8gb]->[3.6gb]/[63.8gb], all_pools {[young][1.4gb]->[20.8mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][2.2gb]->[3.4gb]/[62.1gb]}
```
* **Solution**

> 1. in "/usr/lib/systemd/system/elasticsearch.service", let LimitMEMLOCK=infinity
> 2. ulimit -l unlimited
> 3. service elasticsearch-elasticsearch restart

* **Sanity Check**

You can check http://<root_uri>/_nodes/process, will see process.mlockall is true.

![mlockall check]({{ site.JB.IMAGE_PATH }}/mlockall.png "Process Url for mlockall check")

# Field Data Cache
* **Background**

ES underneath lucen implementation (inverted index), it's like one concordances for whole book from key term mapping to whole content.
It's pretty efficient & useful solution for search request, but not friendly for key term aggregation.

ES field data cache is used mainly when sorting on or faceting on a field. It loads all the field values to memory in order to provide fast document based access to those values. The field data cache can be expensive to build for a field, so its recommended to have enough memory to allocate it, and to keep it loaded.

You need to control field data cache size in caution to avoid it become as major memory killer.


* **Field Data Cache Settings**
Http Post Reuqest with end point http:\\<ES_Root_URL>\_cluster\settings, related payload as below:
```
indices:  fielddata:    cache:
    	size: 40%
```

Otherwise, you will encounter the below issue "[FIELDDATA] Data too large" when lots of aggregation & faceting request to ES cluster.

```
Caused by: org.elasticsearch.common.breaker.CircuitBreakingException: [FIELDDATA] Datatoo large, data for [@timestamp] would be larger than limit of [41085134438/38.2gb]atorg.elasticsearch.common.breaker.ChildMemoryCircuitBreaker.circuitBreak(ChildMemoryCircuitBreaker.java:97)        atorg.elasticsearch.common.breaker.ChildMemoryCircuitBreaker.addEstimateBytesAndMaybeBreak(ChildMemoryCircuitBreaker.java:148)        atorg.elasticsearch.index.fielddata.RamAccountingTermsEnum.flush(RamAccountingTermsEnum.java:71)        atorg.elasticsearch.index.fielddata.RamAccountingTermsEnum.next(RamAccountingTermsEnum.java:85)
       atorg.elasticsearch.index.fielddata.ordinals.OrdinalsBuilder$3.next(OrdinalsBuilder.java:472)        atorg.elasticsearch.index.fielddata.plain.PackedArrayIndexFieldData.loadDirect(PackedArrayIndexFieldData.java:109)        atorg.elasticsearch.index.fielddata.plain.PackedArrayIndexFieldData.loadDirect(PackedArrayIndexFieldData.java:49)        atorg.elasticsearch.indices.fielddata.cache.IndicesFieldDataCache$IndexFieldCache$1.call(IndicesFieldDataCache.java:187)        atorg.elasticsearch.indices.fielddata.cache.IndicesFieldDataCache$IndexFieldCache$1.call(IndicesFieldDataCache.java:174)        atorg.elasticsearch.common.cache.LocalCache$LocalManualCache$1.load(LocalCache.java:4742)        atorg.elasticsearch.common.cache.LocalCache$LoadingValueReference.loadFuture(LocalCache.java:3527)        atorg.elasticsearch.common.cache.LocalCache$Segment.loadSync(LocalCache.java:2319)        atorg.elasticsearch.common.cache.LocalCache$Segment.lockedGetOrLoad(LocalCache.java:2282)        at org.elasticsearch.common.cache.LocalCache$Segment.get(LocalCache.java:2197)        ... 19 more[2015-03-26 20:55:18,280][DEBUG][action.search.type       ][lvs3b01c-7mmq-elasticsearch] [logstash-2015.03.27][0], node[6gYLm4m9RxmDk7F3Ok3IWQ],[R], s[STARTED]: Failed to execute[org.elasticsearch.action.search.SearchRequest@4b0172d1] lastShard [true]org.elasticsearch.transport.RemoteTransportException:[slc3b02c-3bpj-elasticsearch][inet[/10.132.155.121:9300]][indices:data/read/search[phase/query]]Caused by: org.elasticsearch.search.query.QueryPhaseExecutionException:[logstash-2015.03.27][0]: query[ConstantScore(*:*)],from[0],size[0]: Query Failed[Failed to execute global facets]        at org.elasticsearch.search.facet.FacetPhase.execute(FacetPhase.java:193)        at org.elasticsearch.search.query.QueryPhase.execute(QueryPhase.java:171)        atorg.elasticsearch.search.SearchService.executeQueryPhase(SearchService.java:275)        atorg.elasticsearch.search.action.SearchServiceTransportAction$SearchQueryTransportHandler.messageReceived(SearchServiceTransportAction.java:776)        atorg.elasticsearch.search.action.SearchServiceTransportAction$SearchQueryTransportHandler.messageReceived(SearchServiceTransportAction.java:767)        atorg.elasticsearch.transport.netty.MessageChannelHandler$RequestHandler.run(MessageChannelHandler.java:275)        atjava.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)        atjava.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)        at java.lang.Thread.run(Thread.java:722)Caused by: org.elasticsearch.ElasticsearchException:org.elasticsearch.common.breaker.CircuitBreakingException: [FIELDDATA] Data too large,data for [@timestamp] would be larger than limit of [41085134438/38.2gb]
 atorg.elasticsearch.index.fielddata.plain.AbstractIndexFieldData.load(AbstractIndexFieldData.java:80)        atorg.elasticsearch.search.facet.datehistogram.CountDateHistogramFacetExecutor$Collector.setNextReader(CountDateHistogramFacetExecutor.java:88)        atorg.elasticsearch.common.lucene.search.FilteredCollector.setNextReader(FilteredCollector.java:67)        atorg.apache.lucene.search.MultiCollector.setNextReader(MultiCollector.java:113)        at org.apache.lucene.search.IndexSearcher.search(IndexSearcher.java:612)        atorg.elasticsearch.search.internal.ContextIndexSearcher.search(ContextIndexSearcher.java:191)        at org.apache.lucene.search.IndexSearcher.search(IndexSearcher.java:309)        at org.elasticsearch.search.facet.FacetPhase.execute(FacetPhase.java:186)        ... 8 moreCaused by: org.elasticsearch.common.util.concurrent.UncheckedExecutionException:org.elasticsearch.common.breaker.CircuitBreakingException: [FIELDDATA] Data too large,data for [@timestamp] would be larger than limit of [41085134438/38.2gb]        at org.elasticsearch.common.cache.LocalCache$Segment.get(LocalCache.java:2203)        at org.elasticsearch.common.cache.LocalCache.get(LocalCache.java:3937)        atorg.elasticsearch.common.cache.LocalCache$LocalManualCache.get(LocalCache.java:4739)        atorg.elasticsearch.indices.fielddata.cache.IndicesFieldDataCache$IndexFieldCache.load(IndicesFieldDataCache.java:174)
```

# Doc_value
Also, another solution to eliminate field data cache issue/constraints due to JVM heap size limitation is to leverage lucene `doc_value`.

For faceting/sorting/grouping Lucene needs to iterate over every document to collect the field values. Traditionally, this is achieved by uninverting the term index. This performs very well actually, since the field values are already grouped (by nature of the index), but it is relatively slow to load and is maintained in memory. DocValues aim to alleviate both of these problems while keeping performance comparable.

Enable doc value for given field as below
```
PUT /c3_metrics/_mapping/vm_availability
{
  "properties" : {
    "vpc": {
      "type":       "string",
      "index" :     "not_analyzed",
      "doc_values": true 
    }
  }
}
```
 
For DocValue performance & disk size consumption, please check [func-with-docvalues](http://lucidworks.com/blog/2013/04/02/fun-with-docvalues-in-solr-4-2/)

For initial Commit in Lucene, please check [Column-stride fields (aka per-document Payloads)](https://issues.apache.org/jira/browse/LUCENE-1231)

# Index Sharding Routing & Recovery Settings
When one data node is get out of ES cluster, ES cluster will trigger process to recover index shards from replica node.To minimize index recovery consumption time, we can enable concurrent mode as below.

Our settings to enable recovery with concurrent mode & traffic throttling as below:

```_cluster/settingstransient: {    indices: {        recovery: {            concurrent_streams: "10",            max_bytes_per_sec: "100mb",            concurrent_small_file_streams: "10"} },    cluster: {        routing: {
	        allocation: {			    enable: "all",    			node_initial_primaries_recoveries: "20",    			node_concurrent_recoveries: "4",			}
} }}
```

Given the limited space available, for detailed settings meaning, please check [shard_allocation_settings](https://www.elastic.co/guide/en/elasticsearch/reference/2.1/shards-allocation.html#_shard_allocation_settings)

# To be dynamic or not?
Highlight this crucial topic as much as we can. It's the biggest issue we encounter during daily work.

Elasticsearch's default mappings allow generic mapping definitions to be automatically applied to types that do not have mappings predefined.It's nice feature to allow us to create schema-less documents at will. And elasticsearch automatically detect target field type and do further analyzation & indexing.However, it will also involve much cpu & memory consumption for whole ES cluster.## Case Study for ES Cluster instability Issue due to Dynamic field Mapping
Below is the case which due to unsuitable dynamic field mapping.Phase A: Eachtime new index document comes in, when detect new field, then do analyzation & index in memory until reach to refresh duration, then flush to disk. Also, trigger update mapping to master coordinator node to merge mapping & flush cluster state to each node.
Below is our template definition for dynamic mapping.
**worst case analysis: assuming each index documents's filed value is fully different and each field value contains 3 special characters (",-,+, WHITE_SPACE) at average.**

|  # of newly inserted fields      | # of index documents per day           |  # of tokens for each field value separation  |
| number of cardinality |
| ------------- |:-------------:| :-------------:| -----:|
| Dedicated Master Node | node.master = true; node.data=false | Only participant to work as coordinator node & won't store index repo locally|
| Dedicated Data Node      | 

| # of newly inserted fields        | # of index documents per day           | # of tokens for each field value separation  |# of field cardinality  |
| ------------- |:-------------:| -----:|-----:|
| 100 | 681,035 | 3 (due to index type: analyzed)| 204,310,500      | 

Assume we enable dynamic mapping as below, thne periodically push index documents in.
```
GET /_template/vm_detailsmappings: { _default_: {dynamic_templates: [    {     string_fields: {      mapping: {       index: "analyzed",       omit_norms: true,       type: "string",       fields: {        raw: {         index: "not_analyzed",         ignore_above: 256,         type: "string"} }       },       match_mapping_type: "string",       match: "*"} }}
```

* Too much update-mapping tasks are running on master coordinator node to refresh cluster states.

```
GET /_cat/pending_tasks77  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [1]199  1.2h IMMEDIATEzen-disco-node_failed([slc5b01c-4259-elasticsearch][O51sjztZTHCSr2MG5T3PKQ][slc5b01c-4259.stratus.slc.ebay.com][inet[/10.120.79.97:9300]]{master=false}), reason failed toping, tried [3] times, each with maximum [30s] timeout 79  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [3] 78  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [2] 81  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [5] 82  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [1] 83  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [6] 80  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [4] 85  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [3] 86  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [7] 87  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [4] 88  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [8]89  1.3h HIGH      update-mapping [vm_details][vm_details] / nodesource
[O51sjztZTHCSr2MG5T3PKQ], order [5] 90  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [9] 91  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [6] 84  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [2] 93  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [7] 94  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [11] 95  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [8] 96  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [12] 97  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [9] 98  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [13] 99  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [14]100  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [10]101  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [15]102  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [11]103  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [16]104  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [12]105  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [17]106  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [13] 92  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [10]108  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [14]109  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [19]110  1.3h HIGH      update-mapping [vm_details][vm_details] / node[O51sjztZTHCSr2MG5T3PKQ], order [15]111  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [20]112  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [21]
....```