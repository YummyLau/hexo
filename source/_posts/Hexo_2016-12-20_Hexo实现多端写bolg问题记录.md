---
title: Hexo实现多端写blog问题记录
date: 2016-12-20 12:04:11
updated: 2017-07-22 14:22
comments: true
categories: Hexo
tags: [Hexo]
---
>因为不想总把自己的笔记本背到公司（渣本太重） =。=，于是不得不同步公司pc与渣本的hexo博客源文件。参考了网上各位大大的做法，折中选择了一种处理方法:
>**hexo生成的博客静态页托管于github静态页，hexo博客源文件另起一个github项目管理。**

因为hexo init之后默认是一个git项目，注意看.gitignore下的内容。
```
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
debug
```
上述文件主要是本地配置文件、内容缓存文件、调试日志文件，其中**public**文件夹中存放的是生成的静态页面，也不需要上传。**public**文件会被hexo d指令动态部署到远端github静态仓库中。
理解.gitignore及策略后，流程就变得很简单。

*  在自己的github上创建一个仓库，比如hexo。

```
远端仓库为：https://github.com/YummyLau/hexo.git(以我为例)
```

*  在本地bolg根目录文件执行命令

```
git init
git remote add origin https://github.com/YummyLau/hexo.git
```

*  由于我用到next主题，next主题为一个git仓库，如果直接把项目推到远端，则无法识别。故先删除next文件，再重新设置子项目

```
git rm -r themes/next
git add . 
git commit -m "首次提交资源，除next主题"
git push -u origin master 
```

*  自此，远端仓库可以看到已经有bolg资源，接下来添加next主题，先绑定子项目，然后更新子项目再提交

```
git remote add -f next git@github.com:iissnan/hexo-theme-next.git   
git subtree add --prefix=themes/next next master --squash
git fetch next master
git subtree pull --prefix=themes/next next master --squash
git subtree push --prefix=themes/next next master
```

*  ~~其他段的同步操作~~

```
git clone https://github.com/YummyLau/hexo.git
cd hexo 
npm install hexo
npm install 
npm install hexo-deployer-git
```

*  其他段的同步操作(更新于2017/07/22)

```
git clone https://github.com/YummyLau/hexo.git
cd hexo 
npm install hexo-cli -g
```

## 参考文章 
>[hexo主题同步](http://w4lle.github.io/2016/06/06/Hexo-themes/)  
>[hexo官网](https://hexo.io/)  
