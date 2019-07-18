---
title: hexo迁移hugo并配置ci自动部署github pages
categories: ["生命不息"]
tags: ["hugo","gitlab","ci","hexo"]
date: 2019-07-18 14:14:00
---



最近一直收到github pages的通知，由hexo生成的静态站存在安全风险，要求升级，升级hexo会升级一大堆依赖node_modules，对于node小白来说一直没有进行升级。

*npm install可以说很形象了*
![](https://raw.githubusercontent.com/yongman/i/img/picgo/output.gif)

还是决定转hugo了，对比hexo来说真是太省事了，没有依赖，一个bin文件就能全部搞定，并且静态资源生成的速度和hexo不在一个量级。

hugo下用了xianmin的[jene主题](https://github.com/xianmin/hugo-theme-jane)，一如既往的简洁清爽，专注阅读。

对于托管在github pages的静态网站，当突然想写几句话的时候，要从上一次的源文件环境或者是同步过的环境中继续，否则就导致文章丢失，所以用hexo和hugo写文章的体验最差的就是环境的同步。

为了不用再担心环境的同步和备份，直接采用gitlab的持续集成。在nas上跑的gitlab环境，docker跑一个gitlab-runner，executor采用shell方式，在gitlab上配置完成后在源repo中增加.gitlab-ci.yml文件。

```
stages:
  - build
  - deploy

job 1:
  stage: build
  script:
    - hugo version

job 2:
  stage: deploy
  script:
    - hugo
    - echo -e "Deploying updates to GitHub..."
    - pwd
    - cd public
    - git init
    - git config core.sshCommand 'ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no' 
    - git add -A
    - git config user.name "name"
    - git config user.email "name@gmail.com"
    - git commit -m "rebuilding site `TZ="Asia/Shanghai" date +"%Y-%m-%d %H:%M:%S"`"
    - git remote add origin git@github.com:your-repo.git
    - git push --force -u origin master
    - cd ..
```
在执行之前，需要将该docker中对应用户的ssh rsa.pub添加到github的ssh中。

随时随地在gitlab的WebIDE中新建markdown，保存后会自动更新到github pages，终于不再受终端限制了。