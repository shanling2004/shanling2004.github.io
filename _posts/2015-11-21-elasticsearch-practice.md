---
layout: post
title: "ElasticSearch Practice"
description: "ElasticSearch Understanding & Performance Tuning & Operations"
category: 
tags: ["elasticsearch"]
---
{% include JB/setup %}
### **Introduction**
We have tried to setup ELK(elasticsearch-logstash-kibana) stack as parrt of our cloud monitor eco-system.

In following page, we call elasticsearch (abbreviation: ES).

## Use Case
In our cloud eco-system, we setup ELK tech stack to collect openstack control plane log and search for troubleshooting.
Also, we routine publish VM availability metrics (with multiple fields [E.g. vpc, image name, VM_uuid, cos]) into ES and work as time-series major portal for KPI evaluation.

We'd like to share some performance tuning & operation experience for elasticsearch. also for some cases, we try to understand more inertanls from source code perspective.

Below is our ELK version.

    Elasticsearch version: 1.5.3 
    Logstash version: 1.0.9
    Kibana version: 4.0.3
 Target OS Version: Ubuntu 14.04

### **Concept Highlights**
## *Understand ES Toplogy*
ES node has 3 role responsibilities as below. They're not exclusive roles. One instance node can take one or more roles (E.g. both matser & data node).

| Role        | settings           | Responsibility  |
| ------------- |:-------------:| :---------------:|
| Dedicated Master Node | node.master = true; node.data=false | Only participant to work as coordinator node & won't store index repo locally|
| Dedicated Data Node      | node.master = false; node.data=true | store index repo locally, won't work as coordinator node|
| client node | node.master = true; node.data=false; http.enabled=false     |    load balance node |

* Only when the nodes is enabled with master attribute, then it's eligible to do leader election

