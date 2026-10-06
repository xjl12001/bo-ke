# 网站自定义配置指南

## 功能概述

管理员可以在后台自定义网站的基本信息和外观，包括：
- 网站名称
- 网站图标（Logo）
- 页面头部自定义代码（插入到 `<head>` 标签中）
- 页面底部自定义代码（插入到 `</body>` 标签前）

## 使用步骤

### 1. 访问网站设置

1. 以管理员身份登录后台
2. 点击左侧菜单的"系统设置"
3. 选择"网站设置"选项卡

### 2. 修改网站名称

在"网站名称"输入框中输入您想要的网站名称，例如：
- 我的个人博客
- 技术分享站
- XX的创作空间

网站名称会显示在：
- 浏览器标签页标题
- 前台导航栏Logo位置
- 后台侧边栏顶部

### 3. 上传网站图标

1. 点击"网站图标"下方的上传区域
2. 选择一张图片文件（支持PNG、JPG、GIF、SVG等格式）
3. 文件大小不超过1MB
4. 推荐尺寸：512x512px（正方形）

网站图标会显示在：
- 浏览器标签页（favicon）
- 前台导航栏Logo位置
- 后台侧边栏顶部

### 4. 添加自定义代码

#### 页面头部代码（Head Code）

插入到所有页面的 `<head>` 标签中，常见用途：

**示例1：添加Google Analytics统计代码**
```html
<!-- Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=GA_MEASUREMENT_ID"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'GA_MEASUREMENT_ID');
</script>
```

**示例2：添加自定义CSS样式**
```html
<style>
  /* 自定义全局样式 */
  body {
    font-family: 'Microsoft YaHei', sans-serif;
  }
  
  /* 自定义链接颜色 */
  a {
    color: #0066cc;
  }
</style>
```

**示例3：添加百度统计**
```html
<script>
var _hmt = _hmt || [];
(function() {
  var hm = document.createElement("script");
  hm.src = "https://hm.baidu.com/hm.js?YOUR_SITE_ID";
  var s = document.getElementsByTagName("script")[0]; 
  s.parentNode.insertBefore(hm, s);
})();
</script>
```

#### 页面底部代码（Footer Code）

插入到所有页面的 `</body>` 标签前，常见用途：

**示例1：添加客服聊天插件**
```html
<!-- 客服插件 -->
<script>
  (function() {
    var script = document.createElement('script');
    script.src = 'https://example.com/chat-widget.js';
    document.body.appendChild(script);
  })();
</script>
```

**示例2：添加返回顶部按钮**
```html
<button id="back-to-top" style="position: fixed; bottom: 20px; right: 20px; display: none; padding: 10px 15px; background: #333; color: #fff; border: none; border-radius: 5px; cursor: pointer;">
  ↑ 顶部
</button>
<script>
  window.addEventListener('scroll', function() {
    var btn = document.getElementById('back-to-top');
    if (window.scrollY > 300) {
      btn.style.display = 'block';
    } else {
      btn.style.display = 'none';
    }
  });
  
  document.getElementById('back-to-top').addEventListener('click', function() {
    window.scrollTo({ top: 0, behavior: 'smooth' });
  });
</script>
```

**示例3：添加页面加载时间统计**
```html
<script>
  window.addEventListener('load', function() {
    var loadTime = window.performance.timing.domContentLoadedEventEnd - window.performance.timing.navigationStart;
    console.log('页面加载时间：' + loadTime + 'ms');
  });
</script>
```

### 5. 保存设置

1. 填写完所有需要修改的内容后，点击"保存网站设置"按钮
2. 等待保存成功提示
3. 页面会自动刷新以应用新设置

## 注意事项

### 安全性
- ⚠️ 自定义代码会在所有页面执行，请确保代码来源可信
- ⚠️ 不要插入恶意代码或未经验证的第三方脚本
- ⚠️ 建议在测试环境先验证代码效果

### 兼容性
- ✅ 支持标准HTML、CSS、JavaScript代码
- ✅ 支持内联样式和脚本
- ✅ 支持外部资源引用（CDN链接等）

### 性能
- 💡 避免插入过大的脚本文件
- 💡 外部资源建议使用CDN加速
- 💡 统计代码建议使用异步加载（async）

### 调试
如果自定义代码不生效：
1. 检查浏览器控制台是否有错误信息
2. 确认代码语法是否正确
3. 清除浏览器缓存后重新访问
4. 使用浏览器开发者工具检查代码是否正确插入

## 常见问题

### Q: 修改后没有立即生效？
A: 保存后需要刷新页面。如果仍未生效，请清除浏览器缓存。

### Q: 可以插入多段代码吗？
A: 可以，在同一个文本框中插入多段代码即可，系统会按顺序执行。

### Q: 代码会影响所有页面吗？
A: 是的，头部和底部代码会在所有前台和后台页面生效。

### Q: 如何删除自定义代码？
A: 清空对应的文本框内容，然后保存即可。

### Q: 支持哪些统计工具？
A: 支持所有基于JavaScript的统计工具，如Google Analytics、百度统计、友盟统计等。

## 技术实现

### 数据存储
- 网站设置存储在 `site_settings` 表中
- 每个网站只有一条配置记录
- 图标文件存储在 Supabase Storage 的 `post-images` bucket

### 代码注入机制
- 头部代码通过 `react-helmet` 注入到 `<head>` 标签
- 底部代码通过 DOM 操作插入到 `<body>` 标签前
- 使用 React Context 管理全局网站设置

### 权限控制
- 只有管理员可以修改网站设置
- 所有用户都可以读取网站设置（用于显示网站名称和图标）
