![](http://www.ruanyifeng.com/blogimg/asset/2015/bg2015120901.png)


专用名词

- Workspace：工作区
- Index / Stage：暂存区
- Repository：仓库区（或本地仓库）
- Remote：远程仓库


# 常见命令 #



# 命令清单 #

## 新建代码库 ##

- 新建一个git代码库

	`git init`

- 新建一个目录,并初始化为代码库

	`git init [project-name]`

- 克隆一个项目
	
	`git clone [url]`

## 配置 ##

Git 的配置文件为.gitconfig

- 当前的git配置

	`git config --list`

- 设置提交代码时的用户信息

	`git config [--global] user.name "[name]"`
	
	`git config [--global] user.email "[email address]"`

## 增/删 文件 ##
### 增 ###

- 添加指定文件到暂存区
	
	`git add [file1] [file2]`
- 添加指定目录
	
	`git add [dir]`
- 添加当前目录所有文件 		
	
	`git add .`
###  删 ###

- 删除工作区文件,并将这次删除放入暂存区

	`git rm [file1] [file2]`
- 停止追踪指定文件，但该文件会保留在工作区
	
	`git rm --cached [file]`
- 改名文件，并且将这个改名放入暂存区

	`git mv [file-original] [file-renamed]`

## 代码提交 ##
- 提交暂存区到仓库区
	
	`git commit -m [message]`
- 提交暂存区的指定文件到仓库区
	
	`git commit [file1] [file2] [file3] ... -m [message]`
- 提交工作区自上次commit之后的变化，直接到仓库区
	
	`git commit -a`
- 提交时显示所有diff信息

	`git commit -v`
-  使用一次新的commit，替代上一次提交(如果代码没有任何新变化，则用来改写上一次commit的提交信息)

	`git commit --amend -m [message]`
-  重做上一次commit，并包括指定文件的新变化

	`git commit --amend [file] [file2]`

## 分支管理 ##

- 本地所有分支

	`git branch`
- 远程所有分支
	
	`git branch -r`
- 本地&远程所有分支

	`git branch -a`
- 新建分支

	`git branch [branch-name]`
- 新建分支,并且切换到该分支

	`git checkout [branch-name]`
- 新建分支,并指定远程分支建立追踪关系

	`git branch --track [branch] [remote-branch]`
- 切换分支

	`git checkout [branch-name]`
- 切换到上个分支

	`git chekout -`
- 建立追踪关系

	`git branch --set-upstream [branch] [remote-branch]`
- 合并指定分支到当前分支

	`git merge [branch]`
- 删除分支

	`git branch -d [branch-name]`
- 删除远程分支

	`git branch -dr [branch-name]`
	
	`git push origin --delete [branch-name]`

## 标签 ##

- 列出所有tag

	`git tag`
- 新建一个tag在当前commit

	`git tag [tagName]`
- 新建一个tag在指定commit

	`git tag [tagName] [commit]`
- 删除本地tag

	`git tag -d [tagName]`
- 删除远程tag

	`git push origin :refs/tags/[tagName]`
- 查看tag信息

	`git show [tagName]`
- 提交指定tag

	`git push [remote] [tagName]`
- 提交所有tag

    `git push [remote] --tags`
- 新建一个分支,指向某个tag

	`git checkout -b [branch] [tag]`

## 查看信息 ##

- 显示变更文件

    `git status`
- 显示当前分支的版本信息

    `git log`
- 显示commit历史,以及每次commit发生变更的文件

	`git log --stat`
- 搜索提交历史,根据关键词

	`git log -S [keyword]`
- 显示暂存区和工作区的差异

	`git diff`
- 显示今天你写了多少行代码

	`git diff --shortstat"@{0 day ago}"`
- 显示当前分支的最近几次提交

	`git reflog`

## 远程同步 ##

- 下载远程仓库所有变动

	`git fetch [remote]`
- 显示所有远程仓库

	`git remote -v`
- 显示某个远程仓库信息

	`git remote show [remote]`
- 新增一个远程仓库

	`git remote add [shortname] [url]`
- 取回远程仓库编号,并且与本地分支合并

	`git pull [remote] [branch]`
- 上传本地分支到远程仓库

	`git push [remote] [branch]`
- 强行推送当前分支到远程仓库,即使有冲突

	`git push [remote] --force`
- 推送所有分支到远程仓库

	`git push [remote] --all`
## 撤销 ##
- 恢复暂存区的指定文件到工作区

	`git checkout [file]`
- 恢复暂存区的所有文件到工作区

	`git checkout .`

