#Nodejs的迁移服务


+ **关于框架：**市面上Nodejs的框架还不少，不过既然工作中也是用的Node，那就都用同一个吧，正好可以同步的升级维护框架。

+ **关于数据：**按理说Nodejs+Mongodb是绝配，不过我之前的功能都是PHP+MySQL写好的现成的功能，好在当时设计的还可以，现在可以分分钟把接口都改造成json-api形式；所以就不倒腾到Mongodb了，前后端通过JSON接口进行交互，后端API接口沿用MySQL存储。

+ **关于Editor：**因为Blog是三年前搞的，一直用的UEditor，前厂战神的作品，功能确实好，不过现在都为了方便，都用MarkDown编辑器了，本来是等着UEditor出了Markdown版本以后再迁移这部分，不过从Rank处了解到，这个可能还需些时日，那还是不要等了，Github上也还蛮多的，我用的是jbt/markdown-editor，简单调整一下，也还蛮好用的。

+ **关于迁移：**不是将全站迁移到Nodejs，这个工作量巨大（组件平台、FeHelper、Demo平台等继续留在PHP-Smarty下）；在Nginx中简单配置下conf来实现部分请求进入Nodejs服务。

+ **关于流量切换：**baidufe.com站点的流量还不少，不敢直接在服务器上修改baidufe.com.conf，这个风险略高，不过还好备用域名还有两个，fehelper.com和doitbegin.com，而且都是备案过的，哈哈，于是拿doitbegin.com.conf开刀，这个流量很少，还好，一切顺利，调试通过，修改Nginx切主站流量，Nginx配置部分：

	server {
	    listen       80;
	    server_name  www.baidufe.com;
	    index index.html index.php;
	    root /alidata/www/baidufe;

	    # 日志文件
	    access_log  /alidata/log/nginx/access/doitbegin.log;
	    error_log  /alidata/log/nginx/error/doitbegin.log;

	    # blog首页的服务，需要走到nodejs服务中
	    location = / {
	        proxy_set_header Host $host;
	        proxy_pass http://127.0.0.1:8793;
	    }

	    # php fastcgi服务继续提供着
	    location ~ .*\.(php|php5)?$ {
	        fastcgi_pass  127.0.0.1:9000;
	        fastcgi_index index.php;
	        include fastcgi.conf;
	    }

	    # 静态文件过期时间设置为30天
	    location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|js|css)$ {
	        expires 30d;
	    }

	    # 其余所有的请求，在这里过滤
	    location / {

	        # 下面这些请求，需要走到nodejs服务中，如果不在这个列表中，则走php
	        if ( $document_uri !~ ^/(item|login|logout|admin|ajax|index|create)\b/? ) {
	            rewrite /. /index.php last;
	        }

	        # nodejs 服务走起
	        proxy_set_header Host $host;
	        proxy_pass http://127.0.0.1:8793;
	    }
	}

错误监控：流量切换后，从Nodejs的日志来看，发现有些异常访问，都是来自Google.hk，略奇怪，有时间再去分析为啥。。。错误Mark：

	2014-10-07 20:30:09 | 200  www.baidufe.com/ | 23ms | api-error:/api/data/bloglist | http://www.google.com.hk/url?url=http://www.baidufe.com/item/92457b4d0bfde1effa40.html&rct=j&frm=1&q=&esrc=s&sa=U&ei=ytwzVNqiOIHloATixIK4BA&ved=0CBMQFjAA&usg=AFQjCNFJJlCWxVSG91U_x6rg1BiLSjkfzw | Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/4.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; .NET4.0C; .NET4.0E; aff-kingsoft-ciba; .NET CLR 1.1.4322; InfoPath.2)

目前看来是基本稳定，不过买的阿里云服务是单核的，也只有一个Node进程在跑着，万一出个什么Fatal，就会彻底崩溃了，呵呵呵。。。
生命在于折腾！