---
title: gitbub问题合集
created: '2023-06-18T05:13:26.553Z'
modified: '2023-07-04T12:26:58.686Z'
---

# gitbub问题合集

- Github新建仓库默认分支由master变更为main
```shell
# 切换到main分支
git checkout main

# pull master最新数据
git pull origin master --allow-unrelated-histories

# main合并master分支数据
git merge master
git push origin main

# 删除旧的 master 分支
git push --delete origin master
```

- git status中文文件名显示乱码
```shell

git config --global core.quotepath false
```

- git submodule 无法下载子模块的代码
存在一种使用场景,一个项目下有些 submodule 项目不能让某些开发者访问。那么执行 git submodule update 会报错导致中断。

方法1：修改配置文件 `.gitmodules` 对应子模块 active=false 和 ignore=all

方法2： 可以单个模块init和update
```
git submodule init xxx
git submotule udpate xxx
```
