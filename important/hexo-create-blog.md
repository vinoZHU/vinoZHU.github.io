---
title: hexo搭建个人博客
date: 2016-05-16 11:21:13
tags: hexo
categories: hexo

---
不得不说，hexo给程序猿提供了一种高逼格的写作方式。配置也比较方便，而且目录、Rss和sitemap都是自动生成，你只需花几个小时就能掌握用hexo来编写博客。

## 安装hexo
安装hexo之前请确保机器上已正确安装、配置了git,node.js,如果出现npm安装太慢，请更换npm源:

```
npm config set registry https://registry.npm.taobao.org
npm info underscore （如果上面配置正确这个命令会有字符串response）
 ```

- 安装

```
mkdir hexoblog  #创建一个文件夹
cd hexoblog
npm install -g hexo-cli
npm install hexo --save
```

- 创建博客

```
hexo init
```

- 生成静态站点文件

```
hexo g
```

- 运行服务器

```
hexo s
```

启动成功的话浏览器输入`http://localhost:4000/`
应该能看到默认的页面
- 安装hexo插件：自动生成sitemap,Rss，部署到git等，建议安装

```
npm install hexo-generator-index --save
npm install hexo-generator-archive --save
npm install hexo-generator-category --save
npm install hexo-generator-tag --save
npm install hexo-server --save
npm install hexo-deployer-git --save
npm install hexo-deployer-heroku --save
npm install hexo-deployer-rsync --save
npm install hexo-deployer-openshift --save
npm install hexo-renderer-marked@0.2 --save
npm install hexo-renderer-stylus@0.2 --save
npm install hexo-generator-feed@1 --save
npm install hexo-generator-sitemap@1 --save
```

## 部署到github
部署到Github前需要配置_config.yml文件

添加如下内容：

```
deploy:
  type: git
  repo: https://github.com/vinoZHU/vinoZHU.github.io.git
  branch: master
```

然后输入：

```
hexo d
```

参考：[Deployment](https://hexo.io/docs/deployment.html)
## fancybox

本篇博客开头的图片是这样实现的，只需要在你的文章*.md文件的头上添加photos项即可，然后一行行添加你要展示的照片:
```
---
title: hexo搭建个人博客
date: 2016-05-16 11:21:13
tags: hexo
categories: hexo
photos:
- http://bruce.u.qiniudn.com/2013/11/27/reading/photos-1.jpg
---
```

## 主题设置
本博客采用了iissnan的Next主题，他的博客有详细的安装教程，这里贴下链接[next](http://theme-next.iissnan.com/)，其中对next主题的设置讲的很详细了，我在这就不多讲了。

参考：[Jekyll迁移到Hexo搭建个人博客](http://www.ezlippi.com//blog/2016/02/jekyll-to-hexo.html)

><font color= Darkorange>如若觉得本文尚可，欢迎转载交流,转载请在正文明显处注明原文地址，谢谢！</font>
