# MisakaTao's Blog

记录技术与生活 ☕️

基于 [Hugo](https://gohugo.io/) 构建的个人博客，使用 [Stack](https://github.com/CaiJimmy/hugo-theme-stack) 主题。

**在线访问**: [misakatao.com](https://misakatao.com/)

## 内容

- **iOS 开发** — Objective-C Runtime、Category、CocoaPods 等技术文章
- **随笔** — 生活感悟与散文

## 特性

- Giscus 评论系统（基于 GitHub Discussions）
- 深色 / 浅色主题自动切换
- 全文搜索、标签云、归档
- RSS 订阅

## 本地运行

```bash
# 克隆仓库（含主题子模块）
git clone --recurse-submodules https://github.com/misakatao/blog.git
cd blog

# 启动开发服务器
hugo server -D
```

访问 `http://localhost:1313` 预览。

## 目录结构

```
├── config/          # Hugo 配置
├── content/posts/   # 博客文章
├── static/          # 静态资源
├── themes/          # Stack 主题（子模块）
└── layouts/         # 自定义布局覆盖
```

## 许可证

博客文章内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可。代码部分采用 [MIT](LICENSE) 许可。
