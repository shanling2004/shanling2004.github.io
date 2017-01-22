---
layout: post
title: "Analysis of Hotspots in Clusters of Log-Normally Distributed Data"
author: Andrew White
date: 2015-06-30
---

This post will be exploring the statistical nature of log normal distributed data with an emphasis on how it affects the
 design of Elasticsearch clusters. However, the data and conclusions should be relevant beyond the scope of Elasticsearch.

<!-- loading MathJax here unless there is a better way elsewhere -->
<script type="text/javascript"
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>

First, let's define some terms. At Datarank, we track "topics". A topic is nothing more than a collection of comments
that match some criteria. These criteria can range from being rather broad to very specific. A shard is a slice of an Elasticsearch
index that holds multiple topics. An index is made up of multiple shards. We will use certain statistical shorthands as
well such as "PDF" to mean a probability distribution function, "stddev/σ" for standard deviation. Lastly, all graphs
based on empirical data were first mapped to a smooth kernel distribution with 1/2 σ bandwidth.

## Topic Sizes

First, lets look at topic sizes. Topics ranged in size from very tiny to very large with most having between 10k and 100k
comments. This looked like a classic log-normal distribution and sure enough it was with a mean of 9.022 and a σ of 2.832.

<!--
  -- switching to raw HTML because...
  -- Liquid Exception: Unknown tag 'img'
  -->
<img src="/images/posts/2015-06-16/pdf-topic-size.png">

Since we will be referencing this distribution later and to save a trip to a stats textbook or Wikipedia, the PDF for
topic sizes can be modeled using the following equation.

 $$P(T)=\frac{1}{x\sigma\sqrt{2\pi}}e^{-\frac{(\ln x - \mu)^2}{2 \sigma^2}};\;\mu=9.022\,\mathrm{and}\,\sigma=2.932$$

As an interesting aside, this is the same equation used to model the spread of highly communicable epidemics. Since
topics track abstract concepts, this may provide some good evidence for modeling highly shared content, such as ads and
memes, as viruses.

## Shard Sizes

For performance reasons, we like to map a topic to a minimum number of shards, ideally one. To that end we use
Elasticsearch's routing feature and use the topic's numerical id as the routing key. It's important to note that routing
is static so that document placement is predictable and stable as more data is added to the index.

Since a shard is simply a collection of topics, it's size PDF can be expressed as follows.

 $$S = T + T + T + ... + T$$

Unfortunately, there is no closed form for the sum of the log-normal distribution. However, we can calculate the expected
  value for $$P(S)$$ by noting that topic sizes are independent. Therefore we can say that
  $$\mu_S = numberOfTopics*\mu_T = numberOfTopics*e^{\mu+\sigma^2/2}$$

The graph below illustrates PDFs for shard sizes of a variety of topic groupings. Note, for this graph we used random
variant sampling from $$P(T)$$ and then a smooth kernel distribution to build the histograms.

<img src="/images/posts/2015-06-16/pdf-single-shard-size.png">

From the graph, it is clear that the mode (and the mean) increase linearly as the number of topics increases. In addition,
we can see that the variance increases as well; however, this rate is not linear.

## Index Balance

Since an index is simply a collection of shards we can start to talk about balancing an index across many shards. Ideally,
shards should map one to one to physical nodes. However, this becomes impractical with log-normally distributed data as
we will soon discover.

Let's define how balanced an index is by the ratio of the smallest shard to the largest shard. A ratio of 1 would
be perfect; whereas, a ratio of 2 would mean that the largest shard is 2x larger than the smallest shard. We use this
ratio over something like a standard deviation due to importance of overloaded nodes. In general a ratio of 1.25 - 1.50
is quite well balanced

There are two variables to consider when thinking about this problem: number of shards and number of topics. While it is
tempting to think of these as symmetric variables, they are not. The graphs below show how the index balance ratio
changes with various topic and shard count configurations.

<img src="/images/posts/2015-06-16/pdf-10-shard-ratio.png">

<img src="/images/posts/2015-06-16/pdf-shard-ratio-10k.png">

Note that the imbalance is least when we have fewer denser shards. This implies that horizontal scaling will
actually make the problem worse. Second, as more topics are added to an existing cluster, it should naturally become
more balanced over time.

Since the cluster naturally becomes more balanced with volume the problem is solved, right? The problem manifests itself
when two critical aspects are noted. First, even in optimistic cases the ratio is still quite high. These high ratios
mean that the cluster runs the risk of overloading a node during a rebalancing. Second, very large shards can run into
performance issues for aggregation of very large topics.

Another aspect to consider is that the default Elasticsearch shard allocator does not take shard size into account when
distributing shards in a cluster. Therefore, simply allocating more shards than nodes and letting Elasticsearch
distribute the shards does not solve the issue and can actually make the problem much worse.

Luckily there are some workarounds to help mitigate the problem. First, separate large topics into their own
single-topic indexes; then use a resource aware balancer to smartly spread out smaller shards.

## Large Topic Separation

Large topics are the root of the problem. One very large topic can overload a node when lots of field data is needed for
aggregations. Getting several large topics on a single shard can make it several times larger than any other shard. In
short, large topics need some special attention.

Since topic's sizes are distributed log-normally, it makes sense to build our exclusion rules around the standard
deviation of $$\ln topicSize$$. Below is the equation used to determine if a topic is excluded for a given σ.


$$topicSize > e^{\mu_T + s\sigma_T} \Rightarrow topicSize > e^{9.022 + 2.832s}$$

The graph below shows the impact on the index balance ratio from separating out large topics from the core index based
on various scales (s) from the equation above.

<img src="/images/posts/2015-06-16/pdf-large-topic-excludes.png">

At σ = 3 almost nothing is being excluded and the mode imbalance ratio is around 2.75. Exclude just 2% of the topics
(σ = 2.25) and the mode imbalance ratio drops to 1.8; however, with thousands of topics, this produces several dozen
separate indices that have to be managed. At Datarank we use a σ of 2.35 and get fair results out of the default allocator.

## Conclusion

In this post we have shown that hotspots in statically routed documents that fit into log-normally distributed topic sizes
will always produce hotspots. However, by separating out very large topics, the imbalance of shard sizes decreases
dramatically. However, this is still far from ideal due to the default Elasticsearch allocator being resource-blind. This
is why Datarank has developed a custom resource-aware allocator that will be released and commented upon soon.
