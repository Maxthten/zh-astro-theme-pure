---
title: "踩坑记录：我是如何把这个双语博客折腾出来的"
publishDate: 2025-12-22 22:41:00
description: "杂谈&&博客艰辛"
tags: ["Astro", "Cloudflare", "踩坑", "折腾"]
language: "中文"
---

如果你看到了这篇文章，说明我已经成功了。🎉

看着现在这个运行流畅、中英文切换丝滑的博客，你可能想象不到，就在几个小时前，我还在对着满屏的报错日志怀疑人生。

这篇文章不聊高深的技术，纯粹记录一下我是如何把 **Astro Pure** 主题魔改成一个**部署在 Cloudflare 上的双语（中/英）独立站点**的。如果你也想搞一个类似的双语博客。

## 1. 架构设想：一分为二

最开始我就决定了，不要搞复杂的国际化路由（i18n routing），太麻烦。我的需求很简单粗暴：

- **中文站**：`zh.maxtonniu.com`
- **英文站**：`en.maxtonniu.com`
- **部署**：Cloudflare Pages（免费又快，还不用备案）

于是我建立了两个 GitHub 仓库，分别对应两个站点。听起来很完美，对吧？噩梦开始了。

## 2. 第一个坑：同构路由跳转 (Magic Switch)

有了两个域名，最大的问题是：**我在中文站看关于页 `/about`，点了 "English" 按钮，怎么自动跳到英文站的 `/about`，而不是傻傻地跳回英文首页？**

由于 Astro Pure 主题的 Header 组件藏得很深（或者说为了不破坏源码），我不想去改组件代码。

**解决方案：暗号拦截法**

我在 `site.config.ts` 里把切换按钮的链接填成了一个“暗号”：`#switch-lang`。

然后，我在全局布局文件 `BaseLayout.astro` 的底部注入了一段魔法脚本：

```javascript
// 脚本逻辑：找到暗号链接，自动拼接当前路径
const targetDomain = '[https://en.maxtonniu.com](https://en.maxtonniu.com)'; // 英文站填这个

document.addEventListener('DOMContentLoaded', () => {
  const langBtn = document.querySelector('a[href="#switch-lang"]');
  if (langBtn) {
    // 自动把 #switch-lang 替换成 [https://en.maxtonniu.com/当前路径](https://en.maxtonniu.com/当前路径)
    langBtn.href = targetDomain + window.location.pathname;
  }
});
```

这样，无论我在哪个页面，点击按钮都能精准穿越到另一个平行宇宙。

## 3.第二个坑：模式切换

解决了跳转，又来了个新问题：浏览器的 `localStorage` 是按域名隔离的。

我在中文站开了“深色模式”，一跳到英文站，瞎了——英文站还是默认的“亮色模式”。这体验太割裂了。

**解决方案：URL 传参接力**

我修改了上面的脚本，在跳转时偷偷在 URL 屁股后面带了个参数：`?sync_theme=dark`。

然后在页面加载的最早期（`<head>` 标签里），写了一段脚本来“接球”：

```javascript
// 检查 URL 有没有带主题参数
const params = new URLSearchParams(window.location.search);
const theme = params.get('sync_theme');

if (theme) {
  // 强制写入本地存储
  localStorage.setItem('theme', theme);
  // 立即给 html 标签加上 dark 类，防止闪烁
  document.documentElement.classList.add('dark');
}
```

现在，主题就像接力棒一样在两个域名之间传递，丝般顺滑。

## 4.第三个坑：“幽灵文件”

本地运行 `npm run dev` 一切正常，一推送到 Cloudflare 就报错：

> `[vite]: Rollup failed to resolve import "@/assets/tools/zotero.svg?raw"`

原因是我删掉了主题自带的示例图片 `zotero.svg`，但在代码里忘了删引用。 **教训**：本地开发环境是“懒加载”的，不报错不代表没问题；线上构建是“地毯式搜索”，眼里容不得沙子。删了资源，一定要记得删引用！

## 5.最终 Boss：灰色的 404

经历了九九八十一难，终于显示 `Success: Uploaded`。我激动的打开网址，结果：

> **404 Not Found** (Astro 默认的灰色页面)

但我明明上传了代码啊！为什么首页是空的？

排查了半天，发现控制台日志里有一行刺眼的信息： `[build] adapter: @astrojs/vercel`

**破案了！** 我复制的配置代码里，居然残留着 `adapter: vercel()`。这相当于我把代码打包成了 **Vercel 专用** 的格式（Serverless Functions），然后硬塞给了 **Cloudflare Pages**。Cloudflare 看不懂这些代码，找不到 `index.html`，只能两手一摊。

**终极修复：** 修改 `astro.config.mjs`，删掉 Vercel 适配器，回归纯真：

```javascript
export default defineConfig({
  // 删掉 adapter: vercel()
  output: 'static', // 告诉 Astro，我要纯纯的静态网页！
  // ...
});
```

推送到 GitHub，Cloudflare 重新构建，绿色的 `Success` 再次亮起。这一次，页面出现了。



> 以上由Gemini代本人亲情书写
