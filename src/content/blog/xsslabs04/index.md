---
title: "Xss-labs通关全解&&XSS笔记04"
publishDate: 2025-12-26 18:37:00
description: "分析以及笔记"
tags: ["XSS", "笔记", "Web"]
language: "中文"

---

## 第十六关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level17.php?arg01=a&arg02=b"; 
}
</script>
<title>欢迎来到level16</title>
</head>
<body>
<h1 align=center>欢迎来到level16</h1>
<?php 
ini_set("display_errors", 0);
$str = strtolower($_GET["keyword"]);
$str2=str_replace("script"," ",$str);
$str3=str_replace(" "," ",$str2);
$str4=str_replace("/"," ",$str3);
$str5=str_replace("    "," ",$str4);
echo "<center>".$str5."</center>";
?>
<center><img src=level16.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str5)."</h3>";
?>
</body>
</html>


```

1.从源码分析可以得到 这一关无法使用大小写绕过，无法使用`script` 甚至于` `（空格）都被转译成了 `&nbsp`

```php
$str = strtolower($_GET["keyword"]);
$str2=str_replace("script"," ",$str);
$str3=str_replace(" "," ",$str2);
$str4=str_replace("/"," ",$str3);
$str5=str_replace("    "," ",$str4);
```

既然不让使用`<script>` 那么不妨使用 `<img>`、`<svg>`、`<body>` 等等，但是这中间的空格怎么办呢。

2.解析 HTML 的时候特别宽容。除了**空格（Space, %20）**，它还认其他的“空白字符”作为分隔符。被封了空格，被封了 Tab还有

* **回车符（CR, Carriage Return）** -> URL 编码是 `%0d`

* **换行符（LF, Line Feed）** -> URL 编码是 `%0a`

所以，我们的思路就是：**用 `%0a` 或者 `%0d` 代替空格，把标签属性隔开。**

3.那么我们就可以构造最经典的`<img>`

```html
keyword=<img%0asrc=x%0aonerror=alert(1)>
```

或者还可以使用[标签逃逸](#标签逃逸) 中的其他办法

### 空格绕过全家桶

| **字符大名**       | **URL编码** | **在 Level 16 的下场** | **实战地位**     | **原理解析**                                                   |
| -------------- | --------- | ------------------ | ------------ | ---------------------------------------------------------- |
| **空格 (Space)** | `%20`     | **被干死了**           | 炮灰           | 最标准的空格。正因为太标准，所有 WAF 第一刀砍的绝对是它。这题直接 `str_replace` 换成空，没法用。 |
| **制表符 (Tab)**  | `%09`     | **被干死了**           | 替补           | 也就是键盘上的 Tab 键。很多懒狗开发只过滤 `%20` 忘了 `%09`，但这题作者把这也堵上了。        |
| **换行符 (LF)**   | `%0a`     | **✅ 存活 (MVP)**     | **隐形刺客**     | Linux 系统的换行。浏览器把它当空格认，但 PHP 代码里没过滤它。**这就是本题解法**。           |
| **回车符 (CR)**   | `%0d`     | **✅ 存活 (MVP)**     | **备胎**       | Windows 系统的回车前一半。跟 `%0a` 是一路货色，这题也能用它过。                    |
| **斜杠 (Slash)** | `/`       | **被干死了**           | **God Like** | 神技。`<img/src=x>` 这种写法完全不需要任何空白符。可惜这题防住了，不然这才是最骚的。          |
| **换页符 (FF)**   | `%0c`     | 看运气                | 神经病          | 极冷门字符。有的老旧浏览器或者特定环境才认。实在没办法了可以拿出来碰碰运气。                     |
| **垂直制表符**      | `%0b`     | 看运气                | 神经病二号        | 同上，属于死马当活马医的选项。                                            |

## 第十七关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！"); 
}
</script>
<title>欢迎来到level17</title>
</head>
<body>
<h1 align=center>欢迎来到level17</h1>
<?php
ini_set("display_errors", 0);
echo "<embed src=xsf01.swf?".htmlspecialchars($_GET["arg01"])."=".htmlspecialchars($_GET["arg02"])." width=100% heigth=100%>";
?>
<h2 align=center>成功后，<a href=level18.php?arg01=a&arg02=b>点我进入下一关</a></h2>
</body>
</html>
```

1.它考的不是 Flash 漏洞，而是**HTML 解析的一个经典逻辑**，所以flash坏了也没事

2.核心代码

```php
echo "<embed src=xsf01.swf?".htmlspecialchars($_GET["arg01"])."=".htmlspecialchars($_GET["arg02"])." width=100% heigth=100%>";
```

坏消息，经过过滤了，好多都不能使用了，好消息，`src=xsf01.swf?...`。 **没有引号！没有引号！没有引号！**

在 HTML 语法里，如果一个属性的值没有用引号包裹，浏览器怎么知道这个属性值在哪里结束？ **答案是：遇到空格就结束。**

3.那么我们就可以利用`arg02` 来构造

