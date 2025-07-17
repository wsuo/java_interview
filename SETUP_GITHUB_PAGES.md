# 🚀 GitHub Pages 快速设置指南

## ✅ 第1步：代码已推送完成
你的代码已经成功推送到GitHub！

## 🛠️ 第2步：启用GitHub Pages

### 在GitHub网站上操作：

1. **打开你的仓库**
   - 访问：https://github.com/wsuo/java_interview

2. **进入Settings**
   - 点击仓库页面顶部的 `Settings` 标签

3. **找到Pages设置**
   - 在左侧菜单中滚动找到 `Pages` 选项
   - 点击进入Pages设置页面

4. **配置部署源**
   - 在 `Source` 部分选择 `Deploy from a branch`
   - Branch 选择 `main`
   - Folder 保持 `/ (root)` 不变
   - 点击 `Save` 按钮

5. **等待部署**
   - 部署通常需要1-3分钟
   - 你会看到一个绿色的 ✅ 提示

## 🌐 第3步：访问你的网站

### 网站地址：
```
https://wsuo.github.io/java_interview/
```

### 📱 移动端访问特性：
- ✅ 完美适配手机屏幕
- ✅ 支持搜索功能
- ✅ 代码块可以复制
- ✅ 左侧菜单可收起
- ✅ PWA支持，可添加到桌面
- ✅ 支持离线阅读

## 🔧 第4步：本地预览（可选）

如果你想在本地预览：

```bash
# 进入项目目录
cd java_interview

# 使用Python启动服务器
python serve.py

# 或者使用Node.js（需要先安装docsify-cli）
npm install -g docsify-cli
docsify serve .
```

然后访问 `http://localhost:3000`

## 📝 第5步：自定义配置（可选）

### 修改网站标题
编辑 `index.html` 中的 `name` 字段：
```javascript
window.$docsify = {
  name: '你的网站名称',
  // ...其他配置
}
```

### 修改仓库链接
编辑 `index.html` 中的 `repo` 字段：
```javascript
window.$docsify = {
  repo: 'https://github.com/你的用户名/仓库名',
  // ...其他配置
}
```

### 添加Google Analytics（可选）
在 `index.html` 的 `<head>` 部分添加：
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

## 🎉 完成！

现在你就有了一个专业的在线Java面试知识库！

**主要优势：**
- 📱 手机随时访问，地铁上也能学习
- 🔍 快速搜索，找知识点很方便
- 📄 清晰的文档结构
- 🎨 专业的UI设计
- 🚀 访问速度快
- 💾 支持离线阅读

**使用建议：**
1. 将网站添加到手机桌面
2. 利用搜索功能快速定位知识点
3. 面试前可以离线浏览
4. 随时更新内容，推送后自动发布

祝你面试顺利！🍀