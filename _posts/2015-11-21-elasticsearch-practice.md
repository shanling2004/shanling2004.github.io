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
I want to share some performance tuning experience for elasticsearch, also leads to under elasticsearch from code perspective.

Below is our ELK version.

    Elasticsearch version: 1.7.3 
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
When setting `bootstrap.mlockall=true`, elasticsearch try to lock the process address space into RAM, preventing any Elasticsearch memory from
> About bootstrap.mlockall option, Please check code for org.elasticsearch.bootstrap.Bootstrap

```java

private void setup(boolean addShutdownHook, Tuple<Settings, Environment>  tuple) throws Exception {

        
}
```

> org.elasticsearch.common.jna.CLibrary


```java
￼
```
* **Linux OS Support for mlockall**


```
2015-05-24 17:44:49,247][WARN ][monitor.jvm              ]
```
* **Solution**

> 1. in "/usr/lib/systemd/system/elasticsearch.service", let LimitMEMLOCK=infinity
> 2. ulimit -l unlimited
> 3. service elasticsearch-elasticsearch restart

* **Sanity Check**

You can check http://<root_uri>/_nodes/process, will see process.mlockall is true.

![mlockall check]({{ site.JB.IMAGE_PATH }}/mlockall.png "Process Url for mlockall check")



