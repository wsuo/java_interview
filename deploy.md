# 部署指南

## 🚀 GitHub Pages部署

### 1. 推送代码到GitHub
```bash
# 添加所有文件
git add .

# 提交更改
git commit -m "Setup docsify documentation site"

# 推送到GitHub
git push origin main
```

### 2. 启用GitHub Pages
1. 打开GitHub仓库页面
2. 点击 `Settings` 标签
3. 滚动到 `Pages` 部分
4. 在 `Source` 下选择 `Deploy from a branch`
5. 选择 `main` 分支
6. 点击 `Save`

### 3. 访问网站
- 几分钟后，你的网站将在以下地址可用：
- `https://wshuo.github.io/java_interview/`

## 📱 移动端访问

### 手机浏览器
直接访问GitHub Pages地址即可，网站已经针对移动端优化。

### 添加到桌面
1. 在手机浏览器中打开网站
2. 点击浏览器菜单中的"添加到主屏幕"
3. 设置图标名称为"Java面试"
4. 点击添加

### PWA支持
网站支持PWA功能，可以：
- 离线访问已浏览的内容
- 像原生APP一样运行
- 自动更新内容

## 🛠️ 本地开发

### 方式1：使用Python（推荐）
```bash
# 运行内置服务器
python serve.py

# 访问 http://localhost:3000
```

### 方式2：使用Docsify CLI
```bash
# 安装docsify-cli
npm install -g docsify-cli

# 启动服务
docsify serve .

# 访问 http://localhost:3000
```

### 方式3：使用Node.js
```bash
# 安装依赖
npm install

# 启动开发服务器
npm run serve
```

## 🔧 自定义域名（可选）

如果你有自定义域名，可以：

1. 在仓库根目录创建 `CNAME` 文件
2. 在文件中写入你的域名，如：`interview.example.com`
3. 在域名提供商处设置CNAME记录指向：`wshuo.github.io`

## 📊 访问统计

可以添加Google Analytics或其他统计工具：

```html
<!-- 在 index.html 中添加 -->
<script async src="https://www.googletagmanager.com/gtag/js?id=GA_MEASUREMENT_ID"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'GA_MEASUREMENT_ID');
</script>
```

## 🔄 自动部署

可以设置GitHub Actions自动部署：

```yaml
# .github/workflows/deploy.yml
name: Deploy to GitHub Pages

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Deploy to GitHub Pages
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./
```

## 🎯 使用建议

### 内容更新
1. 直接在GitHub上编辑markdown文件
2. 提交更改后，GitHub Pages会自动更新
3. 大约1-2分钟后生效

### 性能优化
- 图片使用压缩格式
- 代码块避免过长
- 合理使用缓存

### SEO优化
- 每个页面都有标题和描述
- 使用语义化的HTML结构
- 添加适当的meta标签

## 📞 故障排除

### 常见问题
1. **网站无法访问**：检查GitHub Pages设置是否正确
2. **样式不正常**：清除浏览器缓存
3. **内容不更新**：等待1-2分钟或强制刷新

### 调试方法
1. 查看GitHub Pages的部署日志
2. 使用浏览器开发者工具检查错误
3. 在本地测试后再推送

## 🎉 完成！

现在你就有了一个专业的在线Java面试知识库！

**访问地址：** https://wshuo.github.io/java_interview/

📱 在手机上也能完美阅读，支持搜索、代码复制等功能。