---
layout: post
title: kubectl命令补全
date: 2020-07-28
Author: gbren 
tags: [K8S]
comments: true
toc: true
pinned: true
---

## 安装bash-completion
```bash
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
```

## kubectl命令补全临时生效
```bash
source <(kubectl completion bash)
```

## kubectl命令补全永久生效
```bash
echo "source <(kubectl completion bash)" >> ~/.bashrc
```
