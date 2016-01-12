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
When setting `bootstrap.mlockall=true`, elasticsearch try to lock the process address space into RAM, preventing any Elasticsearch memory frombeing swapped out.
> About bootstrap.mlockall option, Please check code for org.elasticsearch.bootstrap.Bootstrap

```java

private void setup(boolean addShutdownHook, Tuple<Settings, Environment>  tuple) throws Exception {
        if (tuple.v1().getAsBoolean('bootstrap.mlockall', false)) {
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




