### git常用操作和命令

1. 几个概念

   > Workspace：工作区---本地未add之前
   >
   > Index/stage：本地暂存区---add后
   >
   > repository：本地仓库---commit后
   >
   > remote：远程仓库---push

   ![img](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/bg2015120901.png)

2. 新建仓库

   ```shell
   # 初始化一个本地git仓库 repository
   $ git init [项目(仓库)名称]可选参数
   # 下载一个远程仓库到本地
   $ git clone 远程代码仓库url
   ```

3. 配置

   Git配置文件为<kbd>.gitconfig</kbd>，
   
4. 增加/删除文件

   ```shell
   # 增加指定文件到本地缓存区
   $ git add [file1] [file2]...
   # 增加指定目录到本地缓存区
   $ git add [dir]
   # 增加当前目录的所有文件到缓存区
   $ git add .
   # 添加每个变化的文件前要求确认
   $ git add -p
   ```

5. 提交文件到本地仓库

   ```shell
   # 提交暂存文件到本地仓库区
   $ git commit -m [message提交注释]
   # 提交指定文件
   $ git commit [file] [].. -m []
   # 提交工作区自上次commit之后的变化
   $ git commit -a
   # 提交时显示所有diff信息
   $ git commit -v
   ```

6. 远程同步

   ```shell
   # 下载远程仓库的所有变动
   $ git fetch [remote] #注意，这里的同步不会处理冲突，pull = fetch + merge
   # 显示所有远程分支
   $ git branch -v
   # 显示某个远程仓库信息
   $ git remote show [remote]
   # 增加一个新的远程仓库，并命名
   $ git remote add [remote] [url]
   # 拉取远程分支，并和本地合并
   $ git pull [remote] [branch] = fetch = merge
   # 上传本地指定分支到远程仓库
   $ git push [remote] [branch]
   ```

7. 分支

   ```shell
   # 列出所有本地分支
   $ git branch
   # 远程分支
   $ git branch -r
   # 所有本地和远程
   $ git branch -a
   # 新建一个分支，但不切换
   $ git branch [branch-name]
   # 新建分支并切换
   $ git branch -b [branch]
   # 新建一个分支，并和远程分支建立追踪关系
   $ git branch --track [branch] [remote-branch]
   # 切换到指定分支，并更新工作区
   $ git checkout [branch-namae]
   # 切换到上一个分支
   $ git branch -
   # 建立追踪关系，在现有分支和指定的远程分支之间
   $ git branch --set-upstream [branch] [remote-branch]
   # 合并指定分支到当前分支
   $ git merger [branch]
   # 选择一个commit 合并进当前分支
   $ git cherry-pick [commit]
   # 删除分支
   $ git branch -d [branch-name]
   # 删除远程分支
   $ git push origin --delete [branch-name]
   $ git branch -dr [remote/branch]
   ```

8. 撤销

   ```shell
   # 恢复暂存区的指定文件到工作区
   $ git checkout [file]
   # 恢复某个commit的指定文件到暂存区和工作区
   $ git checkout [commit] [file]
   # 恢复暂存区的所有文件到工作区
   $ git checkout .
   # 重置缓存区的指定文件，和上一次commit保持一致，但工作区不变
   $ git reset [file]
   # 重置缓存区到上一次commit，
   $ git reset --hard
   # 重置缓存区到指定commit
   $ git reset [commit]
   # 新建一个commit来覆盖指定commit
   $ git revert [commit]
   # 暂时将未提交的变化移除，稍后再加载
   $ git stash
   $ git stash pop
   ```

9. 查看信息

   ```shell
   # 显示有变化的文件
   $ git status
   # 显示当前分支的版本历史
   $ git log
   # 显示commit历史以及变化文件
   $ git log --stat
   ```

   