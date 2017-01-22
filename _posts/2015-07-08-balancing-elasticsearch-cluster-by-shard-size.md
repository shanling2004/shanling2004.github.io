---
layout: post
title: "Balancing an Elasticsearch Cluster by Shard Size"
author: Sawyer Anderson
date: 2015-07-08
---

This post will introduce [tempest](https://github.com/datarank/tempest "Tempest Plugin") - a new plugin for Elasticsearch to better balance clusters via resource awareness. I'll start by briefly outlining how Elasticsearch balances a cluster by default and some of the critical issues that can emerge, and then discuss how tempest addresses these critical issues in a more stable and efficient manner. For an in-depth analysis of some of the problems examined in this post, see [Analysis of Hotspots in Clusters of Log-Normally Distributed Data](http://engineering.datarank.com/2015/06/30/analysis-of-hotspots-in-clusters-of-log-normally-distributed-data.html) by my fellow DataRank engineer Andrew White.

[Elasticsearch](https://www.elastic.co/products/elasticsearch) is a near realtime search platform that provides a wealth of features for indexing, retrieving, and analyzing data, particularly text documents. Before we dive in to routing and balancing strategies, a quick review of the [definitions of cluster, node, index, and shard within the context of Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/_basic_concepts.html) might provide a useful refresher.

## Default Elasticsearch Cluster Balancing

Straight out of the .tar.gz (or your favorite package manger), Elasticsearch assigns shards to nodes based on the *count* of shards, attempting to assign an equal number of shards to each node in the cluster. To the credit of the Elasticsearch development team, the default balancer also supports [a number of balance weighting features](https://www.elastic.co/guide/en/elasticsearch/reference/1.6/cluster-update-settings.html#_balanced_shards) such as equalizing the number of shards per index across all nodes, or raising the tendency to equalize the number of primary shards across all nodes. 

Note, however, that none of these balancing options are resource-aware: there is no "balance shards across nodes by the size of the shards" flag to set or knob to turn. This omission becomes conspicuous to the point of instability when shards and indices vary widely in size. The graphic below illustrates this common scenario in which all nodes contain the same number of shards but differ greatly in total shard size.

<center><img src="/images/posts/2015-07-08/elasticsearch_imbalance_diagram.png"></center>

Nodes A, B, and C contain the same number of shards, but node A is larger than node B by a factor of 10, as B is larger than C. This imbalance is problematic for a number of reasons:

1. Large queries and aggregations will take substantially longer than if the cluster were evenly balanced.
2. Node A will degrade faster than the rest of the cluster due to extra CPU use, memory writes, and disk read/writes.
3. Node A will require significantly more memory to store the [field data cache](https://www.elastic.co/guide/en/elasticsearch/reference/1.6/index-modules-fielddata.html) than other nodes<sup>1</sup>, possibly exceding the memory available to Elasticsearch on that node. Elasticsearch supports [disk watermarks](https://www.elastic.co/guide/en/elasticsearch/reference/1.6/index-modules-allocation.html#disk) to prevent exceding a *disk's* capacity<sup>1</sup>, but no such breaker exists for memory usage (estimation of which would be nigh impossible anyways). This problem was the primary motivator for DataRank to create a better balancer - the largest node in a cluster would occassionally excede its available memory, which could lead to a cascading failure as shards were relocated from downed nodes to active nodes, which may already be approaching their maximum available memory.

To avoid these problems, we would much prefer an allocation such as the one shown below, which would quite probably not occur with Elasticsearch's shard count balancing strategy:

<center><img src="/images/posts/2015-07-08/elasticsearch_balance_diagram.png"></center>

Clearly a strategy that accounts for the sizes of shards and attempts to distribute equal sizes to each node could provide a better balance in many scenarios.

## Tempest Cluster Balancing

Tempest is a plugin for Elasticsearch that replaces the default balancer with its own resource-aware balancer. The balance of a cluster is determined by the ratio of the size of the largest node to the size of the smallest node<sup>3</sup>. Tempest attempts to reduce this ratio to below a [configured value](https://github.com/datarank/tempest/blob/master/README.md#configuration-elasticsearchyml), or 1.5 by default. The balancing process can be broken down into three steps: model, permute, and relocate.

#### Model

When a rebalance of the cluster is triggered (occurs when a shard changes state, a node is added or removed, etc), tempest models in-memory the current allocation of shards across the cluster. The model represents nodes and the shards assigned to them, as well as the sizes of the shards. It also expresses a few key AllocationDeciders to closely mimic Elasticsearch's allocation rules during the permute step.

#### Permute

Once the cluster is modelled, tempest looks a small number<sup>4</sup> of randomly-generated shard relocation operations in the future to determine if moving a shard will result in a better cluster-wide balance. A generated relocation must conform to the modelled allocation rules, such as not allowing allocation of a replica shard before its primary shard is active or not assigning a replica shard to the same node as its primary. The permute process of finding a better balanced model is best understood with a snippet of pseudocode:

{% highlight java linenos %}

ModelCluster bestCluster = new ModelCluster(cluster);
double bestBalance = evaluateBalance(bestCluster);
double currentBalance;

for (int i = 0; i < iterations; i++) {
	// fork the model cluster and generate a random relocation operation
	ModelCluster currentCluster = bestCluster.generateRelocationOperation();
	if (currentCluster == bestCluster) {
        // no valid relocation operations are possible
        return bestCluster;
	}

	// see if the new model cluster's balance is better than the best we've found so far
	currentBalance = evaluateBalance(currentCluster);
	if (currentBalance < bestBalance) {
        bestCluster = currentCluster;
        bestBalance = currentBalance;
	}

	// if we have generated enough relocations, return the best cluster we've found
	if (bestCluster.getGeneratedRelocatedOperations().size() >= cluster_concurrent_rebalance) {
        return bestCluster;
	}
}

// hopefully the number of iterations was sufficient to find a better cluster
return bestCluster;
{% endhighlight %}

#### Relocate

If the permute step returns a model cluster with generated relocation operations, the balancer attempts to relocate shards in the cluster to match the model<sup>5</sup>. Sometimes this isn't possible due to factors absent from the model, such as the time it takes to relocate a primary shard before its replica can be relocated. When such situations arise, the model-permute-relocate process is simply repeated when rebalancing is triggered, such as when the in-transit primary shard changes state from INITIALIZING to STARTED.

This model-permute-relocate process is repeated until the cluster's balance ratio is below the configured value, or until no better-balanced cluster can be found during the permute step.

## Results

We recently activated the tempest plugin at DataRank on a 17-node Elasticsearch cluster; it has performed remarkably well in the time since. Where previously we experienced node outages on occasion during daily bulk indexing due to overuse of memory, we have yet to experience any such outages since activating tempest. We use New Relic to monitor the performance and resource usage of this cluster, capturing the following 3-day metrics after plugging in tempest.

#### Heap Usage

<center><img src="/images/posts/2015-07-08/heap_used.png"></center>

After the switch, average heap usage decreased from ~70% to ~50% across the cluster. This is the most informative metric, as it clearly shows a significant decrease in average memory usage, and much lower variance in memory usage across the entire cluster.

#### Garbage Collection

Another major benefit of activating tempest was that the increased availability of memory on each node allowed the Java Garbage Collector to [collect in the young memory generation](http://www.cubrid.org/blog/dev-platform/understanding-java-garbage-collection/) instead of performing stop-the-world collection in the old generation. The graph below shows that garbage collection disappeared completely in the old generation.

<center><img src="/images/posts/2015-07-08/gc_old.png"></center>


The graph below illustrates that the young generation is now continuously garbage collected throughout the twice-daily bulk-indexing jobs, at a much faster rate (~10x on average) than the old generation.

<center><img src="/images/posts/2015-07-08/gc_young.png"></center>

Put simply, nodes in this cluster no longer have to cease all processing to perform garbage collection, which can now occur much faster and concurrently with indexing.

## Conclusion

Tempest increases stability, boosts performance, and improves resource efficiency in Elasticsearch clusters, particularly when shard size variance is large. Although tempest's resource usage is minor compared to that of the indices managed, tempest is best suited for mid- to large-size clusters that are not already naturally very balanced.

The [tempest repository](https://github.com/datarank/tempest "Tempest Plugin") contains installation and configuration instructions, as well as a list of known issues. The project is open-source under the MIT License; we welcome questions, comments, bug reports, and code contributions.

---
<sup>1</sup>: Disclaimer: field data cache size doesn't always map to the same memory footprint across indices. The effect of balancing based on size has the greatest effect on field data when the cluster's field cache memory usage maps to document count by some linear scaling factor.

<sup>2</sup>: Disk capacity breakers are also problematic when an Elasticsearch node shares a server with other applications. For example, if a node is part of an Elasticsearch cluster and a Hadoop cluster, used disk space is going to change quite drastically and frequently as Hadoop expands and compresses its data. If Elasticsearch starts rebalancing due to exceeding the disk high watermark when Hadoop is preparing to compress, we're needlessly (and possibly riskily, depending on the resource availability of other nodes) moving shards around the cluster.

<sup>3</sup>: That is, (node with the largest total shard size).getSize() / (node with the smallest total shard size).getSize()

<sup>4</sup>: Actually [cluster.routing.allocation.cluster_concurrent_rebalance](https://www.elastic.co/guide/en/elasticsearch/reference/1.6/cluster-update-settings.html#_concurrent_rebalance) to avoid generating moves that the cluster wouldn't be able to perform anyways. At the time of writing, [the unlimited value of -1 is not supported](https://github.com/datarank/tempest/issues/7).

<sup>5</sup>: At the time of writing, [shard allocation awareness](https://www.elastic.co/guide/en/elasticsearch/reference/1.6/modules-cluster.html#allocation-awareness) and [disk threshold watermarks](https://www.elastic.co/guide/en/elasticsearch/reference/1.6/index-modules-allocation.html#disk) are supported [but not modelled](https://github.com/datarank/tempest/issues/). That is, no move operations will occur that violate any awareness settings you may have configured, but the model-permute portion of model-permute-move does not look at allocation attributes when attempting to find move operations that result in a better balanced cluster.

