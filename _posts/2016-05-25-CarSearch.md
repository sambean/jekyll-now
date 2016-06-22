---
layout: post
tags: [jekyll, github, git, markdown]
title: 如何更精准的搜索到车
---

最近在做一个新车/二手车搜索的项目,车有很多需要被搜索的属性,例如
   <ul>
		<li>颜色</li>
		<li>品牌/车系</li>
		<li>国别</li>
		<li>年款</li>
		<li>排量/国标</li>
		<li>其他....</li>
	</ul>
 通过分析用户的搜索关键字可以发现[日志分析](http://www.baidu.com)用户很喜欢输入诸如以下的关键字:
<ul>
	<li>白色奥迪</li>
	<li>15万左右的SUV</li>
	<li>便宜的日系车</li>
	<li>1.5l排量的车</li>
	<li>诸如此类....</li>	
</ul>
我是用[elasticsearch](https://www.elastic.co/products)实现搜索的,一个车的部分数据如下


 
	seriesPriceHigh: 8.16,
	priceTypeId: 0,
	indexTimeStamp: 1466545195782,
	modelId: 15927,
	modelName: "2013款 1.8L 手动尊贵型",
	brandId: 25,
	seriesId: 989,
	modelType: "2013款",
	seriesName: "GX7",
	modelLevelName: "紧凑型SUV",
	seriesBodyStructure: "SUV",
	brandName: "吉利汽车",
	national: "中国",
	engineCapacity: 1792,
	elecSky: "●",
	modelTransmission: "手动变速箱(MT)",
	panSky: "-",
	stabCon: "-",
	engineCapacityTypeId: 3,
	brandEName: "geely",
	revVideo: "●",
	brandPYEM: "jiliqiche",
	fuelConsumption: "8.1",
	modelCode: "SUV",
	seriesPrice: "8.16-13.69",
	seriesPic: "http://car0.autoimg.cn/car/upload/2015/5/12/t_201505120909270105132112.jpg",
	vendor: "吉利汽车",
	seriesTransmission: "手动,自动",
	gearbox: "手动变速箱(MT)",
	gearboxShortName: "5挡手动",
	engine: "1.8L 139马力 L4",
	saleState: 1,
	modelCode: "跑车",
	seriesPrice: "376.11-485.84",
	seriesPic: "http://car3.autoimg.cn/cardfs/product/g6/M07/E0/74/t_autohomecar__wKgHzVZIVpyAGHyxAAVthSQi_RQ315.jpg",
	vendor: "阿斯顿&middot;马丁",
	seriesTransmission: "自动,手动",
	gearbox: "手动变速箱(MT)",
	gearboxShortName: "6挡手动",
	engine: "6.0L 456马力 V12",
	saleState: 1,
	color: "星空蓝"

例如搜索 白色奥迪,需要搜索color字段和brandName字段,这时候就需要使用elasticsearch提供的[multi_match](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/multi-match-query.html),multi_match是将match query 应用到不同的filed上,[match](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/match-multi-word.html) query 有个参数是 operator，operator有2个值 and 和 or,如果是and的话,那就意味着query的term必须同时出现在搜索的filed里,而or的话则只要有一个term出现在搜索的field里即可,同时可以通过 minimum_should_match来控制匹配度,下面我们来分析一下使用multi_match去查询 color 和 brandName字段是否可以满足需求:
首先白色奥迪这个query会被ik分词器分为  白色,奥迪 这2个term
<ul>
	<li>
		1.operator使用or，那么会导致颜色为白色的其他车子也被搜索到,这个不符合业务场景
	</li>
	<li>
		2.operator使用and,那么肯定是搜素不到数据,因为brandName里肯定没有颜色相关的term，通用color里也没有brandName相关的term
	</li>
	<li>
		3.operator使用or，minimum_should_match设置为50%,同样会导致颜色为白色的其他车子也被搜索到,这个不符合业务场景
	</li>
</ul>
那么如何满足这个需求呢,一个可行的方案是使用一个额外的复合字段searchKey,这个字段包含了车辆的品牌颜色(另外一种方案是enable [_all](https://www.elastic.co/guide/en/elasticsearch/reference/2.3/mapping-all-field.html#excluding-from-all)字段,同时将不需要被搜索的字段的include_in_all设置为true,然后去match all字段,这个方案的缺点是无法通过http api直观的看到_all字段的值)，这时候只需要搜索这个searchKey就可以了

那 15万左右的SUV 这种搜索如何实现呢? 有2种方案可选
<ul>
	<li>在搜索的时候分析关键词,提取出和价格相关的关键词,然后转换成数字,再使用range query去查询,这个方案的缺点很明显,从关键字中提取价格相关的关键字不是那么简单的一件事情
	</li>
	<li>
		在构建的时候进行预处理，例如一辆车的售价是 14.98万,那么就往searchKey里设置 12万,13万,14万,15万,16万，同时把 左右 做为停用词配置在词库里,至于为什么是上下2万这是个经验值,更好的做法是根据不同的价位做不同的浮动,例如想要买几万以下的车的用户也许几千块的差距都会影响他们做判断,而想买100万以上的豪车的土豪们,也许10W左右的差价对于他们来说也能接受
	</li>
</ul>
毫无疑问,方案二是更优的选择   

 那 便宜的日系车  ,油耗低的suv 这种搜索呢?这就需要给车辆打标签了,例如 我们定义油耗在 2.0l以下的车为油耗低的车,那么在构建索引的时候就把满足条件的车 打上 油耗低 这个标签,同样 便宜 也是这样处理

 综上,最终的searchKey的值可能是这样的  
<p><b>
 2013款 1.8 手动尊贵型,2013款,紧凑型SUV,GX7,null,,SUV,SUV,geely,jiliqiche,吉利汽车,国产,国产车,中国,中国车,1.1l左右,1.0l左右,0.9l左右,0.9l左右排量,0.9l左右,排量0.9l左右,1.0l左右排量,1.0l左右,排量1.0l左右,1.1l左右排量,1.1l左右,排量1.1l左右,,,电动天窗,,电动天窗,,倒车视频影像,后倒车雷达,前倒车雷达,手动变速箱(MT),便宜,油耗低
</b></p>
 思路就是把搜索可能会用到的配置项,自定义标签全部都设置到searchKey里然后搜索searchKey这个字段

 这样做可以满足大部分的用户需求,当然也会有一些意外的情况出现,例如 用户在 搜索 宝马的时候 竟然出现了 奇瑞,排除后发现是奇瑞的一款车的颜色是 宝马白,词库里又没有 宝马白 这个词就将 宝马白  切分为了 宝马,白,这个就只能通过不断的更新词库来优化搜索结果了

来个实例:  [15万左右油耗低的日系SUV](http://www.che.com/ershouche/15%E4%B8%87%E5%B7%A6%E5%8F%B3%E6%B2%B9%E8%80%97%E4%BD%8E%E7%9A%84%E6%97%A5%E7%B3%BBSUV)


   

 