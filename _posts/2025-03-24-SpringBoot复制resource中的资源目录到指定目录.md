---
title: SpringBoot复制resource中的资源目录到指定目录
date: 2025-03-24 17:00:00
categories: [后端, 技术细节]
tags: [SpringBoot,框架操作,操作resource]
---

#### SpringBoot复制resource中的资源目录到指定目录
---
当想在程序运行时动态的更改部分配置,可以将配置放置在resource目录中,然后程序启动时在进行配置文件的读取,想要动态刷新配置可以重复的进行读取,或者代码中监听文件的变动,此时就需要将resource目录中的文件,拷贝到jar包外部的动作
---

