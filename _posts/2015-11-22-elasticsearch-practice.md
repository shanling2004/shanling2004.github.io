---
layout: post
title: "ElasticSearch Practice"
description: "ElasticSearch Understanding & Tuning"
category: 
tags: ["es","tuning"]
---
{% include JB/setup %}
# Intorudction
We have tried to setup ELK(elasticsearch-logstash-kibana) stack as parrt of our cloud monitor eco-system.
I want to share some performance tuning experience for elasticsearch, also leads to under elasticsearch from code perspective.

Below is our ELK version.

    Elasticsearch version: 1.7.3 
    Logstash version: 1.0.9
    Kibana version: 4.0.3

#Perf Tuning Tips
## Understand ES Toplogy
ES node has 3 role responsibilities as below. They're not exclusive roles. One instance node can take one or more roles (E.g. both matser & data node).

| Role        | settings           | Responsibility  |
| ------------- |:-------------:| -----:|
| Dedicated Master Node | node.master = true; node.data=false | Only participant to work as coordinator node & won't store index repo locally|
| Dedicated Data Node      | node.master = false; node.data=true | store index repo locally, won't work as coordinator node|
| clinet node | node.master = true; node.data=false; http.enabled=false     |    load balance node |

For Detials: Please check [Elasticsearch Node](https://www.elastic.co/guide/en/elasticsearch/reference/master/modules-node.html)

Here's our ES Topo.
![es-topo]({{ site.JB.IMAGE_PATH }}/es_practice/es_topo.png "ES Topology Diagram")

