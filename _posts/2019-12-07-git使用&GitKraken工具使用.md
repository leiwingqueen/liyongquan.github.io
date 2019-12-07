---
layout: post
title:  "git使用&GitKraken工具使用"
description: git使用&GitKraken工具使用
date:   2019-12-07 12:01:00 +000
categories: git
tags: git
---

### 命令行

- 基本命令

```shell
git clone rep_address       // 克隆一个远程库
git status                  //查看当前状态
git add file_name           //将某个文件添加到git
git add .                  // 将所有新文件添加到git
git commit -am 'message'   // 提交所有更改，带描述
git push                   //提交到远程库
git diff                     //查看diff
```

- 分支

```shell
git checkout branch_name      //checkout到某个分支
git branch                    //查看分支
git branch some               // 新建一个名为some的分支
git push [远程名] :[分支名]  //删除某一远程分支
```

-  提交到远程仓库 

```shell
//创建本地仓库，并push到远程仓库
git init        //将本地目录添加到仓库
git remote add [shortname] [url] 
git add .
git commit -am 'first commit'
git push --set-upstream [url]/[shortname] master       
```

-  提交者 

```shell
(PS: Md里'-'分点，下面的内容有断行就会分开，所以嵌入的代码块不能带空行，不然显示出错)
// 设置全局
git config --global user.name 'Author Name'    
git config --global user.email 'Author Email'
// 或者设置本地项目库配置
git config user.name 'Author Name'
git config user.email 'Author Email'   
```

-  cherry pick

```shell
git cherry-pick [commit-id]  //将某个commit，cherry pick到当前分支
```

-  合并 

```shell
git merge xxx              //将某个指定分支合并到当前分支
git merge [commit-id]  //将另一个分支某个commit-id为止的代码merge到当前分支
```

-  stash 

```shell
git stash                      //保存当前的工作进度。会分别对暂存区和工作区的状态进行保存
git stash save "message..."   //这条命令实际上是第一条 git stash 命令的完整版
git stash list                //显示进度列表。此命令显然暗示了git stash 可以多次保存工作进度，并用在恢复时候进行选择
git stash pop [--index] [<stash>]              //如果不使用任何参数，会恢复最新保存的工作进度，并将恢复的工作进度从存储的工作进度列表中清除。如果提供参数（来自 git stash list 显示的列表），则从该 <stash> 中恢复。恢复完毕也将从进度列表中删除 <stash>。选项--index 除了恢复工作区的文件外，还尝试恢复暂存区。
git stash apply [--index] [<stash>]          //除了不删除恢复的进度之外，其余和 git stash pop 命令一样
git stash clear                              //删除所有存储的进度
git stash show -p                      //查看最近stash的diff
git stash show -p stash@{1}     //查看某个stash的diff
```

-  回滚 

```shell
git reset --hard [commit-id]  //回滚到某个commit ， 可选 --soft, --mixed
```

-  标记 

```shell
git update-index --assume-unchanged [filename]     //假定某个文件没有发生变化，但是切换分支，pull代码会更新index
git update-index --no-assume-unchange [filename]   //取消
git update-index --skip-worktree  [filename]       //将某个文件从git检测中忽略
git update-index --no-skip-worktree [filename]     //取消
```

-  统计 

```shell
//统计仓库里每个人的提交行数
git log --format='%aN' | sort -u | while read name; do echo -en "$name\t"; git log --author="$name" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }' -; done
```

- ssh

见  [ssh-keys_8](https://gitee.com/oschina/git-osc/wikis/帮助#ssh-keys_8) 

### 参考文献

[Git实用操作和GitKraken工具使用](https://blog.csdn.net/xiaopihai86/article/details/80894862)

[Gitkraken使用教程](https://blog.csdn.net/niuchenliang524/article/details/81355699)