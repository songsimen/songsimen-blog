---
title: typora-vue-theme Theme introduction
date: 2015-09-07 09:25:00
author: Qi Zhao
img: /source/images/xxx.jpg
top: false
cover: false
coverImg: /images/1.jpg
password: 8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
toc: false
mathjax: false
summary: This is the content of your custom post summary. If there is a value for this attribute, the post card summary will display the text, otherwise the program will automatically intercept part of the post content as a summary.
categories: Markdown
tags:
  - Typora
  - Markdown
---



细的前端选项
Front-matter选项中的所有内容都不是必需的。不过我还是建议在值至少灌装title和date。

选项	默认	描述
标题	Markdown的文件标题	帖子标题，强烈建议填写此选项
日期	创建文件的日期和时间	发布时间，强烈建议填写此选项，最好确保它是全局唯一的
作者	author 在根 _config.yml	发表作者
IMG	一个值 featureImages	发布要素图像，例如： http://xxx.com/xxx.jpg
最佳	true	推荐帖子（帖子是否有顶部），如果是top值true，则推荐为主页帖子。
覆盖	false	该v1.0.2添加的版本指示后是否需要添加到网页转盘盖。
coverImg	空值	新版本v1.0.2表示帖子需要在主页的封面上显示图像路径。如果不是，则默认使用帖子的默认图像。
密码	空值	帖子读了密码。如果要为文章设置阅读验证密码，可以设置password必须加密的值，SHA256以防止其他人看到它。前提是该verifyPassword选项在主题中激活config.yml
TOC	true	无论TOC是否打开，您都可以关闭文章的TOC功能。前提是该toc选项在主题中激活config.yml
mathjax	false	是否启用数学公式支持，是否启动本文mathjax，并且需要在主题_config.yml文件中打开它。
摘要	空值	发布摘要，自定义发布摘要内容，如果属性有值，则明信片摘要会显示文字，否则程序会自动截取部分文章作为摘要
类别	空值	文章分类，该主题的分类代表宏观上较大的分类，一个分类只推荐一篇文章。
标签	空值	贴标签，帖子可以有多个标签
注意：

如果没有写img属性，帖子的特色piature将采取余数，并选择主题的特色图片让所有帖子的图片都有自己的特点。
值date应尽量保证每一篇文章都是独一无二的，因为Gitalk和Gitment认识id本主题中的价值被唯一地标识date。
如果要设置读取文章验证密码的功能，则不仅应在Front-matter中使用SHA256加密设置密码值，还应激活主题中的配置_config.yml。
以下是帖子的例子Front-matter。