# Bzone Blog

这是一个基于 `Hugo + Blowfish` 的个人博客。

## 日常维护入口

- 站点基础配置：`hugo.toml`
- 主题和展示配置：`config/_default/params.toml`
- 菜单：`config/_default/menus.toml`
- 文章：`content/posts/`
- 关于页：`content/about.md`
- 写作模板：`archetypes/posts.md`
- 自定义样式：`assets/css/extended.css`
- 首页布局：`layouts/partials/home/card.html`

## 常用命令

```bash
hugo server -D
hugo new posts/my-new-post.md
hugo
```

## 推荐写作流程

1. 新建文章：`hugo new posts/my-new-post.md`
2. 按 `archetypes/posts.md` 填写标题、摘要、标签、分类
3. 写完后把 `draft` 改成 `false`
4. 本地预览确认无误后再提交

## 结构说明

- `hugo.toml` 只保留 Hugo 运行级配置，避免和主题参数混在一起
- `params.toml` 统一维护作者信息、首页信息和展示开关
- 菜单单独放在 `menus.toml`，改导航时不需要翻主配置
- 文章模板单独维护，后续写文章不需要每次手敲 front matter
