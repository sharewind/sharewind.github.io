---
layout: post
title: "git 常用命令"
date: 2012-12-06 18:34
comments: true
categories: git DevTools
---

生成SSH公钥

	cd ~/.ssh
	ssh-keygen -t rsa -C "your_email@youremail.com"
	cat ~/.ssh/id_rsa.pub  显示公钥

配置全局用户名

	git config --global user.name "username"
	git config --global user.email "xxxx@email.com" 
	git config -l  查看所有配置参数

克隆仓库
	
	git clone git://github.com/jquery/jquery.git

项目初始化

	mkdir project_dir 创建项目目录
	git init  初始化项目
	touch README 生成readme 的空文件
	> README 【或】生成readme 的空文件

	git add -A  添加当前文件夹所有文件	
	git commit -m '[comment]'    进行提交
	git commit -a  提交所有修改的文件（不包括新建的文件）
	git commit -am '[comment]'  全部提交
	git status  查看git状态

建立git仓库
	
	mkdir project.git
	cd project.git
	git --bare init 

删除文件	

	git rm [file_name] 移除文件（包括路径）
	git rm -r -f [dir_name]  遍历移除文件夹的所有内容
	

工作目录中添加一个叫".gitignore"的文件，用来忽略某些文件。

日志

	git log 查看git日志
	git log -p 查看详细日志
	git log --pretty=oneline 查看日志显示在一行


远程操作

	git remote -v 查看远程仓库
	git remote add origin git@github.com:xxxx.git 添加远程仓库
	git remote rm origin 移动远程仓库
	git remote set-url --push origin [new_url] 修改远程仓库
	git remote show origin 显示远程库origin的资源 
	git push origin master
	git pull origin master

分支操作

	git branch 查看本地所有分支
	git branch -a 查看所有分支[包括远程]
	git branch -r 查看远程所有分支 
	
	git branch feature1 master 从主分支master创建分支feature1
	git branch -m old_branch_name new_branch_new 分支改名

	git checkout [branch] 切换到分支
	git checkout -b [branch] 创建并切换到分支
	git checkout -t origin/dev 切换到远程分支

	删除分支
	git branch -d [branch_name] 删除一个分支
	git branch -D [branch_name] 强制删除未合并的分支
	git push origin :branch_remote_name 删除远程分支
	git branch -r -d branch_remote_name 删除远程分支

	分支合并(将dev分支合并到master)	
	git checkout master
	git pull origin master
	git merge dev  执行合并
	git push origin master	

标签
	
	git tag 列出所有标签
	git tag [tag_name] 打标签
	git tag [tag_name] -a "comment" 打标签并加注释
	git tag -d [tag_name] 删除本地的标签

	git pull origin --tags 获取远程的所有标签
	git push origin --tags 推送本地的标签到远程
	git push origin [tag_name] 将标签推送到远程
	git push origin :refs/tags/tag_name 删除远程标签

比较
	git diff 查看尚末缓存的更新
	git diff [commit_version] [commit_version] 显示两次提交的区别
	git diff master dev 显示两个分支的差异
	git diff master 显示工作目录与master分支的差异
	git diff --cached 显示当前索引与上次提交的差异	

版本管理 
	
	git reset --hard HEAD 撤消一个合并
	git revert [commit_version] 还原某个版本的修改
	git reset [commit_version] 将工作目录还原到某个版本
	git checkout [branch_name] [commit_version] 签出某个提交版本的分支
	git reabse 重新定义分支的版本状态

![git rebase](/pics/git_rebase.jpg)

子模块
	
	git submodule add  url path  添加子模块


gitignore

顶层工作目录中添加一个叫".gitignore"的文件，用来忽略某些文件。

例子：
<pre>
# 以'#' 开始的行，被视为注释.
# 忽略掉所有文件名是 foo.txt 的文件.
foo.txt
# 忽略所有生成的 html 文件,
*.html
# foo.html是手工维护的，所以例外.
!foo.html
# 忽略所有.o 和 .a文件.
*.[oa]
</pre>

参考资料


- generating-ssh-keys [https://help.github.com/articles/generating-ssh-keys](https://help.github.com/articles/generating-ssh-keys)

- Git 命令参数及用法详解
[http://hi.baidu.com/sunboy_2050/item/81651e19339c8f6b3e87cecc](http://hi.baidu.com/sunboy_2050/item/81651e19339c8f6b3e87cecc)

