---
layout: post
tags: [jekyll, github, git, markdown]
title: Redis持久化策略
---
本文是根据官网的持久化策略部分，自己做的一些总结，详情请移步  [https://redis.io/topics/persistence](https://redis.io/topics/persistence)

redis一般都会用来和memecached比较，除了redis支持的数据类型以外,还有一个很大的区别是redis支持数据持久化，在系统宕机了以后可以通过快照文件或者aof文件恢复数据
# Redis持久化策略
## RDB
rdb是指按照指定的间隔，将当前的数据全部转储到一个快照文件中

### 优势
	可以按照不同的策略备份文件，比如每24小时备份一次，每小时备份一次等等
	
	备份文件可以转储到远端的数据中心进行加密存储
	
	最大化了redis的性能，因为rdb只需要父进程fork一个子进程，父进程不会做相关的io操作，当然这里也有一个潜在的问题是，fork子进程会消耗大量的内存，所以对机器的内存要求较高
	
	rbd恢复速度比aof快，rdb是直接恢复数据，aof是重放命令
	
### 劣势
	rdb有丢失数据的风险，因为rdb的备份策略是定时，或者通过save point触发备份，那么在没有达到rdb条件时系统宕机，就会造成数据丢失
	
	rdb的fork操作在数据量很大的时候，有可能会造成秒级的服务停止响应
   

## AOF

#### 优势
	aof数据丢失风险小，aof可选策略包括 每秒同步，每个命令同步，不过即使在每秒同步的情况下，性能也是非常ok的
	
	aof文件是一种只追加的数据格式，不会有数据正确性的问题，就算有没有完全写入的命令，redis的 check-aof的工具也能自动修复（如何修复？）
	
	当aof文件过大时，redis有自动rewrite机制（例如 命令优化，最简单的例子 合并n个incr命令），当新文件创建好后，redis会快速的切换过去
	
	aof包含了可读的命令，比如如果错误的指向了flush all，那么这时候可以关闭服务，修改aof文件删除掉这个命令中，再重启服务就可以恢复数据了
	
	

#### 劣势
	aof文件通常比rdb文件大
	
	相对来说，aof比rdb慢
	
	aof恢复数据的过程中，执行某些命令可能会有bug，rbd则健壮得多
	
## 如何使用
	官方推荐 aof和rdb一起启用，后续有计划统一aof 和 rdb
	
## 细节
#### RBD工作机制
	redis的父进程fork一个子进程
	
	子进程将数据集写到一个临时的rdb文件
	
	写完后，替换旧的rdb文件
	
	这种方式可以让redis受益于copy-on-write的语义	
	
#### AOF工作机制
	每一个会改变数据集的命令，都会被写入到aof文件中
	
	redis支持aof文件到重建，例如一个incr命令执行了100次，在原始状态下就会有100行命令，但是在重建后会精简为1个命令incr 100
	
##### 当aof文件被截断，会发生什么
	当正在写入aof时系统宕机，或者磁盘满了，这时候会发生什么呢？
	
	