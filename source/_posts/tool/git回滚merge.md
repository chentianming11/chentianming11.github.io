---
title: git回滚merge
tags: git
categories: tool
abbrlink: 226800025
date: 2021-05-19 21:15:49
---

在日常工作中，经常出现分支合并错误的时候，这个时候需要将此次`merge`操作回滚。本篇文章介绍了回滚`merge`的方法。

<!--more-->

## 命令

1. `git reflog`：查看merge操作的上一个提交记录的版本号
2. `git reset --hard 版本号` ：这样可以回滚到merge之前的状态

## 示例

误将`dev`合并到了`master`分支，现要回滚`merge`操作

1. 首先`git reflog`
    ```text
    ee0ee93 HEAD@{0}: merge dev: Merge made by the ‘recursive’ strategy.
    7335548 HEAD@{1}: checkout: moving from dev to master
    ```
    可以看到需要回滚到 7335548 这个提交记录上。

2. 执行`git reset --hard 7335548`
    再次查看提交记录：
   ```text
    7335548 HEAD@{0}: reset: moving to 7335548
    ee0ee93 HEAD@{1}: merge dev: Merge made by the ‘recursive’ strategy.
    ```
