---
layout: post
title: "SOSP17 Azure Resource Central Paper Read Report"
keywords: ["cloud","resource-scheudler"]
description: "High Efficient CLoud Resource Management"
category: "cloud"
tags: ["cloud","resource-scheduler"]
---
{% include JB/setup %}


# 背景

这两天 学习[SOSP17 会议](https://www.sigops.org/sosp/sosp17/)时，有幸发现一篇文章 和之前在云平台工作相关度很高 相关背景和挑战一点就明了。这篇文章给了我很多启示 开了眼界 也在反思当年思考的不足。

文章全名叫： [Resource Central: Understanding and Predicting Workloads for Improved Resource Management in Large Cloud Platforms](https://www.microsoft.com/en-us/research/wp-content/uploads/2017/10/Resource-Central-SOSP17.pdf)

本文旨在概述Azure云平台资源调度系统 如何更高效 更节能地调度云计算资源。我相信 这对很多公有云而言都是million-dollar问题。

给大家举个实际的例子，当年我们在国外某电商云平台的单个数据中心 针对QA环境做20% VM Oversubscription, 累计硬件投资和水电煤开销 每年能节省**200万**美金 （不计人力维护成本）。

* 需要声明的是 这并非原创文章 几乎是Azure 云团队论文的知识搬运总结而已。而且 由于个人能力和兴趣点 并非面面俱到（比如 关于线上线下 系统构架和反馈机制 我并未涉及过多）。

# 总结分享

## 云资源调度管理平台的硬指标

![Cloud Resource KPI]({{ site.JB.IMAGE_PATH }}/cloud/Cloud_Resource_KPI.png "Cloud Resource KPI")

* 有些指标是相互矛盾的，比如我们需要只是over-subscription超卖 但同时又需要保证上线所有VM资源 能在充足的宿主机资源上运行 不影响线上服务质量。

* 又如 Resource Compaction，这事需要全面考虑（很容易得出局部最优解，但很难快速收敛到全局较优解） 而且是在动态变化的。

简单的例子 如果我有3台宿主机 各自剩下 1VM Slot可用，能不能调整VM分配之后，让只有一台宿主机能单独分配3VM slot，其他两台分配饱和）。
同时 如何调整（如何防止flapping），到底是搬出大VM还是搬出小VM，有哪些影响因素。

* 一旦为了达成更紧凑的VM资源分配，我们势必要让某些VM搬家 从hypervisorA 搬去 hypersiorB, 同时最好不要有线上downtime，这就要求VM Live MIgration，这是个高开消的行为 可能需要几个小时才能完成（如果有大数据的local disk需要一起迁移）。能不能尽量在同Rack 同一个top-of-rack switch进行migration 以减少跨网络开销。
* 不同环境决策也不同 在QA环境（线上影响小） 完全可以采取更为激进的策略 production环境需要更加小心 甚至需要和SRE团队紧密协调合作 
* 不同VM上跑的不同类型service 也有很多差异性， 比如有些是风控 支付 敏感度很高的服务 有些也许只是后台服务 或者线下分析业务的service，他们对QoS要求也完全不同。

## 云资源调度 核心因素
![Cloud Resource Character]({{ site.JB.IMAGE_PATH }}/cloud/Cloud_Resource_Character.png "Cloud Resource Character")
【参看Paper Section#3】
* WorkLoad Class: 化繁为简的方式 有助于之后的预测 不追求精准度 追求问题决策的收敛。
* Metrics Correlation: 这是云资源调度和监控的一大难点和重点。很多指标都有一定的相关性，
比如Hypervisor和VM metrics的联动
比如hypervisor MCE issue（memory chip exception）引发VM不稳定 进而引发系统开销陡增。

## 云资源调度 技术挑战和 Azure团队应对之策
![Cloud Resource Analysis]({{ site.JB.IMAGE_PATH }}/cloud/Cloud_Resource_Difficulty.png "Cloud Resource Difficulty")
1. scheduler逻辑实现:（a）硬性逻辑filter （b）根据实际allocation quota 和预测utilization 做进一步filter （c）apply 软限制 进行排序 【参看Paper Section#4.5】
2. [bin-packing problem](https://en.wikipedia.org/wiki/Bin_packing_problem):对于CPU Memory Disk资源 全考虑 要分配的完全没有碎片 可以抽象成三维 装箱问题，一般都会至少降维成2维 甚至1维（这也是为什么Azure团队引入workload class的原因之一）装箱问题

## 云资源调度 个人建议
![Cloud Resource Analysis]({{ site.JB.IMAGE_PATH }}/cloud/Cloud_Resource_Suggestion.png "Cloud Resource Suggestion")
* 事后调度: 调度分两大阶段[事前和事后]。
由于事前调度 追求在较短时间内完成较优的决策，而且多个较优解 不一定能达成 最优解。同时VM Resource Utitlization是个动态变化 的因素 因此很有必要进行事后VM调整 为了以下的目的：

	** 更好的紧凑度 & 压缩VM碎片
       ** 防止高敏感度的服务竞争
* CPU Underlocking: 有时候 会骗人 在低workload情况下 CPU会做降频的动作来降低能耗开销 具体要查看BIOS配置。

# 分析方式学习

这块也是我的短板 当年就是苦于这部分知识和经验不足 导致不能完全推进项目进一步发展。来看看Azure团队数据分析的实践

![Cloud Resource Analysis]({{ site.JB.IMAGE_PATH }}/cloud/Cloud_Resource_Analysis.png "Cloud Resource Analysis")

* [CDF](https://en.wikipedia.org/wiki/Cumulative_distribution_function): 全文通篇都是CDF 先行 尤其是在判断云调度 核心因素的时候，简单来说 通过了解每个因素值的使用比例 来专注精力解决具有最大比例的问题 不追求大而全。
* [COV](https://en.wikipedia.org/wiki/Coefficient_of_variation): 为了判断相似度， 文中很多归类的预处理都是通过COV完成 例如判断同类型的subscription VM resource usage相似度很高。
* [Spearman Correlation Coefficient](https://en.wikipedia.org/wiki/Spearman%27s_rank_correlation_coefficient)：在分析correlation metrics的时候， 作者使用Spearman相关性分析来评估不同 调度因素之间的关联度。至于为何没有使用，我个人猜测是对于连续数据，正态数据，线性数据用[person相关系数](https://en.wikipedia.org/wiki/Pearson_correlation_coefficient)是最恰当的，当然也可以用spearman相关系数，虽然效率没前者高。spearman适用的场景更宽泛些。
* [Fast Fourier Transformation](https://en.wikipedia.org/wiki/Fast_Fourier_transform): 根据CPU load trend来快速归类VM workload class，作者通过快速傅里叶变换 做出特征值 然后在由此建模 决策分类 打分
* Classifier: [Random Forest](https://en.wikipedia.org/wiki/Random_forest) & [Gradient boosting Decision Tree](https://en.wikipedia.org/wiki/Gradient_boosting), 文中并未细说构建过程 估计是为了防止决策偏见，通过以上2种方式，统一最最后决策.


# 个人感想

* 你不是一个人在战斗 山外有山 要用更开放的心态 多和业界交流 
* 产学研结合 这个问题 不是纯粹工程性的问题 或者 研究 数理分析问题 一定要整合多种人才一起合作。理论通过事实数据论证 同时一线的同志也能基于use case快速提供建特征变量和规律的建议
* 你不是超人 不可能搞定所有问题 了解所有知识领域
* 有多大的业务挑战和体量才有多大的技术挑战和研发投入
* 对于自己做过的项目 要定期总结 思考 回顾 我曾经想写一篇做了2-3年云资源掉嘟嘟恩分享 写了一大半 可惜没做好代码管理 宕机之后数据资料丢失(sigh~~~)


# 开放性问题

* 这套云资源调度中心系统 和Openstack nova scheduler 和 kubernetes resource management for container有何差别 侧重有何不同？
* 整套系统 对私有云 有何实际意义？思路 构架 数据分析 建模有何难易 有何参考价值？