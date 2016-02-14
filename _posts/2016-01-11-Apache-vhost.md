---
layout: post
title: Apache vhost
category: Apache vhost web
tags: [Apache]
---

刚开始开发web的时候，我总是将项目直接部署至Apache的DocumentRoot目录，这样我只需要将自己的域名指到localhost即可。但是随着项目开发的进行，我需要同时开发前端和后台时，一个致命的问题就暴露了出来，<kp>我需要在每次切换项目的时候，替换DocumentRoot下部署的文件</kp>，这样反而让我的开发效率降低了不少。于是乎，最终选择了Apache的vhost方式  

### 打开Apache对vhost的支持
编辑Apache配置文件,在macos上的默认路径为
{% highlight sh%}
sudo vim /etc/apache2/httpd.conf
{% endhighlight %}
找到下面这行代码，将前边的注释去掉
{% highlight sh%}
498 # Virtual hosts
499 Include /private/etc/apache2/extra/httpd-vhosts.conf
{% endhighlight %}
重启Apache
{% highlight sh%}
sudo apachectl restart
{% endhighlight %}

### 编辑httpd-vhosts.conf文件
在macos上的默认路径为
{% highlight sh%}
sudo vim /etc/apache2/extra/httpd-vhosts.conf
{% endhighlight %}
加入如下配置
{% highlight sh%}
 23 <VirtualHost *:80>
 24     ServerAdmin codecooker@outlook.com
 25     DocumentRoot "/Library/WebServer/Documents/abc"
 26     ServerName abc.local.com
 27     ServerAlias abc.local.com
 28     ErrorLog "/private/var/log/apache2/abc.local.com-error_log"
 29     CustomLog "/private/var/log/apache2/abc.local.com-access_log" common
 30 </VirtualHost>
{% endhighlight %}
其中DocumentRoot行表示我们要配置的虚拟主机对应的DocumentRoot路径.ServerName表示我们的域名.ServerAlias表示别称.最后两行是log路径。  
截止现在其实我们已经添加了一个vhost,这时我们只需重启Apache，然后用abc.local.com访问我们的项目即可<kp>在配置完Apache后，我们需要将abc.local.com这个域名指定到localhost，即在hosts文件内添加映射</kp>
{% highlight sh%}
abc.local.com 127.0.0.1
{% endhighlight %}

### 如何快速的配置
按照上述配置的话，我们每次配置一个目录就需要添加对应vhost配置，不是很方便。我们可以在httpd-vhosts.conf文件中添加
{% highlight sh%}
rewriteEngine on
RewriteMap lowercase int:tolower
RewriteMap vhost txt:/etc/apache2/extra/vhost.map
RewriteCond ${lowercase:%{SERVER_NAME}} ^(.+)$
RewriteCond ${vhost:%1} ^(/.*)$
RewriteRule ^/(.*)$ %1/$1
{% endhighlight %}
然后在/etc/apache2/extra/目录下创建vhost.map内容格式如下
{% highlight sh%}
abc.local.com /Library/WebServer/Documents/abc
{% endhighlight %}
这样就可以用abc.local.com
以后添加新的vhost只需要在vhost.map添加对应关系即可




