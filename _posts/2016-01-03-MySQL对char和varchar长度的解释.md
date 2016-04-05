---
layout: post
title: MySQL对char和varchar长度的解释
category: MySQL
tags: [MySQL,数据库]
---

这篇文章纯属实践检验型的，没有技术分析的成分。这段时间在建表的时候，对字符串存取时设置预置大小时产生了一些小得疑问，所以就自己检验了下。要是平时不太注意的同学也可以参考下。  

### 实验
首先我们建立一个数据表
{% highlight sql%}
CREATE TABLE `test` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(5) NOT NULL DEFAULT '',
  `ic_number` varchar(18) NOT NULL DEFAULT '',
  `bank_card` varchar(30) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `name_number` (`name`,`ic_number`)
) ENGINE=InnoDB AUTO_INCREMENT=374 DEFAULT CHARSET=utf8;
{% endhighlight %}
表的结构如下：
{% highlight sql%}
mysql> desc test;
+-----------+------------------+------+-----+---------+----------------+
| Field     | Type             | Null | Key | Default | Extra          |
+-----------+------------------+------+-----+---------+----------------+
| id        | int(11) unsigned | NO   | PRI | NULL    | auto_increment |
| name      | varchar(5)       | NO   | MUL |         |                |
| ic_number | varchar(18)      | NO   |     |         |                |
| bank_card | varchar(30)      | YES  |     | NULL    |                |
+-----------+------------------+------+-----+---------+----------------+
4 rows in set (0.00 sec)
{% endhighlight %}
我们插入如下数据
{% highlight sql%}
INSERT INTO `test` (`id`, `name`, `ic_number`, `bank_card`)
VALUES
    (289, '王晓明', '111111111111111111', '222222222222222222');
{% endhighlight %}
{% highlight sql%}
mysql> INSERT INTO `test` (`id`, `name`, `ic_number`, `bank_card`)
    -> VALUES
    -> (289, '王晓明', '111111111111111111', '222222222222222222');
Query OK, 1 row affected (0.00 sec)
{% endhighlight %}
插入成功，这是我们将我们插入的name修改成，a是王晓明，同样可以插入成功，然而当我们在插入name='ab是王晓明'时
{% highlight sql%}
mysql> INSERT INTO `test` (`id`, `name`, `ic_number`, `bank_card`)
    -> VALUES
    -> (289, 'ab是王晓明', '111111111111111111', '222222222222222222');
ERROR 1406 (22001): Data too long for column 'name' at row 1
{% endhighlight %}
返回数据长度超过了最大长度，这个实验在没有遇到具体场景的时候确实不明白目的是什么，现在我来大致总结下结论

### 结论
<kp>varchar</kp>的长度限制的是插入字符串字符的长度，而不是字节的长度。所以当我们在建表的时候，我们不用去考虑插入的字符串是汉字还是字母或者数字，只需要按照字符来统计就行  
同样，我实验了<kp>char</kp>也得到了同样的结论，需要注意的是<kp>char设置的字符串长度，而varchar设置的字符串最大长度，两者的区别是在对char插入一个不足最大长度的字符串时，后边会补空，而varchar则不会</kp>

最后附上我从w3cschool上看到的几个主流数据库对这些数据类型的解释，共大家参考

### Microsoft access
![PhpStorm配置选项]({{ site.res }}/images/{{page.title}}/Snip20160103_3.png)

### MySQL
![PhpStorm配置选项]({{ site.res }}/images/{{page.title}}/Snip20160103_4.png)
![PhpStorm配置选项]({{ site.res }}/images/{{page.title}}/Snip20160103_5.png)
![PhpStorm配置选项]({{ site.res }}/images/{{page.title}}/Snip20160103_6.png)

### SQL Server
![PhpStorm配置选项]({{ site.res }}/images/{{page.title}}/Snip20160103_7.png)
![PhpStorm配置选项]({{ site.res }}/images/{{page.title}}/Snip20160103_8.png)
![PhpStorm配置选项]({{ site.res }}/images/{{page.title}}/Snip20160103_9.png)
![PhpStorm配置选项]({{ site.res }}/images/{{page.title}}/Snip20160103_10.png)








