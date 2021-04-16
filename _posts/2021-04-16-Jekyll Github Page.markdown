---
layout: post
title:  "Jekyll Github Page 部署"
date:   2021-04-16 13:07:06 +0800
categories: jekyll github
---
记录一下 jekyll 搭建个人博客遇到的几个坑

1. 请参见 [Jekyll SEO Tag install][Jekyll_SEO_Tag_install] 安装 Jekyll SEO Tag，否则在github上部署的时候会提示seo标签无法识别

2. 调整当前时区，默认情况下Jekkly 仅显示比当前时间早的博客，调整时区可以通过在_config.yml中设置timezone: Asia/Shanghai，也可以通过在_config.yml中设置future: true 来去掉对于时间的限制






[Jekyll_SEO_Tag_install]: https://github.com/jekyll/jekyll-seo-tag/blob/master/docs/installation.md