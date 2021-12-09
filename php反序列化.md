# php序列化与反序列化

## serialize() 序列化函数

### 1. 对于类变量 public 、protected 、private 序列化的区别

**在 url 传参时注意区分！！！**

```php+HTML
<?php
class myClass{
    public     $first  = 123  ;
    protected  $second = "456";
    private    $third  = "789";
}
echo serialize(new myClass());
```

有如下结果

```php+HTML
O:7:"myClass":3:{s:5:"first";i:123;s:9:"*second";s:3:"456";s:14:"myClassthird";s:3:"789";}
```

由上图实验发现，区别只在于对变量名添加了标记：

**public**       无标记，变量名不变，长度不变 `s:5:"first";i:123`
**protected** 在变量名前添加标记 \00*\00 ，长度加3 `s:9:"\00*\00second";s:"456"`
**private**      在变量名前添加标记 \00(classname)\00 ，长度 + 2 + 类名长度 `s:14:"myClassthird";s:3:"789"`

### 2. __sleep() 函数

在调用 `serialize()` 函数前执行

## unserialize() 反序列化函数

```php+HTML
<?php
class myClass{
    public     $first  = 123;
    protected  $second = "456";
    private    $third  = "789";
}
$str = 'O:7:"myClass":3:{s:5:"first";i:123;s:9:"*second";s:3:"456";s:14:"myClassthird";s:3:"789";}';
$obj = unserialize($str);
```

### 1. __wakup() 函数

在调用 `unserialize()` 函数前执行

### 2. 反序列化漏洞

当序列化字符串表示对象属性个数的值大于或小于（依据 PHP 版本而不同）真实个数的属性时就会跳过 `__wakeup()` 的执行

例如对于传入下面的序列化字符串

```php+HTML
O:7:"myClass":3:{s:5:"first";i:123;s:9:"*second";s:3:"456";s:14:"myClassthird";s:3:"789";}
```
旧版本可更改 `O:7:"myClass":3:` 为 `O:7:"myClass":4:`
新版本可更改 `O:7:"myClass":3:` 为 `O:7:"myClass":2:`

实现绕过 `__wakeup()` 的执行

## 题目1 [网鼎杯 2020 青龙组]AreUSerialz1

buuoj.cn

```php+HTML
<?php

include("flag.php");

highlight_file(__FILE__);

class FileHandler {

    protected $op;
    protected $filename;
    protected $content;

    function __construct() {
        $op = "1";
        $filename = "/tmp/tmpfile";
        $content = "Hello World!";
        $this->process();
    }

    public function process() {
        if($this->op == "1") {
            $this->write();
        } else if($this->op == "2") {
            $res = $this->read();
            $this->output($res);
        } else {
            $this->output("Bad Hacker!");
        }
    }

    private function write() {
        if(isset($this->filename) && isset($this->content)) {
            if(strlen((string)$this->content) > 100) {
                $this->output("Too long!");
                die();
            }
            $res = file_put_contents($this->filename, $this->content);
            if($res) $this->output("Successful!");
            else $this->output("Failed!");
        } else {
            $this->output("Failed!");
        }
    }

    private function read() {
        $res = "";
        if(isset($this->filename)) {
            $res = file_get_contents($this->filename);
        }
        return $res;
    }

    private function output($s) {
        echo "[Result]: <br>";
        echo $s;
    }

    function __destruct() {
        if($this->op === "2")
            $this->op = "1";
        $this->content = "";
        $this->process();
    }

}

function is_valid($s) {
    for($i = 0; $i < strlen($s); $i++)
        if(!(ord($s[$i]) >= 32 && ord($s[$i]) <= 125))
            return false;
    return true;
}

if(isset($_GET{'str'})) {

    $str = (string)$_GET['str'];
    if(is_valid($str)) {
        $obj = unserialize($str);
    }

}
```

### 题解

#### 1. is_valid()

说明：`ord()` 将字符转化为 ASCII 码，要求我们传入的 str 的每个字符的 ASCII 值在 32 和 125 之间。因为 protected 属性在序列化之后会出现不可见字符 \00*\00 ，不符合上面的要求。

绕过方法：因为 PHP7.1 以上的版本对属性类型不敏感，所以可以将属性改为 public ，public 属性序列化不会出现不可见字符

#### 2. unserialize()

执行反序列化函数时不会调用 `__construct()` 方法，无视其中的内容

观察 `__deconstruct()` 方法，发现需要在调用 `process()` 方法时进入 `read()` 方法，读取 `flag.php` 文件，返回flag

为了进入 `read()` 方法，在 `process()` 中要使 `$this->op == "2"` ，而 `__deconstruct()` 中不能有 `$this->op === "2"` ，利用 PHP 的强弱类型比较不同，传入 `$this->op == 2` ，实现绕过

#### 3. 构造 payload

注意 payload 中使用了 public 属性的 op 实现绕过 `is_valid()`

```php+HTML
O:11:"FileHandler":3:{s:2:"op";i:2;s:11:"*filename";s:8:"flag.php";s:10:"*content";s:0:"";}
```

F12 观察 PHP 源码得到 flag

## Extra

参考资料

https://blog.csdn.net/weixin_45844670/article/details/108171963

https://blog.csdn.net/weixin_45844670/article/details/108934194

https://www.php.cn/php-weizijiaocheng-427052.html