---
title: "Basic Idea of Database System: Bustub"
date: 2024-08-22
description: CMU-15445学习笔记
cover: /img/cmu15445/cover.png
tags: 
    - 数据库系统
    - C++
    - CMU-15445
categories:
    - 笔记
---

## Bustub

Bustub is a educational database with simple structure. The basic structure of the system is: 

![Structure](/img/cmu15445/project-structure.svg)

Due to the academic integrity policy of this course, I am NOT allowed to post any code in a public repository. Thus, I'll just demonstrate the key ideas of every project with diagram.

From the very low level, we have the BufferPoolManager, which only cares about blocks of memory and has no understanding of the data it holds.

![BPM](/img/cmu15445/BPM.png)

The database system has a Catolog, which holds all tables and indexes. In this level, you can have fine-grained control over tuples and indexes.

![Catalog](/img/cmu15445/Catalog.png)

Then execute queries with Vocano Model:

![Execution](/img/cmu15445/Execution.png)

Multi-Version Concurrency Control:

![MVCC](/img/cmu15445/MVCC.png)

This is the score on gradescope. The last 20 pts of Project 4 has not been finished yet. 

![Score](/img/cmu15445/score.png)
