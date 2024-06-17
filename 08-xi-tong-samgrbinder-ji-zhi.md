---
description: 跨进程通信
---

# 08-系统samgr-binder机制

## 简介

服务与服务之间通信的机制。

Binder机制通常采用客户端-服务器（Client-Server）模型，服务请求方（Client）可获取服务提供方（Server）的代理 （Proxy），并通过此代理读写数据来实现进程间的数据通信。

## 框架

<figure><img src=".gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>

官方文档：[https://gitee.com/openharmony/systemabilitymgr\_safwk](https://gitee.com/openharmony/systemabilitymgr\_safwk)



