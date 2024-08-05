---
title: Android开发体系结构
date: 2024-05-26 20:59:57
tags: Android
categories: Android
---

<style>
    .pre-wrap {
        white-space: pre-wrap;
    }
</style>


<p class="pre-wrap" style="font-family: consolas; font-weight: 500; font-size: 15px; background-color: #EEE; border-radius: 4px; padding: 16px;"
>|-- Android
.&nbsp&nbsp&nbsp|-- Android 基础
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 四大组件
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- UI 界面
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 常用控件
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- SurfaceView
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- TextureView
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- RecyclerView
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- ConstraintLayout
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- CoordinatorLayout
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 属性动画
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- <a href="https://www.hipoom.com/2024/06/02/%E7%82%B9%E5%87%BB%E4%BA%8B%E4%BB%B6%E7%9A%84%E5%88%86%E5%8F%91/">点击事件分发</a>
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- Window 与 Surface
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- View 的工作机制
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- 大图加载
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- Jetpack
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- Android 特有的数据结构
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- ArrayMap
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 消息机制
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- epoll 机制
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 跨进程通信
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- Binder 机制
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- AIDL
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 多媒体
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 录音
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 音频播放
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- 相机
.&nbsp&nbsp&nbsp|-- Android 源码
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- Activity 启动流程
.&nbsp&nbsp&nbsp|-- Android 专项技术
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 性能优化
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- CPU 占用率
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 启动耗时优化
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 线程数量
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 线程数量监控
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- 线程数量优化
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- FPS和卡顿优化
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 卡顿监控
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- 主线程 IO 检测
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- 内存优化
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp  |-- 内存泄露
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp  |-- 内存抖动
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp  `-- 内存占用
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- C++ 内存占用分析
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- Java 内存占用分析
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 编译流程
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- Gradle
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 插件化
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- APT
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 代码插桩
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- 安全性
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 代码混淆
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- 签名
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- <a href="https://www.hipoom.com/2024/08/05/%E6%91%98%E8%A6%81%E5%92%8C%E7%AD%BE%E5%90%8D%E4%B8%8E%E6%95%B0%E5%AD%97%E8%AF%81%E4%B9%A6%E9%83%BD%E6%98%AF%E4%BB%80%E4%B9%88/">摘要、签名与数字证书都是什么？</a>
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- 加固
.&nbsp&nbsp&nbsp|-- 开源库
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- EventBus
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- RxJava
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- Okhttp
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- Retrofit
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- Glide
|-- Java
.&nbsp&nbsp&nbsp|-- 线程安全
.&nbsp&nbsp&nbsp`-- 数据结构
|-- Kotlin
`-- 架构设计
.&nbsp&nbsp&nbsp|-- <a href="https://www.hipoom.com/2024/06/01/%E5%85%AD%E7%A7%8D%E8%AE%BE%E8%AE%A1%E5%8E%9F%E5%88%99/">六种设计原则</a>
.&nbsp&nbsp&nbsp|-- <a href="https://www.hipoom.com/2024/05/26/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/">23种设计模式</a>
.&nbsp&nbsp&nbsp`-- 架构
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- MVC
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- MVP
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- MVVM
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp|-- VIPER
.&nbsp&nbsp&nbsp.&nbsp&nbsp&nbsp`-- Clean
</p>