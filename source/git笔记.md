---
title: git笔记
tags: ['git']
categories: ['git']
---

Git主分支名为master,版本库初始化后自动建立，默认就是在master分支上开发。

master分支一般用来发布重大版本，日常开发在develop上

临时分支用完后会删除
项目中也会用来临时分支：
feature[功能]
  从develop分支迁出
```
  新建
  git checkout -b feature-x develop
  用完后合并并删除
  git checkout develop
  git merge --no-ff feature-x
  --no-ff参数：不要快进合并，会在develop分支产生新节点，而不是直接指向feature-x分支
  删除
  git branch -d feature-x

```
release[预发布]
  布正式版本之前,使用预发布的版本进行测试
bug[补丁]，
