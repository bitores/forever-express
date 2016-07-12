#forever守护进程

**备注：**
forever作为node的守护进程而存在的，因为node服务端很容易遇到异常就崩溃，在用这个的时候最好给系统添加 异常报警，异常记录之类的措施，不然用了forver也没有起到任何真正的作用， 不能因为有了forever就忽略了try catch，代码里应该多加异常捕捉，防止程序经常出现崩溃的情况。让程序更加健壮。

我们可以uncaughtException来全局捕获未捕获的Error，同时你还可以将此函数的调用栈打印出来，捕获之后可以有效防止node进程退出，如： 

 

	process.on(＇uncaughtException＇, function (err) {
	  //打印出错误
	  console.log(err);
	  //打印出错误的调用栈方便调试
	  console.log(err.stack)；
	});



###forever使用说明
1. 简单的启动
forever start app.js

2. 指定forever信息输出文件，当然，默认它会放到~/.forever/forever.log
forever start -l forever.log app.js

3. 指定app.js中的日志信息和错误日志输出文件，-o 就是console.log输出的信息，-e 就是console.error输出的信息
forever start -o out.log -e err.log app.js

4. 追加日志，forever默认是不能覆盖上次的启动日志，所以如果第二次启动不加-a，则会不让运行
forever start -l forever.log -a app.js

5. 监听当前文件夹下的所有文件改动（不太建议这样）
forever start -w app.js

6.启动所有
forever restartall


###开发和线上建议配置
开发环境下
NODE_ENV=development forever start -l forever.log -e err.log -a app.js
线上环境下
NODE_ENV=production forever start -l ~/.forever/forever.log -e ~/.forever/err.log -w -a app.js

上面加上NODE_ENV为了让app.js辨认当前是什么环境用的。不加它可能就不知道哦？

有可能你需要使用unix下的crontab（定时任务）
这个时候需要注意配置好环境变量。
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin


我们要让Forever自动运行，先在/etc/init.d目录创建一个文件node，内容如下：
```
	#!/bin/bash
	#
	# node      Start up node server daemon
	#
	# chkconfig: 345 85 15
	# description: Forever for Node.js
	#
	PATH=/home/node/0.8.9/bin
	DEAMON=/home/ftp/1520/weizt-20120918-tKx/weizt.com/app.js
	LOG=/home/hosts_log
	PID=/tmp/forever.pid

	case "$1" in
	    start)
	        forever start -l $LOG/forever.log -o $LOG/forever_out.log -e $LOG/forever_err.log --pidFile $PID -a $DEAMON
	        ;;
	    stop)
	        forever stop --pidFile $PID $DEAMON
	        ;;
	    stopall)
	        forever stopall --pidFile $PID
	        ;;
	    restartall)
	        forever restartall --pidFile $PID
	        ;;
	    reload|restart)
	        forever restart -l $LOG/forever.log -o $LOG/forever_out.log -e $LOG/forever_err.log --pidFile $PID -a $DEAMON
	        ;;
	    list)
	        forever list
	        ;;
	    *)
	        echo "Usage: /etc.init.d/node {start|stop|restart|reload|stopall|restartall|list}"
	        exit 1
	        ;;
	esac
	exit 0
```
以上代码是我在本地虚拟机的配置，根据实际情况修改相关参数，主要是DEAMON的路径参数，赋予该文件可执行权限，并运行chkconfig添加自动运行：

	chmod 755 /etc/init.d/node
	chkconfig /etc/init.d/node on

reboot重启系统，通过浏览器进入网站可发现，该NodeJS已经可自动运行了……


###node nginx

	server {
	    listen 80;
	    server_name mysite.com;
	    location / {
	        proxy_set_header    X-Real-IP           $remote_addr;
	        proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
	        proxy_set_header    Host                $http_host;
	        proxy_set_header    X-NginX-Proxy       true;
	        proxy_set_header    Connection          "";
	        proxy_http_version  1.1;
	        #响应慢的解决方法
	        proxy_connect_timeout 1; #后端服务器连接的超时时间
	        proxy_send_timeout 30; #等候后端服务器响应时间
	        proxy_read_timeout 60; #后端服务器数据回传时间
	        proxy_pass          http://localhost:3333;
	    }
	}