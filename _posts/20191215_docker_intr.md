docker实乃神奇也

## docker的作用
	将应用程序和依赖进行打包，方便部署，避免出现“在我的机器上是好好的”这种问题，和虚拟机的区别是 docker容器化是基于进程级别的隔离，底层的操作系统还是公用的，虚拟机则是操作系统层面都是隔离的


## docker的优点
	速度快，方便

## dokcer的相关概念
	镜像  
	容器
	仓库
	

## docker常用命令

### 启动容器
	docker run -d -p x:y  image_id
	-d : 后台运行
	-p : 端口映射
	
### 停止/移除容器
	docker stop/rm container_id	