```
level17.php?arg01=a&arg02=b%20onmouseover=alert(1)
```

我们让 `arg01` 随便填个 `a`。我们在 `arg02` 里填：`b onmouseover=alert(1)`

## 第十八关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level19.php?arg01=a&arg02=b"; 
}
</script>
<title>欢迎来到level18</title>
</head>
<body>
<h1 align=center>欢迎来到level18</h1>
<?php
ini_set("display_errors", 0);
echo "<embed src=xsf02.swf?".htmlspecialchars($_GET["arg01"])."=".htmlspecialchars($_GET["arg02"])." width=100% heigth=100%>";
?>
</body>
</html>

```

1.和十七没有任何区别，payload同第十七

```
level18.php?arg01=a&arg02=b%20onmouseover=alert(1)
```

## 第十九关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level20.php?arg01=a&arg02=b"; 
}
</script>
<title>欢迎来到level19</title>
</head>
<body>
<h1 align=center>欢迎来到level19</h1>
<?php
ini_set("display_errors", 0);
echo '<embed src="xsf03.swf?'.htmlspecialchars($_GET["arg01"])."=".htmlspecialchars($_GET["arg02"]).'" width=100% heigth=100%>';
?>
</body>
</html>


```

1.这一关把上面最大的问题给修复了 `"` 

这一关的注入就需要使用flash了 但是现代浏览器无法使用flash怎么办 

这里可以用 `Ruffle` 这个浏览器插件，非常简单（但是有时候可能bug修的太好了导致flash的漏洞它用不了）

或者去找一个专门的`尸体浏览器`

2.想要解析`xsf03.swf` 我们就必须用到 另一个工具 `JPEXS` `sIFR`文件太长了就不展示了(这里就意思意思了，flash逆向没学明白)

这个 `xsf03.swf` 是个老演员了。它的 ActionScript 逻辑大概是这样的：

1. 它接收一个名为 `version` 的参数。

2. 它把这个参数直接赋值给了一个文本框的 `htmlText` 属性。

3. Flash 的 `htmlText` 支持有限的 HTML 标签，其中就包括 `<a>` 标签（超链接）。

这就好办了，我们只要构造一个带 `javascript:` 伪协议的 `<a>` 标签，传给 `version` 参数，然后在 Flash 里点一下生成的链接，就能触发 JS。

3.我们就可以如下构造出

```
level19.php?arg01=version&arg02=<a href="javascript:alert(1)">快点我拿flag</a>
```

这里利用了浏览器在渲染 HTML 标签时，有一个规则：**在解析 HTML 属性值（比如 src、href、value）时，会自动进行“HTML 实体解码”。** 使用就算转义了也没事。

## 第二十关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level21.php?arg01=a&arg02=b"; 
}
</script>
<title>欢迎来到level20</title>
</head>
<body>
<h1 align=center>欢迎来到level20</h1>
<?php
ini_set("display_errors", 0);
echo '<embed src="xsf04.swf?'.htmlspecialchars($_GET["arg01"])."=".htmlspecialchars($_GET["arg02"]).'" width=100% heigth=100%>';
?>
</body>
</html>


```

1.这个 `xsf04.swf` 是一个名为 **ZeroClipboard** 的组件（早期版本）。它的功能是帮助网页实现“复制到剪贴板”的功能。它内部使用 `ExternalInterface.call` 来调用浏览器的 JavaScript 函数。其 ActionScript 逻辑大致如下（简化版）：

```php
// 获取传入的 id 参数
var id = loaderInfo.parameters.id;
// 调用 JS，构造类似于这样的代码：
ExternalInterface.call("ZeroClipboard.dispatch", id, "mouseOut", null);
```

或者在生成的 JS 中，它会被拼接到一个字符串里： `try { __flash__toXML(ZeroClipboard.dispatch("你的ID")); } catch (e) { ... }`

**漏洞点：** Flash 在拼接这段 JS 代码时，没有正确处理我们传入的 `id` 参数。如果我们传入特定的字符，就能闭合掉原本的引号和括号，插入我们自己的 JS 代码。

3.我们需要做两件事：

1. **参数名 (`arg01`)**：必须是 Flash 预定义的参数名，这里是 `id`。

2. **参数值 (`arg02`)**：需要闭合 Flash 构造的 JS 语句。

**构造 Payload：** 通常这关的标准 Payload 是 `\"))alert(1)//`。

* `\"`：这里的反斜杠通常是为了转义 Flash 内部可能存在的转义，或者配合双引号闭合字符串。

* `))`：闭合前面的函数调用括号。

* `alert(1)`：执行我们的代码。

* `//`：注释掉后面原本的 JS 代码，防止语法错误。

```php
level20.php?arg01=id&arg02=\"))alert(1)//
```

遗憾的是不知道为什么我就是触发不了，下面附上大佬的解析

> [XSS LABS - Level 20 过关思路_xss-lab20-CSDN博客](https://blog.csdn.net/m0_73360524/article/details/141619635)