* For Details: Please check [Elasticsearch Node](https://www.elastic.co/guide/en/elasticsearch/reference/master/modules-node.html)

Here's our ES Topo with 3 matser nodes & 3 data nodes (2TB disk space).

This architect is more reliable & isolate data & cluster control impact when each node is down. 

Also, master & data node have different hardware SKU requirement. 

Master node takes responsibility to collect & dispatch cluster info & heartbeat from each non-client node. It does not need large volume disk, but more powerful CPU Cores & enough memory.

Data node need large volume disk for index repository storage and enough memory for index caching purpose.

![es-topo]({{ site.JB.IMAGE_PATH }}/es_topo.png "ES Topology Diagram")


### **Perf Tuning & Operation Tips**
## *Avoid Swap Out Memory*
When set `bootstrap.mlockall=true`, elasticsearch try to lock the process address space into RAM, preventing any Elasticsearch memory frombeing swapped out.
> About bootstrap.mlockall option, Please check code for org.elasticsearch.bootstrap.Bootstrap

```java

private void setup(boolean addShutdownHook, Tuple<Settings, Environment>  tuple) throws Exception {
        if (tuple.v1().getAsBoolean("bootstrap.mlockall", false)) {
                    Natives.tryMlockall();
}
```

> org.elasticsearch.common.jna.CLibrary

```java￼public static native int mlockall(int flags);
```
* **Linux OS Support for mlockall**
However, only enable the above option is not enough, it needs support from OS layer. Otherwise, we will encounter the below issue.

```
2015-05-24 17:44:49,247][WARN ][monitor.jvm              ][XXX5b01c-6gjg-elasticsearch] [gc][old][75551][4260] duration [39.9s], collections[1]/[40.3s], total [39.9s]/[1.7d], memory [62.8gb]->[62.8gb]/[63.8gb], all_pools{[young] [769.1mb]->[769.2mb]/[1.4gb]}{[survivor] [0b]->[0b]/[191.3mb]}{[old][62.1gb]->[62.1gb]/[62.1gb]}[2015-05-24 17:48:19,869][WARN ][common.jna               ] Unable to lock JVM memory(ENOMEM). This can result in part of the JVM being swapped out. IncreaseRLIMIT_MEMLOCK (ulimit).[2015-05-24 17:48:41,028][WARN ][monitor.jvm              ][XXX5b01c-6gjg-elasticsearch] [gc][young][4][3] duration [1.5s], collections[1]/[2.5s], total [1.5s]/[12.6s], memory [2.4gb]->[2.4gb]/[63.8gb], all_pools {[young][735.6mb]->[22.3mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][1.5gb]->[2.2gb]/[62.1gb]}[2015-05-24 17:49:20,911][WARN ][monitor.jvm              ][XXX5b01c-6gjg-elasticsearch] [gc][young][42][4] duration [2.7s], collections[1]/[2.8s], total [2.7s]/[15.3s], memory [3.8gb]->[3.6gb]/[63.8gb], all_pools {[young][1.4gb]->[20.8mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][2.2gb]->[3.4gb]/[62.1gb]}
```
* **Solution**

> 1. in "/usr/lib/systemd/system/elasticsearch.service", let LimitMEMLOCK=infinity
> 2. ulimit -l unlimited
> 3. service elasticsearch-elasticsearch restart

* **Sanity Check**

You can check process.mlockall is true or not via GET request to `/_nodes/process` .

![mlockall check]({{ site.JB.IMAGE_PATH }}/mlockall.png "Process Url for mlockall check")

## *Field Data Cache*
* **Background**

For Lucene's inverted index, it's like one concordances for whole book from key term mapping to whole content.
It's quite efficient & useful solution for search request, but not friendly for key term aggregation.

ES field data cache is used mainly when sorting on or faceting on a field. It loads all the field values to memory in order to provide fast document based access to those values. The field data cache can be expensive to build for a field, so its recommended to have enough memory to allocate it, and to keep it loaded.

You need to control field data cache size in caution to avoid it become as major memory killer.


* **Field Data Cache Settings**
Be caution of field cache size, it's one of major memory killer for elasticsearcg during search phase.

Our field cache settings as below:

```
POST /_cluster/settings -d
indices:  fielddata:    cache:
    	size: 40%
```

Otherwise, you might encounter the below issue "[FIELDDATA] Data too large" when lots of aggregation & faceting request to ES cluster.

```
Caused by: org.elasticsearch.common.breaker.CircuitBreakingException: [FIELDDATA] Datatoo large, data for [@timestamp] would be larger than limit of [41085134438/38.2gb]atorg.elasticsearch.common.breaker.ChildMemoryCircuitBreaker.circuitBreak(ChildMemoryCircuitBreaker.java:97)        atorg.elasticsearch.common.breaker.ChildMemoryCircuitBreaker.addEstimateBytesAndMaybeBreak(ChildMemoryCircuitBreaker.java:148)        atorg.elasticsearch.index.fielddata.RamAccountingTermsEnum.flush(RamAccountingTermsEnum.java:71)        atorg.elasticsearch.index.fielddata.RamAccountingTermsEnum.next(RamAccountingTermsEnum.java:85)
       atorg.elasticsearch.index.fielddata.ordinals.OrdinalsBuilder$3.next(OrdinalsBuilder.java:472)        atorg.elasticsearch.index.fielddata.plain.PackedArrayIndexFieldData.loadDirect(PackedArrayIndexFieldData.java:109)        ... 19 moreCaused by: org.elasticsearch.common.util.concurrent.UncheckedExecutionException:org.elasticsearch.common.breaker.CircuitBreakingException: [FIELDDATA] Data too large,data for [@timestamp] would be larger than limit of [41085134438/38.2gb]        at org.elasticsearch.common.cache.LocalCache$Segment.get(LocalCache.java:2203)        at org.elasticsearch.common.cache.LocalCache.get(LocalCache.java:3937)        atorg.elasticsearch.common.cache.LocalCache$LocalManualCache.get(LocalCache.java:4739)        atorg.elasticsearch.indices.fielddata.cache.IndicesFieldDataCache$IndexFieldCache.load(IndicesFieldDataCache.java:174)
```

## *Doc_values*
Also, another possible solution to eliminate field data cache issue/constraints due to JVM heap size limitation is to leverage lucene `doc_value`.

For faceting/sorting/grouping Lucene needs to iterate over every document to collect the field values. Traditionally, this is achieved by uninverting the term index. This performs very well actually, since the field values are already grouped (by nature of the index), but it is relatively slow to load and is maintained in memory. DocValues aim to alleviate both of these problems while keeping performance comparable.

Enable doc_values for given field as below

```json
PUT /c3_metrics/_mapping/vm_availability -d

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
When one data node left out of ES cluster, ES cluster will trigger process to recover index shards from replica node. To minimize index recovery consumption time & impact, we can enable concurrent mode & trigger criteria as below.

Our settings to enable recovery with concurrent mode & traffic throttling as below:

```jsonGET _cluster/settings
transient: {    indices: {        recovery: {            concurrent_streams: "10",            max_bytes_per_sec: "100mb",            concurrent_small_file_streams: "10"} },    cluster: {        routing: {
	        allocation: {			    enable: "all",    			node_initial_primaries_recoveries: "20",    			node_concurrent_recoveries: "4",			}
} }}
```

Given the limited space available, for detailed settings meaning, please check [shard_allocation_settings](https://www.elastic.co/guide/en/elasticsearch/reference/2.1/shards-allocation.html#_shard_allocation_settings)

### **To be dynamic or not?***
**Highlight this crucial topic as much as we can**, it's the biggest issue we encounter during daily support work.

Elasticsearch's default mappings allow generic mapping definitions to be automatically applied to types that do not have mappings predefined.It's nice feature to allow us to create schema-less documents at will. And elasticsearch automatically detect target field type and do further analyzation & indexing.
However, it will also involve much cpu & memory consumption for whole ES cluster.
## Case Study for ES Cluster instability Issue due to Dynamic field Mapping
Below is the case which due to unsuitable dynamic field mapping.
- **Phase A**

Eachtime new index document comes in, when ES detect new field, then do analyzation & index in memory until reach to refresh duration, then flush to disk. Also, trigger update mapping to master coordinator node to merge mapping & flush cluster state to each node.
Below is our template definition for dynamic mapping.

|  newly inserted fields number    | index documents count per day           |  token count for each field value separation  | field cardinality count  |
| :----: |:-------------:| :-----------:|-----|
| 100 | 681,035 | 3 (due to index type: analyzed)| 204,310,500      | 

> Worst case analysis: assuming each index documents's filed value is fully different and each field value contains 3 special characters (",-,+, WHITE_SPACE) at average.


Assume we enable dynamic mapping as below

```
GET /_template/vm_details
mappings: { _default_: {dynamic_templates: [    {     string_fields: {      mapping: {       index: "analyzed",       omit_norms: true,       type: "string",       fields: {        raw: {         index: "not_analyzed",         ignore_above: 256,         type: "string"} }       },       match_mapping_type: "string",       match: "*"} }}
```

- **Phase B** 

Too much update-mapping tasks are running on master coordinator node to refresh cluster states.

```
GET /_cat/pending_tasks 79  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [3] 78  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [2] 81  1.3h HIGH      update-mapping [vm_details][vm_details] / node[crM-bvkmTHaQRybGhETtsg], order [5]... ...
... ...
173  1.3h HIGH      update-mapping [vm_details][vm_details] / node```

* When checking target index mapping, bunch of dynamic field mapping are created as below.

```
GET /vm_details/_mapping
{"vm_details":{"mappings":{"_default_":{"dynamic_templates":[{"string_fields":{"mapping":{"index":"analyzed","omit_norms":true,"type":"string","fields":{"raw":{"index":"not_analyzed","ignore_above":256,"type":"string"}}},"match":"*","match_mapping_type":"string"}}],"date_detection":false,"_all":{"enabled":true},"properties":{"availability_status":{"type":"string","index":"not_analyzed"},"availability_value":{"type":"integer"},"az":{"type":"string","index":"not_analyzed"},"cell":{"type":"string","index":"not_analyzed"},"cell_name":{"type":"string","index":"not_analyzed"}
... ...
... ...
```

- **Phase C**

It involves too much memory occupation for both data node (for reindex) & master node (for sync cluster status), then trigger long-time (more than 3 minutes) JVM GC pause time (even enabled with CMS GC policy).

```
[2015-05-26 20:14:07,091][WARN ][monitor.jvm              ][XXX5b01c-6ofi-elasticsearch] [gc][young][39135][23857] duration [3.2s], collections[2]/[4.4m], total [3.2s]/[54.7m],memory [73.9gb]->[50gb]/[95.8gb], all_pools {[young][329.7mb]->[5.5mb]/[1.4gb]}{[survivor] [65.4mb]->[0b]/[191.3mb]}{[old][73.5gb]->[50gb]/[94.1gb]}[2015-05-26 20:14:07,091][WARN ][monitor.jvm              ][XXX5b01c-6ofi-elasticsearch] [gc][old][39135][8] duration [4.3m], collections[2]/[4.4m], total [4.3m]/[5.5m], memory[73.9gb]->[50gb]/[95.8gb], all_pools {[young] [329.7mb]->[5.5mb]/[1.4gb]}{[survivor][65.4mb]->[0b]/[191.3mb]}{[old] [73.5gb]->[50gb]/[94.1gb]}[2015-05-26 20:48:15,962][WARN ][monitor.jvm              ][XXX5b01c-6ofi-elasticsearch] [gc][young][41161][24715] duration [6.8s], collections[1]/[7.2s], total [6.8s]/[57m], meemory [67.3gb]->[67.4gb]/[95.8gb], all_pools {[young][1gb]->[13.5mb]/[1.4gb]}{[survivor] [142.5mb]->[191.3mb]/[191.3mb]}{[old][66.1gb]->[67.2gb]/[94.1gb]}[2015-05-26 20:48:21,142][WARN ][monitor.jvm              ][XXX5b01c-6ofi-elasticsearch] [gc][young][41163][24716] duration [3.2s], collections[1]/[4.1s], total [3.2s]/[57m], meemory [68.4gb]->[68.9gb]/[95.8gb], all_pools {[young][1gb]->[19.5mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][67.2gb]->[68.7gb]/[94.1gb]}[2015-05-26 20:48:27,182][WARN ][monitor.jvm              ][XXX5b01c-6ofi-elasticsearch] [gc][young][41167][24717] duration [2.4s], collections[1]/[3s], total [2.4s]/[57.1m], meemory [70.1gb]->[70.1gb]/[95.8gb], all_pools {[young][1.2gb]->[23.5mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][68.7gb]->[69.9gb]/[94.1gb]}[2015-05-26 20:48:31,835][WARN ][monitor.jvm              ][XXX5b01c-6ofi-elasticsearch] [gc][young][41170][24718] duration [2.1s], collections[1]/[2.3s], total [2.1s]/[57.1m],memory [71.4gb]->[71gb]/[95.8gb], all_pools {[young][1.3gb]->[26.1mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][69.9gb]->[70.8gb]/[94.1gb]}[2015-05-26 20:48:35,594][WARN ][monitor.jvm              ][XXX5b01c-6ofi-elasticsearch] [gc][young][41172][24719] duration [2.1s], collections[1]/[2.7s], total [2.1s]/[57.2m],memory [71.9gb]->[71.9gb]/[95.8gb], all_pools {[young][940.7mb]->[27.8mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][70.8gb]->[71.6gb]/[94.1gb]}[2015-05-26 20:48:39,354][WARN ][monitor.jvm              ][XXX5b01c-6ofi-elasticsearch] [gc][young][41175][24720] duration [1.1s], collections[1]/[1.7s], total [1.1s]/[57.2m],memory [72.9gb]->[72.4gb]/[95.8gb], all_pools {[young][1gb]->[28.5mb]/[1.4gb]}{[survivor] [191.3mb]->[191.3mb]/[191.3mb]}{[old][71.6gb]->[72.1gb]/[94.1gb]}
```

- **Phase D**

It leads to cluster node failed to report its status to master coordinator node within acceptable time (3*30s by default).

```
[2015-05-26 20:50:41,402][INFO ][discovery.zen            ][XXX5b01c-6ofi-elasticsearch] master_left[[XXX5b01c-5qth-elasticsearch][R0Bb9W6mQCyoikmccAgsyQ][XXX5b01c-5qth][inet[/XX.XX.XX.43:9300]]{data=false, master=true}], reason [failed to ping,tried [3] times, each with  maximum [30s] timeout]
```

- **Phase E**

Eventually master coordinator node notice whole cluster topology has been changed. Due to previous data node didn't report heartbeat to coordinator node for long time, coordinator regards the data node is dead, then trigger data shards recovery job, go to vici ous cycle.

```
GET /_cat/shards
vm_details 2 p INITIALIZING 134292 340.3mb XX.XX.XX.7  XXX5b01c-6ofi-elasticsearchvm_details 2 r UNASSIGNED 134292 350.3mb XX.XX.XX.90 XXX5b01c-5cf9-elasticsearchvm_details 0 p INITIALIZING 134959   342mb XX.XX.XX.97  XXX5b01c-4259-elasticsearchvm_details 0 r UNASSIGNED 134959 358.4mb XX.XX.XX.7  XXX5b01c-6ofi-elasticsearchvm_details 3 p INITIALIZING 134710 358.2mb XX.XX.XX.97  XXX5b01c-4259-elasticsearchvm_details 3 r UNASSIGNED 134710 339.4mb XX.XX.XX.7  XXX5b01c-6ofi-elasticsearchvm_details 1 r INITIALIZING 134259 337.8mb XX.XX.XX.97  XXX5b01c-4259-elasticsearchvm_details 1 p UNASSIGNED 134259 348.3mb XX.XX.XX.90 XXX5b01c-5cf9-elasticsearchvm_details 4 r INITIALIZING 135060   339mb XX.XX.XX.97  XXX5b01c-4259-elasticsearchvm_details 4 p UNASSIGNED 135060   354mb XX.XX.XX.90 XXX5b01c-5cf9-elasticsearch
```

# **Conclusion & Suggestion for dynamic field mapping**
When bundch of new field inserted, dynamic field mapping occupy too much resources (CPU/memory) to build lucene index repo especially when emerge lots of unexpected fields. Also, thhe situation will continue to worsen when enabling with analyzed index which will spawn more tokens/terms.

Even for the worst case, the above time-consuming index process trigger long GC(pause) time which force data nodes failed to finish heartbeat communication within acceptable intervals. Then it leads master node regards given data node is leave the whole cluster and trigger index recovery process. For our case, we're fully familiar with expected fields subset which are used for search/aggregation/primary keys. So we can explicitly disable dynamic field mapping and list all critical field mappings in ES template.

```
GET /_template/vm_details
mappings: { _default_: {  dynamic: false,  date_detection: false,  numeric_detection: false,  _ttl: {   enabled: true,   default: "60d"   },  properties: {   reason: {    index: "not_analyzed",    type: "string"   },   cos: {    index: "not_analyzed",    type: "string"}.... ...
```

## Graceful Decomm ES Data Node
Assume the scenario we detected one ES data node whose has some hardware issue,
we need to decomm it at once without any data loss as possible.

We can tell ES cluster to exclude it from allocation as below command.This will cause Elasticsearch to allocate the shards on that node to the remaining nodes, without the state of the cluster changing to yellow or red (even if you have replication 0).
Once all the shards have been reallocated you can shutdown the node and do whatever you need to do there. Once you're done, include the node for allocation and Elasticsearch will rebalance the shards again.

```
PUT _cluster/settings -d '{  "transient" :{      "cluster.routing.allocation.exclude._ip" : "XX.XX.XX.132"   }}';
```

Then we can confirm ES node topology as below:

```
GET _cat/nodes

XXX-6svl XX.XX.XX.132 29 28 1.56 d -XXX-6svl-elasticsearch
```

Then we can confirm ES node topo & sharding alloaction as well. During shard slice auto-rebalance period, we can see the relocating status in shards.

```
GET _cat/shards
logstash-2015.06.01  4 p RELOCATING 30518010 13.2gb XX.XX.XX.132XXX-6svl-elasticsearch -> YY.YY.YY.169 YYY-6f2b-elasticsearch
```

Also, [bigdesk plugin](https://github.com/lukas-vlcek/bigdesk) is useful visualization tool to demonstrate each shards distribution within whole ES node cluster as below.

![es-shard-distribution]({{ site.JB.IMAGE_PATH }}/es-shard-distribution.png "ES Shards Distribution Diagram")

## TimeZone
When we're using Kibana 4, one thing you won't skip is to set target timestamp field for one index repository. Hence, we need to define date format for target timestamp filed by aware of time zone.

We can take [mapping-date-format](https://www.elastic.co/guide/en/elasticsearch/reference/1.6/mapping-date-format.html) as reference. 

|  Date Format  | Timestamp Field Value           |  
| ------------- |:-------------| 
| check_at: {type: "date", format: "yyyy-MM-dd'T'HH:mm:ssZ" } | check_at: "2015-05-30T17:35:06-0700"| 
| @timestamp: {type: "date", format: "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'" } | timestamp: "2015-07-10T00:06:02.605Z"| 


### Analyzed or not_analyzed?
The big difference for index: "not_analyzed" or "analyzed" is the field value being broken down into token using an analyzer or not.

```
org.elasticsearch.index.mapper.core.TypeParsers

public static void parseIndex(String fieldName, String index,AbstractFieldMapper.Builder builder) throws MapperParsingException {        index = Strings.toUnderscoreCase(index);        if ("no".equals(index)) {            builder.index(false);        } else if ("not_analyzed".equals(index)) {            builder.index(true);            builder.tokenized(false);        } else if ("analyzed".equals(index)) {            builder.index(true);            builder.tokenized(true);        }
```

Internally for non_analyzed field, ES just set target fieldType to ignore norms & only keep field value only without any word segmentation (decide related word frequency & position & offset).

```java
org.elasticsearch.index.mapper.core.StringFieldMapper
@Override        public StringFieldMapper build(BuilderContext context) {            if (positionOffsetGap > 0) {                indexAnalyzer = new NamedAnalyzer(indexAnalyzer, positionOffsetGap);                searchAnalyzer = new NamedAnalyzer(searchAnalyzer, positionOffsetGap);                searchQuotedAnalyzer = new NamedAnalyzer(searchQuotedAnalyzer,positionOffsetGap);            }            // if the field is not analyzed, then by default, we should omit norms andhave docs onlyemits what// index options, as probably what the user really wants// if they are set explicitly, we will use those values// we also change the values on the default field type so that toXContent// differs from the defaultsFieldType defaultFieldType = new FieldType(Defaults.FIELD_TYPE);if (fieldType.indexed() && !fieldType.tokenized()) {         defaultFieldType.setOmitNorms(true);defaultFieldType.setIndexOptions(IndexOptions.DOCS_ONLY);         if (!omitNormsSet && boost == Defaults.BOOST) {             fieldType.setOmitNorms(true);         }         if (!indexOptionsSet) {             fieldType.setIndexOptions(IndexOptions.DOCS_ONLY);         }}```

## Tokenized Sample
For input field value "meta-data-kernel-id", ES will regard "-" as tokenizer and split to multiple terms as below:

```html 
GET /anaylze_test/_analyze?text=meta-data-kernel-id&analyzer=standard
{
tokens: [{token: "meta", start_offset: 0, end_offset: 4, type: "<ALPHANUM>", position: 1},{token: "data", start_offset: 5, end_offset: 9, type: "<ALPHANUM>", position: 2},{token: "kernel", start_offset: 10, end_offset: 16, type: "<ALPHANUM>", position: 3},{token: "id", start_offset: 17, end_offset: 19, type: "<ALPHANUM>", position: 4} ] }
```

## Analyzed field vs not_analyzed field
* **Analyzed field vs not_analyzed field**
Assume we have two fields (Field `new_uuid` with analyzed index type, Field `uuid` with not_analyzed index type)

```
GET /anaylze_test/_mapping (note index default value is "analyzed")

new_uuid: { type: "string"}, uuid: { type: "string", index: "not_analyzed"}
```

Assuming we have one index document as below

```
GET /anaylze_test/_search?q=id:3

{_index: "anaylze_test", _type: "anaylze_test", _id: "3", _score: 0.095891505, _source: {  new_uuid: "wrong-test-val-id",  uuid: "wrong-data-kernel-id" }}
```

* QueryString **without** specific field

|  Search Pattern  | Match Result for Target Field: new_uuid Field Value           |  Match Result for Target Field: uuid Field Value           |  
| ------------- |:-------------:| :-------------:| 
| substring | /analyze_test/_search?q=test-val-id<br/>/analyze_test/_search?q=_all:test-val-id<br/> (**Match**)|/analyze_test/_search?q=data-<br/>/analyze_test/_search?q=_all:data-<br/>(**Match**)|
| disorder substring| /analyze_test/_search?q=test-id<br/>/analyze_test/_search?q=_all:test-id<br/> (**Match**)|/analyze_test/_search?q=data-id<br/>/analyze_test/_search?q=_all:data-id<br/> (**Match**)|
| regular expression without "-" tokenizer| /analyze_test/_search?q=\*test\*<br/>/analyze_test/_search?q=_all:\*test\*<br/> (**Match**)|/analyze_test/_search?q=\*data\*<br/>/analyze_test/_search?q=_all:\*data\*<br/> (**Match**)|
| regular expression with "-" tokenizer & "\""| /analyze_test/_search?q="\*test-\*"<br/>/analyze_test/_search?q=_all:"\*test-\*"<br/> (**Match**)|/analyze_test/_search?q="\*data-\*"<br/>/analyze_test/_search?q=_all:"\*data-\*"<br/> (**Match**)|

# Analysis for query string with default field
For query_string Query, if no prefix field is specified by `index.query.default_field`, default query field for the  index settings is `_all`.

* **Source Code confirmation**

```java
org.elasticsearch.index.query.IndexQueryParserService
@Inject    public IndexQueryParserService(Index index, @IndexSettings Settings indexSettings,                                   IndicesQueriesRegistry indicesQueriesRegistry,                                   ScriptService scriptService, AnalysisService                                   MapperService mapperService, IndexCache indexCache,CacheRecycler cacheRecycler,analysisService,IndexFieldDataService fieldDataService,fixedBitSetFilterCache,namedQueryParsers,IndexEngine indexEngine, FixedBitSetFilterCache@Nullable SimilarityService similarityService,@Nullable Map<String, QueryParserFactory>@Nullable Map<String, FilterParserFactory>namedFilterParsers) {        super(index, indexSettings);  ... ....        this.defaultField = indexSettings.get(DEFAULT_FIELD, AllFieldMapper.NAME);
``````java
org.elasticsearch.index.mapper.internal.AllFieldMapper
public class AllFieldMapper extends AbstractFieldMapper<String> implementsInternalMapper, RootMapper { ... ...    public static final String NAME = "_all";
```

The _all field is meant to return the index content that come from all the fields that your documents are composed of. You can search on it but never return it, since it's indexed but not stored in lucene.When not starting & ending with double quotation marks, query string itself also will be tokenized based on the key words: `+ - = && || > < ! ( ) { } [ ] ^ " ~ * ? : \ /`.

* **Dig in details for Tokenizer**

For Index document tokenization, field 'new\_uuid' is analyzed field with standard analyzer. And "-" is special character for tokenization. 

Field "uuid" is not_analyzed field, so it should be kept for whole text without tokenization.

```
_all: {"new_uuid":[{"token":"wrong"},{"token":"test"},{"token": "val"},{"token":"id"}]  "uuid":[{"token":"wrong-data-kernel-id"}]}
```

Query String: `q=test-id`

```
q: [{"term":"test"},{"term":"id"}]
```
If all content (terms) match with the above query_string's terms, then result is `match`.

# Analysis for query string with given field

|  Search Pattern  | Match Result for Target Field: new_uuid Field Value           |  Match Result for Target Field: uuid Field Value           |  
| ------------- |:-------------:| :-------------:| 
| substring | /analyze_test/_search?q=new\_uuid:test-<br/> (**Match**)|/analyze_test/_search?q=uuid:data-<br/>(**Mismatch**)|
| disorder substring | /analyze_test/_search?q=new\_uuid:test-id<br/> (**Match**)|/analyze_test/_search?q=uuid:data-id<br/>(**Mismatch**)|
| full string | /analyze_test/_search?q=new\_uuid:wrong-test-val-id<br/> (**Match**)|/analyze_test/_search?q=uuid:wrong-data-kernel-id<br/>(**Match**)|
| regular expression without "-" tokenizer| /analyze_test/_search?q=new\_uuid:\*test\*<br/> (**Match**)|/analyze_test/_search?q=uuid:\*data\*<br/> (**Match**)|
| regular expression with "-" tokenizer & "\""| /analyze_test/_search?q=new\_uuid:"\*test-\*"<br/> (**Match**)|/analyze_test/_search?q=uuid:"\*data-\*"<br/> (**Mismatch**)|
| regular expression with "-" tokenizer & "\""| /analyze_test/_search?q=new\_uuid:/.\*test-.\*/<br/> (**Match**)|/analyze_test/_search?q=uuid:/.\*data-.\*/<br/> (**Match**)|
| regular expression with full-text string| /analyze_test/_search?q=new\_uuid:/wrong-test-val-id/<br/> (**Match**)|/analyze_test/_search?q=uuid:/wrong-data-kernel-id/<br/> (**Match**)|

* **Analysis for query string with specific field & double quotation marks**
Obviously, when we specify field in query_string, then ES only match target fields without checking with "_all" field.When starting & ending with double quotation marks, query_astring won't be tokenized with special character, just keep full text in one term.

* **Suggestion for query_string Usage**

- For exact match, please start & end with double quotation marks (") to avoid query_string being tokenized.
		> `q="req-7d502c63-e5f7-4561-b3df-3281b02965ab b91dcfcd7f1c4b3e97e43cc9723d82f1"` is **better** than `q=req-7d502c63-e5f7-4561-b3df-3281b02 965ab b91dcfcd7f1c4b3e97e43cc9723d82f1`

  
- If we know which field want to match, please explicitly specify target field to avoid checking with _all field

> `q=@fields.reqid:"req-7d502c63-e5f7-4561-b3df-3281b02965ab b91dcfcd7f1c4b3e97e43cc9723d82f1"` is **better** than `q="req-7d502c63-e5f7-4561- b3df-3281b02965ab b91dcfcd7f1c4b3e97e43cc9723d82f1"`

- For regular expression match, please start & end with "/"

> `q=new_uuid:/.*test-.*/` is **better** than `q=new_uuid:*test-*`

### Reference* [Query DSL for elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html)
* [Java_Native_Access](https://en.wikipedia.org/wiki/Java_Native_Access)* [mlockall](http://linux.die.net/man/2/mlockall)* [lucene doc_value] (http://blog.trifork.com/2011/10/27/introducing-lucene-index-doc-values/)
* [performance-considerations-elasticsearch-indexing](https://www.elastic.co/blog/performance-considerations-elasticsearch-indexing)
* [evacuate index from target node & gracefully mov out cluster] (https://www.elastic.co/guide/en/elasticsearch/reference/1.6/index-modules-allocatio n.html)
* [reindex with zero-downtime] (https://www.elastic.co/guide/en/elasticsearch/guide/current/index-aliases.html)
