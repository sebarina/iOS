# git常用命令解析

## 一、 git下载代码

> git clone <git address> -b <branch name> location

-b <branch name> : optional, 如果指定了，则下载分之代码，如果没有指定，则下载master分支代码；

location： optional， 如果指定了，则将远程代码下载到location目录下（location目录不一定要存在），如果没有指定，则会新建一个git名称的文件夹存放远程代码

## 二、 git分支操作

### 查看远程分支

> git branch --remote

### 查看本地分支

> git branch

### 查看本地和远程分支

> git branch -a

### 创建分支

> git branch <branch name> 

**注：** 此时，创建的是本地的分支

### 将本地分支推送到远程分支

> git push origin <branch name>

### 切换分支

> git checkout <branch name>

如果此时本地有修改没有提交，则该本地修改也会被相应的代入这个分支

### 删除本地分支

> git branch -d <branch name>

**注：** 删除本地分支,首先切换到别的分支,然后才能删除某个分支

### 删除远程分支

> git push origin --delete <branch name>

### 分支合并

合并本地分支：如本地develop分支合并release-1.0分支的代码

> - 切换到dev分支： git checkout develop
> - merge 本地release-1.0的代码： git merge release-1.0
> - merge完成后，将修改推送到远程： git push origin develop

## 三、 git tag操作

### 查看tag

> git tag

git tag -l 'v0.1.*'    # 搜索符合模式的标签

### 新建tag

创建轻量标签

> git tag <tag name>

创建带注解的标签

> git tag -a v1.01 -m "Relase version 1.01"

注解：git tag 是打标签的命令，-a 是添加标签，其后要跟新标签号，-m 及后面的字符串是对该标签的注释。

### 将本地tag push到远程

> git push origin <tag name>   #将指定标签提交到git服务器
> git push origin –tags    #将本地所有标签一次性提交到git服务器

### 删除本地tag

> git tag -d <tag name>

### 删除远程tag

> git push origin :refs/tags/v1.01

**注解：** 冒号前为空表示删除远程仓库的tag。

### 切换到tag

> git checkout <tag name>

