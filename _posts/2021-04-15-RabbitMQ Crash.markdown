---
layout: post
title:  "RabbitMQ Crash问题"
date:   2021-04-15 13:07:06 +0800
categories: rabbitmq otp
---
RabbitMQ 版本：3.6.6

OTP 版本：19.1

这个问题事件跨度比较长，复现的难度比较大，而且RabbitMQ 和 OTP 的版本较低，可能在实际应用的参考意义不大，但是对于理解erlang的内存模式是一个非常完整的教学案例。鉴于erlang语言的受众范围较小，本文会尽量减少语言相关的词汇的应用。
