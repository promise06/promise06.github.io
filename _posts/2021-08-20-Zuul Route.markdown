---
layout: post
title:  "Zuul Route 流程"
date:   2021-08-20 16:07:06 +0800
categories: netflix zuul ribbon hystrix
---

SpringCloud Netflix Core 版本：1.3.x

最近在开发网关，了解一下SpringCloud 封装之后的Zuul的工作流程。注意一下版本号，Zuul 1和2 之间差别还是很大的，Netflix官网对于Zuul 1和2版本的工作原理都有详细的解释，可以参见 https://netflixtechblog.com/announcing-zuul-edge-service-in-the-cloud-ab3af5be08ee。本文从代码的层面去解释一下Zuul 1 在被作为反向代理时的工作原理,注意：所有的配置参数均使用默认值。

前置知识： [RxJava][Reactivex.io]

我把Zuul处理请求简单的分成了3个阶段：
第一阶段，请求是怎么到达Zuul所在的容器的，说直白点，请求是怎么到达ZuulServlet的
第二阶段，请求对应的目标服务是怎么被定为到的
第三阶段，请求是怎么到达目标服务的
下面我们一个阶段一个阶段的解释。














[Reactivex.io] : http://reactivex.io/intro.html