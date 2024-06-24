# 16-分布式软总线

## 简介

分布式软总线子系统旨在为OpenHarmony系统提供的通信相关的能力，包括：WLAN服务能力、蓝牙服务能力、软总线、进程间通信RPC（Remote Procedure Call）等通信能力。

WLAN服务：为用户提供WLAN基础功能、P2P（peer-to-peer）功能和WLAN消息通知的相应服务，让应用可以通过WLAN和其他设备互联互通。

蓝牙服务：为应用提供传统蓝牙以及低功耗蓝牙相关功能和服务。

软总线：为应用和系统提供近场设备间分布式通信的能力，提供不区分通信方式的设备发现，连接，组网和传输功能。

进程间通信：提供不区分设备内或设备间的进程间通信能力。

## 架构

<figure><img src=".gitbook/assets/image (65).png" alt=""><figcaption></figcaption></figure>

