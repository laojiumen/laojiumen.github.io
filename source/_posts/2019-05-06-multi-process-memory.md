---
title: multi-process_memory
date: 2019-05-06 11:16:12
categories:
  - python
tags: 
  - multi-process
---


在python使用多线程的时候，子进程会fork当前进程的所有数据，所以在初始化的时候，主进程和子进程不要读取任何大文件，在target函数快中读取，这样可以极大节省内存的使用率!!!!!!!!!!!
