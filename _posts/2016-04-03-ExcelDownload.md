---
layout: post
tags: [jekyll, github, git, markdown]
title: 运营后台报表下载优化
---

   <h2>背景描述:</h2>
   		导出Excel是一个运营系统必备的功能,最简单的做法是在Controller里完成下载逻辑,生成excel以后将OutputStream输出到HttpServletReponse的Writer，这种做法的问题在于如果下载报表的逻辑较为复杂,无法在http请求的超时时间内完成就会导致运营无法下载到报表







   <h2>解决方案:</h2>
需要一个报表下载调度框架完成这个事情,当运营点击下载按钮的时候只是提交一个下载请求,由框架异步去完成下载的任务,然后将生成的excel上传到服务器之后发邮件/短信通知运营报表已经下载完成了

框架只是做调度的事情,实际的下载工作是由业务方完成的.业务方需要将下载excel的接口注册到报表调度平台,这个接口的参数是一个map,因为如果要报表平台去做参数解析的工作的话就会比较麻烦,接口如下
	
	/**
	业务系统需要根据参数生成excel，然后将excel上传到服务端,最后返回一个文件下载的链接
	**/
	String downloadExcel(Map param)


调度平台的代码

   
     DownloadService dowloadService = DownloadServiceFactory.get(key)
	 String  file dowloadService.downloadExcel(param);
	 notify();


这样的话运营就只需要点击一下按钮就可以了，而不需要苦苦等待，或者是在等待了几分钟后看到一个下载失败的页面

	 

