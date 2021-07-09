---
title: Hexo+Next主题+Travis CI搭建过程
categories: 配置
tags:
  - Hexo
  - latex
mathjax: true
date: 2021/5/9
update: 2021/7/8
---

翻阅网络上诸多博客，很难找到一篇完整的hexo next主题支持latex渲染并配置travis CI自动化工具可用的文章，因此发布这篇博客记录下我的配置过程。主要采用Pandoc+MathJax渲染引擎对latex进行渲染。

另外，有许多博客已经讲了如何使用图床插入图片，然而关于`pandoc`markdown渲染器的图片标题插入会存在诸多大大小小的问题，这里记录以便查阅。

后续不间断对本篇更新我的配置过程。



## Latex渲染支持

### 安装Pandoc

Pandoc的安装配置过程见[官方网站](https://pandoc.org/installing.html)，在此不过多赘述。



### 更换markdown渲染引擎

将markdown渲染引擎更换为`hexo-renderer-pandoc`，命令如下

```bash
npm un hexo-renderer-marked
npm install hexo-renderer-pandoc
```

<!-- more -->

### 修改主题配置

修改next主题配置，next主题配置文件一般为`themes/next/_config.yml`文件，作如下修改方式：

```yml
# Math Formulas Render Support
math:
  # Default (true) will load mathjax / katex script on demand.
  # That is it only render those page which has `mathjax: true` in Front-matter.
  # If you set it to false, it will load mathjax / katex srcipt EVERY PAGE.

  per_page: true

  # hexo-renderer-pandoc (or hexo-renderer-kramed) required for full MathJax support.
  mathjax:
    enable: true # 默认这里为false，改为true即可
    # See: https://mhchem.github.io/MathJax-mhchem/
    mhchem: false

  # hexo-renderer-markdown-it-plus (or hexo-renderer-markdown-it with markdown-it-katex plugin) required for full Katex support.
  katex:
    enable: false
    # See: https://github.com/KaTeX/KaTeX/tree/master/contrib/copy-tex
    copy_tex: false
```

另外需要提供Mathjax的cdn地址，一般只需将注释行取消注释即可：

```yml
vendors:
  # Internal path prefix.
  _internal: lib

  # Internal version: 3.1.0
  # anime: //cdn.jsdelivr.net/npm/animejs@3.1.0/lib/anime.min.js
  anime:

  # Internal version: 5.13.0
  # fontawesome: //cdn.jsdelivr.net/npm/@fortawesome/fontawesome-free@5/css/all.min.css
  # fontawesome: //cdnjs.cloudflare.com/ajax/libs/font-awesome/5.13.0/css/all.min.css
  fontawesome:

  # MathJax
  # mathjax: //cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js
  mathjax: //cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js
```

此时可在本地测试latex公式，理论上可以正确显示：

```latex
$$
a^2 = b^2 + c^2 \label{1}
$$
```


$$
\begin{equation}
	a^2 = b^2 + c^2 \label{1}
\end{equation}
$$




```latex
$\ref{1}$是勾股定理
```

$\eqref{1}$是勾股定理



### 修改travis CI配置

`travis CI`是一个持续集成的工具，可将Hexo持续集成并部署到Github Pages，详细的配置过程见[这里](https://easyhexo.com/1-Hexo-install-and-config/1-5-continuous-integration.html)，默认已经配置好`travis CI`。

在博客目录下的`.travis.yml`文件中作如下修改，关键是在脚本中写好安装`pandoc`并更换渲染器的代码

```yml
language: node_js # 编译语言、环境

sudo: required # 需要管理员权限

dist: xenial # 指定 CI 系统版本为 Ubuntu16.04 LTS

node_js: stable #Node.js 版本

branches:
  only:
    - source # 只有 hexo 分支检出更改才触发 CI

before_install: 
  - export TZ='Asia/Shanghai' #配置时区为东八区 UTC+8
  - npm install hexo-cli # 安装 hexo
  - wget https://github.com/jgm/pandoc/releases/download/2.7/pandoc-2.7-1-amd64.deb # 下载pandoc
  - sudo dpkg -i ./pandoc-2.7-1-amd64.deb # 以sudo方式安装

#  - sudo apt-get install libpng16-dev # 安装 libpng16-dev CI 编译出现相关报错时请取消注释

install:
  - npm install # 安装依赖
  - npm uninstall hexo-renderer-marked --save # 同上，更换markdown渲染器
  - npm install hexo-renderer-pandoc --save

script: # 执行脚本，清除缓存，生成静态文件
  - hexo clean
  - hexo Generate


deploy:
  provider: pages
  skip_cleanup: true # 跳过清理
  local_dir: public # 需要推送到 GitHub 的静态文件目录 
  name: $GIT_NAME # 用户名变量
  email: $GIT_EMAIL # 用户邮箱变量
  github_token: $GITHUB_TOKEN # GitHub Token 变量
  keep-history: true # 保持推送记录，以增量提交的方式
  target-branch: post # 推送的目标分支 local_dir->>master 分支
  on:
    branch: source # 工作分支
```

其中`before_install`会在`install`阶段之前执行，更多的`travis_CI`用法详见[持续集成服务 Travis CI 教程](http://www.ruanyifeng.com/blog/2017/12/travis_ci_tutorial.html)。

之后按照`travis_CI`提供的方式提交博客，测试成功（顺便测试我的图床）。



![latex测试](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/image-20210618143216597.png)



## 图片标题支持

矛盾点在于我们利用`pandoc`作为hexo的markdown渲染器后，`pandoc`会自动渲染如下格式中的图片标题：

```markdown
![标题](/imagesource)
```

而标题会自动左对齐且样式与正文无疑，十分丑陋，首先需要配置`pandoc`使得其对图片标题不进行渲染，从而便于采用`next`主题的`fancybox`进行配置，在hexo配置文件`_config.yml`添加如下内容：

```yml
pandoc:
  extensions:
    - '-implicit_figures'
```

接着需要启用`next`主题中对此的配置，在`next`主题中的`config.yml`找到`fancybox`并添加如下配置即可：

```yml
fancybox:
  enable: true
  caption: true
```

对于标题样式的修改可在`next\source\css\_common\components\post\post.styl`中找到`.image-caption`修改，例如我自己将`margin-top`稍微调大了些，更加美观，测试结果如下图所示：

![测试图片](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/wallhaven-392p2d.jpg)



