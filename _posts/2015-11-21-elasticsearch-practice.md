---
layout: post
title: "ElasticSearch Practice"
description: "ElasticSearch Understanding & Tuning"
category: 
tags: ["es","tuning"]
---
{% include JB/setup %}
### **Introduction**
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

### **Concept Highlights**
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


### **Perf Tuning Tips**
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

## *Field Data Cache*
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

## *Doc_value*
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

## *Index Sharding Routing & Recovery Settings*
When one data node is get out of ES cluster, ES cluster will trigger process to recover index shards from replica node.To minimize index recovery consumption time, we can enable concurrent mode as below.

Our settings to enable recovery with concurrent mode & traffic throttling as below:

```GET _cluster/settingstransient: {    indices: {        recovery: {            concurrent_streams: "10",            max_bytes_per_sec: "100mb",            concurrent_small_file_streams: "10"} },    cluster: {        routing: {
	        allocation: {			    enable: "all",    			node_initial_primaries_recoveries: "20",    			node_concurrent_recoveries: "4",			}
} }}
```

Given the limited space available, for detailed settings meaning, please check [shard_allocation_settings](https://www.elastic.co/guide/en/elasticsearch/reference/2.1/shards-allocation.html#_shard_allocation_settings)

## *To be dynamic or not?*
Highlight this crucial topic as much as we can. It's the biggest issue we encounter during daily work.

Elasticsearch's default mappings allow generic mapping definitions to be automatically applied to types that do not have mappings predefined.It's nice feature to allow us to create schema-less documents at will. And elasticsearch automatically detect target field type and do further analyzation & indexing.However, it will also involve much cpu & memory consumption for whole ES cluster.# Case Study for ES Cluster instability Issue due to Dynamic field Mapping
Below is the case which due to unsuitable dynamic field mapping.Phase A: Eachtime new index document comes in, when detect new field, then do analyzation & index in memory until reach to refresh duration, then flush to disk. Also, trigger update mapping to master coordinator node to merge mapping & flush cluster state to each node.
Below is our template definition for dynamic mapping.

| # of newly inserted fields        | # of index documents per day           | # of tokens for each field value separation  |# of field cardinality  |
| ------------- |:-------------:| -----:|-----:|
| 100 | 681,035 | 3 (due to index type: analyzed)| 204,310,500      | 

> **worst case analysis: assuming each index documents's filed value is fully different and each field value contains 3 special characters (",-,+, WHITE_SPACE) at average.**

Assume we enable dynamic mapping as below

```
GET /_template/vm_detailsmappings: { _default_: {dynamic_templates: [    {     string_fields: {      mapping: {       index: "analyzed",       omit_norms: true,       type: "string",       fields: {        raw: {         index: "not_analyzed",         ignore_above: 256,         type: "string"} }       },       match_mapping_type: "string",       match: "*"} }}
```

- **Phase A** periodically push index documents in.

* Too much update-mapping tasks are running on master coordinator node to refresh cluster states.

```
GET /_cat/pending_tasks 79  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [3] 78  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [2] 81  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [5]... ...
... ...
173  1.3h HIGH      update-mapping [vm_details][vm_details] / node```

* When checking target index mapping, bunch of dynamic field mapping are created as below.

```
GET /vm_details/_mappingExpand{"vm_details":{"mappings":{"_default_":{"dynamic_templates":[{"string_fields":{"mapping":{"index":"analyzed","omit_norms":true,"type":"string","fields":{"raw":{"index":"not_analyzed","ignore_above":256,"type":"string"}}},"match":"*","match_mapping_type":"string"}}],"date_detection":false,"_all":{"enabled":true},"properties":{"availability_status":{"type":"string","index":"not_analyzed"},"availability_value":{"type":"integer"},"az":{"type":"string","index":"not_analyzed"},"cell":{"type":"string","index":"not_analyzed"},"cell_name":{"type":"string","index":"not_analyzed"},"check_at":{"type":"date","format":"yyyy-MM-dd'T'HH:mm:ssZ"},"city":{"type":"string","index":"not_analyzed"},"cms_account":{"type":"string","index":"not_analyzed"},"cms_environment":{"type":"string","index":"not_analyzed"},"cms_organization":{"type":"string","index":"not_analyzed"},"cos":{"type":"string","index":"not_analyzed"},"country_alpha2":{"type":"string","index":"not_analyzed"},"country_name":{"type":"string","index":"not_analyzed"},"diagnose_consumption_seconds":{"type":"float"},"display_name":{"type":"string","index":"not_analyzed"},"first_failure_at":{"type":"date","format":"yyyy-MM-dd'T'HH:mm:ssZ"},"flavor_name":{"type":"string","index":"not_analyzed"},"host":{"type":"string","index":"not_analyzed"},"hostname":{"type":"string","index":"not_analyzed"},"hv":{"type":"string","index":"not_analyzed"},"id":{"type":"string","index":"not_analyzed"},"image_id":{"type":"string","index":"not_analyzed"},"image_kernel_id":{"type":"string","index":"not_analyzed"},"image_name":{"type":"string","index":"not_analyzed"},"image_ramdisk_id":{"type":"string","index":"not_analyzed"},"image_ref":{"type":"string","index":"not_analyzed"},"instance_user_id":{"type":"string","index":"not_analyzed"},"ip_address":{"type":"string","index":"not_analyzed"},"ips":{"type":"string","index":"not_analyzed"},"kernel_id":{"type":"string","index":"not_analyzed"},"key_data":{"type":"string","index":"not_analyzed"},"key_name":{"type":"string","index":"not_analyzed"},"last_diagnose_at":{"type":"date","format":"yyyy-MM-dd'T'HH:mm:ssZ"},"last_failure_at":{"type":"date","format":"yyyy-MM-dd'T'HH:mm:ssZ"},"launched_on":{"type":"string","index":"not_analyzed"},"ldap_company":{"type":"string","index":"not_analyzed"},"ldap_country":{"type":"string","index":"not_analyzed"},"ldap_cube":{"type":"string","index":"not_analyzed"},"ldap_department":{"type":"string","index":"not_analyzed"},"ldap_l4_manager":{"type":"string","index":"not_analyzed"},"ldap_location":{"type":"string","index":"not_analyzed"},"ldap_mail":{"type":"string","index":"not_analyzed"},"ldap_phone":{"type":"string","index":"not_analyzed"},"mpt.environment":{"type":"string","index":"not_analyzed"},"networks":{"type":"string","index":"not_analyzed"},"node":{"type":"string","index":"not_analyzed"},"project_cms_account":{"type":"string","index":"not_analyzed"},"project_cms_environment":{"type":"string","index":"not_analyzed"},"project_cms_organization":{"type":"string","index":"not_analyzed"},"project_computed":{"type":"string","index":"not_analyzed"},"project_cos":{"type":"string","index":"not_analyzed"},"project_id":{"type":"string","index":"not_analyzed"},"project_mpt.environment":{"type":"string","index":"not_analyzed"},"project_name":{"type":"string","index":"not_analyzed"},"project_vpc":{"type":"string","index":"not_analyzed"},"ramdisk_id":{"type":"string","index":"not_analyzed"},"reason":{"type":"string","index":"not_analyzed"},"reservation_id":{"type":"string","index":"not_analyzed"},"root_device_name":{"type":"string","index":"not_analyzed"},"state_abbr":{"type":"string","index":"not_analyzed"},"sys_arch":{"type":"string","index":"not_analyzed"},"user_cms_account":{"type":"string","index":"not_analyzed"},"user_cms_organization":{"type":"string","index":"not_analyzed"},"user_dedicated_team_cms_account":{"type":"string","index":"not_analyzed"},"user_dedicated_team_cms_organization":{"type":"string","index":"not_analyzed"},"user_domain_cms_account":{"type":"string","index":"not_analyzed"},"user_domain_cms_organization":{"type":"string","index":"not_analyzed"},"
... ...
... ...
```

- **Phase B**: It involves too much memory occupation, then trigger long-time (more than 3 minutes) JVM GC pause time (even enabled with CMS GC policy).

```
[2015-05-26 20:14:07,091][WARN ][monitor.jvm              ][slc5b01c-6ofi-elasticsearch] [gc][young][39135][23857] duration [3.2s], collections[2]/[4.4m], total [3.2s]/[54.7m],memory [73.9gb]->[50gb]/[95.8gb], all_pools {[young][329.7mb]->[5.5mb]/[1.4gb]}{[survivor] [65.4mb]->[0b]/[191.3mb]}{[old][73.5gb]->[50gb]/[94.1gb]}[2015-05-26 20:14:07,091][WARN ][monitor.jvm              ][slc5b01c-6ofi-elasticsearch] [gc][old][39135][8] duration [4.3m], collections[2]/[4.4m], total [4.3m]/[5.5m], memory[73.9gb]->[50gb]/[95.8gb], all_pools {[young] [329.7mb]->[5.5mb]/[1.4gb]}{[survivor][65.4mb]->[0b]/[191.3mb]}{[old] [73.5gb]->[50gb]/[94.1gb]}[2015-05-26 20:48:15,962][WARN ][monitor.jvm              ][slc5b01c-6ofi-elasticsearch] [gc][young][41161][24715] duration [6.8s], collections[1]/[7.2s], total [6.8s]/[57m], meemory [67.3gb]->[67.4gb]/[95.8gb], all_pools {[young][1gb]->[13.5mb]/[1.4gb]}{[survivor] [142.5mb]->[191.3mb]/[191.3mb]}{[old][66.1gb]->[67.2gb]/[94.1gb]}[2015-05-26 20:48:21,142][WARN ][monitor.jvm              ][slc5b01c-6ofi-elasticsearch] [gc][young][41163][24716] duration [3.2s], collections[1]/[4.1s], total [3.2s]/[57m], meemory [68.4gb]->[68.9gb]/[95.8gb], all_pools {[young][1gb]->[19.5mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][67.2gb]->[68.7gb]/[94.1gb]}[2015-05-26 20:48:27,182][WARN ][monitor.jvm              ][slc5b01c-6ofi-elasticsearch] [gc][young][41167][24717] duration [2.4s], collections[1]/[3s], total [2.4s]/[57.1m], meemory [70.1gb]->[70.1gb]/[95.8gb], all_pools {[young][1.2gb]->[23.5mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][68.7gb]->[69.9gb]/[94.1gb]}[2015-05-26 20:48:31,835][WARN ][monitor.jvm              ][slc5b01c-6ofi-elasticsearch] [gc][young][41170][24718] duration [2.1s], collections[1]/[2.3s], total [2.1s]/[57.1m],memory [71.4gb]->[71gb]/[95.8gb], all_pools {[young][1.3gb]->[26.1mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][69.9gb]->[70.8gb]/[94.1gb]}[2015-05-26 20:48:35,594][WARN ][monitor.jvm              ][slc5b01c-6ofi-elasticsearch] [gc][young][41172][24719] duration [2.1s], collections[1]/[2.7s], total [2.1s]/[57.2m],memory [71.9gb]->[71.9gb]/[95.8gb], all_pools {[young][940.7mb]->[27.8mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][70.8gb]->[71.6gb]/[94.1gb]}[2015-05-26 20:48:39,354][WARN ][monitor.jvm              ][slc5b01c-6ofi-elasticsearch] [gc][young][41175][24720] duration [1.1s], collections[1]/[1.7s], total [1.1s]/[57.2m],memory [72.9gb]->[72.4gb]/[95.8gb], all_pools {[young][1gb]->[28.5mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][71.6gb]->[72.1gb]/[94.1gb]}
```

- **Phase C**: It leads to cluster node failed to report its status to master coordinator node within acceptable time (3*30s by default).

```
[2015-05-26 20:50:41,402][INFO ][discovery.zen            ][slc5b01c-6ofi-elasticsearch] master_left[[slc5b01c-5qth-elasticsearch][R0Bb9W6mQCyoikmccAgsyQ][slc5b01c-5qth.stratus..slc.ebay.com][inet[/10.120.124.43:9300]]{data=false, master=true}], reason [failed to ping,tried [3] times, each with  maximum [30s] timeout]
```

- **Phase D**: Eventually master coordinator node notice whole cluster topology has been changed. Due to previous data node didn't report heartbeat to coordinator node for long time, coordinator regards the data node is dead, then trigger data shards recovery job, go to vici ous cycle.

```
GET /_cat/shardsvm_details 2 p INITIALIZING 134292 340.3mb 10.120.107.7  slc5b01c-6ofi-elasticsearchvm_details 2 r UNASSIGNED 134292 350.3mb 10.120.104.90 slc5b01c-5cf9-elasticsearchvm_details 0 p INITIALIZING 134959   342mb 10.120.79.97  slc5b01c-4259-elasticsearchvm_details 0 r UNASSIGNED 134959 358.4mb 10.120.107.7  slc5b01c-6ofi-elasticsearchvm_details 3 p INITIALIZING 134710 358.2mb 10.120.79.97  slc5b01c-4259-elasticsearchvm_details 3 r UNASSIGNED 134710 339.4mb 10.120.107.7  slc5b01c-6ofi-elasticsearchvm_details 1 r INITIALIZING 134259 337.8mb 10.120.79.97  slc5b01c-4259-elasticsearchvm_details 1 p UNASSIGNED 134259 348.3mb 10.120.104.90 slc5b01c-5cf9-elasticsearchvm_details 4 r INITIALIZING 135060   339mb 10.120.79.97  slc5b01c-4259-elasticsearchvm_details 4 p UNASSIGNED 135060   354mb 10.120.104.90 slc5b01c-5cf9-elasticsearch
```