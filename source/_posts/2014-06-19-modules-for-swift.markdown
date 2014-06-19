---
layout: post
title: "Modules for Swift (简析 Swift 的模块系统)"
date: 2014-06-19 15:14:31 +0800
comments: true
categories: swift
---

模块是什么，当写下 Swift 中一句 ``import Cocoa`` 的时候到底整了个什么玩意？

以及 ``import Darwin``、``import CoreFoundation`` 时，从这些模块 dump 出的定义来看，几乎是完全自动生成的。






- Modules - Doug Gregor, Apple, 2012 LLVM DevMeeting [PDF](http://llvm.org/devmtg/2012-11/Gregor-Modules.pdf) [Video](http://llvm.org/devmtg/2012-11/videos/Gregor-Modules.mp4)
- [为什么应该用模块取代C/C++中的头文件？](http://www.csdn.net/article/2012-11-28/2812274-module-replace-C-based-languages-headers)
- [Modules - Clang 3.5 documentation](http://clang.llvm.org/docs/Modules.html)
