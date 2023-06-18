---
title: gitbub问题合集
created: '2023-06-18T05:13:26.553Z'
modified: '2023-06-18T06:47:50.308Z'
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
