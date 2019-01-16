---
title: Github分支备份Hexo博客源文件
date: 2017-11-01 12:22:52
draft: true
---
### 需求
将hexo部署在GitHub Pages上，但仅同步编译好的静态文件，如果想要在多个平台写博客，hexo的源码就需要同步到云端，方便拉取部署。
最简单的方法就是利用博客的 repo 分支（ master 分支的必须用来存放你博客网站文件）托管 Hexo 源文件和配置达到备份的目的。

### 将博客目录源文件push到repo分支上
```shell
# 在博客目录下
git init
git add .
git commit -m "commit source files"
# 将hexo源文件映射到远程repo上
git remote add origin https://github.com/your-name/your-name.github.io.git
# 新建分支（hexo静态文件必须使用master）
git branch blogSource
# 切换分支
git checkout blogSource
# 拉取远程代码，将源文件push到分支
git pull origin master
git push -u origin blogSource
```
这里有个小坑，源文件中themes下主题如果是从原作者那里git下来的话，这里是无法提交的，需要使用fork + subtree的方法同步主题，具体可以看
http://w4lle.com/2016/06/06/Hexo-themes/index.html

我嫌麻烦，直接把git下来的next目录下文件复制出来，在themes里面新建文件夹，在博客目录重复进行以上初始化操作即可(感觉可以直接把next下的.git/目录删掉)

### 更新博客源文件
```
git add .
git commit -m "modify blog"
git push --set-upstream origin blogSource  //配置push,方便以后直接git push推送
git push #不用上一个步骤的话可直接使用git push -u origin blogSource
```
### 在其他机器上使用repo分支上的博客源文件
```
git clone xxxx.xxx(你的github page repo 地址)
git checkout origin/blogSource
npm install -g hexo-cli 
npm install 
npm install hexo-deployer-git --save
```