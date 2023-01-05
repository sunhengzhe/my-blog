# Blog

使用 [Hugo](https://gohugo.io/getting-started/quick-start/) 搭建

## 搭建过程

Hugo 基本使用：[官方文档](https://gohugo.io/getting-started/quick-start/)

接入评论：[valine 教程](https://www.smslit.top/2018/07/08/hugo-valine/)、[valine 官方文档](https://valine.js.org/)

代码高亮：[官方文档](https://gohugo.io/getting-started/configuration-markup#highlight)、[所有主题](https://xyproto.github.io/splash/docs/all.html)

主题修改：[hugo 主题修改](https://blog.rool.me/post/hugo%E4%B8%BB%E9%A2%98%E4%BF%AE%E6%94%B9/)

## 步骤

防止自己忘了 😅

```shell
# 本地开发
1. hugo new moments/new-article.md
2. hugo server # preview
3. git add/commit/push

# 部署
1. git pull
2. hugo
3. nginx -s reload
```