---
layout: post
title: nginx vhost
category: nginx vhost web
tags: [nginx]
---
在服务器架构上，一般我们会选用LAMP或者LNMP的配置。前面我们说明了一些关于apache配置的东西，这里大概描述下nginx的配置。关于怎么安装LNMP这里就不做赘述，网上也存在很多教程。

### nginx配置文件
nginx的配置文件，一般都存在于nginx目录下的conf目录。如下图所示
![nginx vhost]({{ site.url }}/images/{{page.title}}/Snip20160403_1.png)
所有的配置都可以在此设置

### 配置vhost
nginx的vhost配置文件存在于conf/vhost目录，结构如下
![nginx vhost]({{ site.url }}/images/{{page.title}}/Snip20160403_2.png)
我们可以为每个虚拟主机路径建立一个文件，当然文件名没有什么特殊的要求，只要以conf结尾即可。一般我们会起一个容易标示的名字。

#### 配置说明
首先我们来看看phabricator.conf的内容
{% highlight sh%}
server {
  server_name phabricator.powell.com;
  root        /alidata/www/phabricator/webroot;

  location / {
    index index.php;
    rewrite ^/(.*)$ /index.php?__path__=/$1 last;
  }

  location = /favicon.ico {
    try_files $uri =204;
  }

  location /index.php {
    fastcgi_pass   localhost:9000;
    fastcgi_index   index.php;

    #required if PHP was built with --enable-force-cgi-redirect
    fastcgi_param  REDIRECT_STATUS    200;

    #variables to make the $_SERVER populate in PHP
    fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
    fastcgi_param  QUERY_STRING       $query_string;
    fastcgi_param  REQUEST_METHOD     $request_method;
    fastcgi_param  CONTENT_TYPE       $content_type;
    fastcgi_param  CONTENT_LENGTH     $content_length;

    fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;

    fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
    fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;

    fastcgi_param  REMOTE_ADDR        $remote_addr;
  }
}
{% endhighlight %}
***server_name***可以理解我们部署的host，如果有多个url指向同一地址的话，只需要输入多个host，以空格隔开即可  
***root***表示我们部署的文件路径，DocumnetPath  

注意下面这段配置
{% highlight sh%}
location / {
    index index.php;
    rewrite ^/(.*)$ /index.php?__path__=/$1 last;
  }
{% endhighlight %}
表示当我们在进入**phabricator.powell.com**时，我们的index文件为**/alidata/www/phabricator/webroot**路径下的**index.php**,也就是我们的默认入口文件；而大括弧内的**rewrite**则表示我们的重写规则，会将**phabricator.powell.com/abc/def/ghi/mn**重写为**phabricator.powell.com/index.php?__path__=abc/def/ghi/mn**  
**location /index.php**这个部分则配置了一些关于fastCGI的东西。由于nginx默认不解析PHP代码，所以我们需要fastCGI的支持，关于fastCGI在后面我会着重介绍下。

### 配置hosts或者解析规则
如果我们是本地开发而并没有真实的域名的话，就如笔者这里**phabricator.powell.com**，我们则需要在访问机器上将该域名指向到服务器
{% highlight sh%}
sudo echo "127.0.0.1 phabricator.powell.com" >> /etc/hosts
{% endhighlight %}
当然如果我们是现有域名的话，只需要在域名服务商处添加解析规则即可，一般10分钟左右生效，如果比较着急也可以先手动指定host  

重启nginx
{% highlight sh%}
sudo service nginx restart
{% endhighlight %}
重启php-fpm(FastCGI Process Manager：FastCGI进程管理器)
{% highlight sh%}
sudo service php-fpm restart
{% endhighlight %}

至此我们的nginx vhost配置已经完成






