---
layout: post
tags: [jekyll, github, git, markdown]
title: 优惠券卡包领取设计
---

  优惠券是运营一个电商网站不可缺少的功能,在搞大促的时候运营有需求需要将不同的优惠券组装成一个优惠券卡包一起发放给用户
  例如一个优惠券卡包由 三张优惠券组成,分别是优惠券 A,B,C

  那么最基本的实现就是  

	for(Card card in package){
	       receive(card)
	}


这种做法其实是有潜在的性能问题的
假如一个卡包由10张优惠券组成,那么就是10次循环,可以通过使用一些并发类来优化这个逻辑

	Executors executor = Executors.newCachedThreadPool();
	List<Future<Result>> futureList = new ArrayList();
	for(Card card in package){
       futureList.add(executor.submit(new ReceiveCardCallable(card));
	}
	boolean isAllSuccess = true
	for (Future future : futureLisT){
		try{
			Result result = future.get(1000);	
			if ( !result.success){
				isAllSuccess = false;
			}
		}catch(Exception e){
			isAllSuccess = false;	
		}
	}
	if ( !isAllSuccess ){
		rallback()
	}


通过并发同时领取卡包里的优化券可以大幅度提升领取操作的响应时间	