## 使用webssh和lshell打造安全的shell环境

为了减轻运维压力和减少开发人员申请堡垒机的时间，我们在发布系统中引入了 [webssh](https://github.com/huashengdun/webssh)，配合 [lshell](https://github.com/chenshun00/lshell) 实现一个目前看来安全的shell环境，一键就可以看服务器日志

### 请求过程

* 开发人员浏览器: `192.168.63.127`
* 发布系统: `192.168.52.34`
* webssh: `192.168.52.75`
* 目标服务器: `阿里云ECS/云厂商ECS/Docker`

流量架构

> 开发人员浏览器 <--websocket--> 发布系统 <--weboscket--> webssh机器 <--ssh---> `阿里云ECS/云厂商ECS/Docker` 

* 安装webssh，按照文档安装即可 https://github.com/huashengdun/webssh
* 在目标机器上安装 `lshell` , 同样按照文章安装，如果需要大批量的安装，建议内部运维成yum安装的格式，因为lshell已经几年没有更新了，我又重新加了一些需求。

```txt
	git clone https://github.com/chenshun00/lshell
	python setup.py install --no-compile --install-scripts=/usr/bin
	
	useradd chen
	group addchen
	usermod -aG group chen
	chsh -s /usr/bin/lshell chen
	# 这个时候使用chen这个用户登陆上来就是处于 `lshell` 环境中
```

* 为了达到直接看日志的效果，我在[webssh](https://github.com/huashengdun/webssh)的基础上修改了前端的js，让webssh前端直接和发布系统通信，而发布系统和webssh机器进行通信，从而达到这个过程。在发布系统上起websocket，将接听的数据原封不动的写入到webssh的websocket通道上，又将返回的数据写回到前端。

### 问题

* 由于lshell是使用 `subprocess.Popen` 执行脚本，然后将返回内容写入到终端，导致内容宽度不合适失去一些数据，解决办法 `ps -efww` 
* 由于lshell是使用的 `ASCII` 编码格式，而一个中文是是3个字节，导致我们在lshell上删除的时候必须要删除3次才能删除掉一个汉字，修复方法: `export LANG=zh_CN.UTF-8` ， 如果没有 `zh_CN.UTF-8` ，使用 `en_US.UTF-8` 测试也没有问题

