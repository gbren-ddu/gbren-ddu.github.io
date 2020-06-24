---
layout: post
title: K8S label的操作
date: 2020-06-12
Author: gbren 
tags: [K8S]
comments: true
toc: true
pinned: true
---

## label的查看
```bash
kubectl get nodes <node_name> --show-labels
```

## label的增加
```bash
kubectl label nodes <node-name> <label-key>=<label-value>
```

## label的修改
```bash
# 需要加上--overwrite参数
kubectl label nodes <node-name> <label-key>=<label-value> --overwrite
```

## label的删除
```bash
kubectl label nodes <node-name> <label-key>-
```