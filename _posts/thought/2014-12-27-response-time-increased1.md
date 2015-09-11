---
layout: post
title: 记一次 Response Time 增加的问题
category: 技术
tags: 故障
keywords: 故障处理
description: 
---

今天 service 发布之后发现两个 API 的 reponse time 增加了大概 20ms， 但是看本次所有的 commit 好像没有和这些 API 相关的修改。观察监控的曲线， 发现我们服务依赖的一个 service 的请求时间增加了很多， 但这种情况仅仅发生在两个 API， 还有其他依赖于该 service 的 API 却没什么变化。

还好这次发 布提交的 commit 不多， 一个个 commit 看过来， 发现仅仅有一个 API 的修改可能涉及到该 service， 它请求该 service 的时候多返回了一些字段的信息。 该 service 主要是返回一些用户的行为， 其中主要包括用户观看 video 的信息， 用户保存video 或者 show 的信息等。相关的修改是多拿了一些用户的信息， 即请求的参数中 fields 增加了一些字段， 返回用户更多的行为。 但是与这个修改相关的 API reponse time 并没有明显的变化， 反倒是没有修改的两个 API 请求该 service 的 response time 有了一个明显的变化， 这个就比较奇怪了。 经过仔细的分析， 之前这个修改的 API 请求的用户行为在修改之前和本次 reponse time 增长的 API 拿的一样。到这里似乎找到了他们的共性， 问题也有了一点眉目， 这次 reponse time 增长的 API 本身的请求量比较小， 修改的 API 请求量比较大， 结合到依赖的 service 有 cache， 似乎问题可以得到解释。RPS 比较大的 API 请求的互用行为很容易进入 cache， 之后两个请求量比较小的 API 的用户行为就可以命中 cache， 但是本次的修改了 rps 较大的 API 请求的用户行为改变之后，请求量较小的 API 就不能命中 cache了，所以 reponse time 增加了 20ms。

---

##### 心得：

1. 事物之间具有比较强的关联性， 需要细心去发现问题。
2. 依赖的 service cache 设置不合理，对于请求的行为是 cache 中的已有行为的一个子集的话， 应该可以直接使用 cache 中的数据， 就可以避免类似的问题。